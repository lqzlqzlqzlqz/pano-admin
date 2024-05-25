<template>
	<div class="pano-btns-container">
		<el-button @click="isAddMarker = !isAddMarker">添加箭头</el-button>
		<el-button>添加图形</el-button>
	</div>
	<div id="viewer"></div>
	<el-dialog
		:before-close="handleClose"
		v-model="isAddMarkerFormOpen"
		title="添加"
		width="500"
		center
	>
		<el-form ref="ruleFormRef" :model="addMarkerFormData" :rules="rules" label-width="auto">
			<el-form-item label="提示" prop="tooltip">
				<el-input v-model="addMarkerFormData.tooltip" />
			</el-form-item>
			<el-form-item v-if="altMarkerId" label="配图" prop="pop">
				<cl-upload v-model="addMarkerFormData.pop" multiple />
			</el-form-item>
			<el-form-item v-else label="链接" prop="to">
				<el-input v-model="addMarkerFormData.to" />
			</el-form-item>
			<el-form-item>
				<el-button type="primary" @click="handleAddMarkerFormSubmit(ruleFormRef)">
					创建
				</el-button>
				<el-button @click="handleAddMarkerFormClose">取消</el-button>
			</el-form-item>
		</el-form>
	</el-dialog>
</template>

<script lang="ts" setup>
import { ref, onMounted, onUnmounted, watch } from "vue";
import { Viewer } from "@photo-sphere-viewer/core";
import { MarkersPlugin } from "@photo-sphere-viewer/markers-plugin";
import { GalleryPlugin } from "@photo-sphere-viewer/gallery-plugin";
import { AutorotatePlugin } from "@photo-sphere-viewer/autorotate-plugin";
import "@photo-sphere-viewer/core/index.css";
import "@photo-sphere-viewer/markers-plugin/index.css";
import "@photo-sphere-viewer/gallery-plugin/index.css";
import arrow from "../../../panos/static/icon/arrow.gif";
import { useCool } from "/@/cool";
import { ElMessage } from "element-plus";

const { service, route } = useCool();

const props = defineProps({
	hasAnimate: {
		type: Boolean,
		default: true
	},
	imgList: Array,
	hasGallery: {
		type: Boolean,
		default: true
	}
});

const viewer = ref(null);
const panoramaUrl = ref("");
const markersPlugin = ref(null);
const autorotatePlugin = ref(null);
const galleryPlugin = ref(null);
const currentMarkers = ref([]);
const currIndex = ref(0);
const animatedValues = {
	pitch: { start: -Math.PI / 2, end: 0.2 },
	yaw: { start: Math.PI, end: 0 },
	zoom: { start: 0, end: 50 },
	fisheye: { start: 2, end: 0 }
};
const defaultUrl =
	"http://192.168.101.30:9000/dev/public/uploads/20240422/0c82b931d10e4e6c9b691b0347c4afa0_(2).JPG";
const panoInfo = ref<any>();
const panoId = ref("");

onMounted(async () => {
	try {
		const promise = [];
		if (route.query?.pano_id) {
			panoId.value = route.query.pano_id;
		}
		promise.push(
			service.panos.panos.info({ id: panoId.value }).then((res) => (panoInfo.value = res))
		);
		promise.push(
			service.markers.markers.list({ panoId: panoId.value }).then((res) => {
				// (currentMarkers.value = res)
				if (res.length) {
					currentMarkers.value = res.map((v) => ({
						id: v.id,
						[v.pt]: v[v.pt],
						html: v.pop
							? undefined
							: `<img src='${arrow}' style='width: 50px; height: 50px; transform: rotate(0deg);'/>`,
						anchor: "bottom center",
						size: { width: 50, height: 50 },
						tooltip: v.tooltip,
						navigate: v.navigate,
						mt: v.mt
					}));
				}
			})
		);
		await Promise.all(promise);
	} catch (e) {
		console.log(e);
	}
	panoramaUrl.value =
		panoInfo.value?.panoSrc.replace(
			"http://127.0.0.1:8001/",
			"http://192.168.101.30:9000/dev/"
		) ?? defaultUrl;
	initViewer();
	handleGalleryChange();
});

watch(
	() => props.imgList,
	(newVal, oldVal) => {
		if (galleryPlugin.value && props.hasGallery) {
			galleryPlugin.value.setItems(newVal);
		}
	}
);

function initViewer() {
	viewer.value = new Viewer({
		container: "viewer",
		panorama: panoramaUrl.value || defaultUrl,
		caption: panoInfo.value?.title ?? "",
		touchmoveTwoFingers: true,
		mousewheelCtrlKey: false,
		navbar: [
			"autorotate",
			"zoom",
			"markers",
			"move",
			"download",
			"gallery",
			{
				title: "Change points",
				content: "🔄",
				onClick: randomPoints
			},
			"caption",
			"fullscreen"
		],
		plugins: [
			[GalleryPlugin, { visibleOnLoad: true, hideOnClick: false }],
			[AutorotatePlugin, { autostartDelay: null, autorotateSpeed: "1rpm" }],
			[MarkersPlugin, { markers: [] }]
		]
	});
	markersPlugin.value = viewer.value.getPlugin(MarkersPlugin);
	autorotatePlugin.value = viewer.value.getPlugin(AutorotatePlugin);
	galleryPlugin.value = viewer.value.getPlugin(GalleryPlugin);
	markersPlugin.value.addEventListener("enter-marker", (e) => {
		if (e.marker?.config?.mt === "graph") {
			const config = e.marker.config;
			config.svgStyle = {
				fill: "rgba(255, 165, 0, 0.3)" // 半透明橙色填充
			};
			markersPlugin.value.updateMarker(config);
		}
	});

	markersPlugin.value.addEventListener("leave-marker", (e) => {
		if (e.marker?.config?.mt === "graph") {
			const config = e.marker.config;
			config.svgStyle = {
				fill: "rgba(0, 0, 0, 0)"
			};
			markersPlugin.value.updateMarker(config);
		}
	});
	viewer.value.addEventListener("ready", handleViewerReady);
	viewer.value.addEventListener("click", handleClick);
}

function handleViewerReady() {
	markersPlugin.value.setMarkers(currentMarkers.value);
	// showInitMarker();
}

function showInitMarker() {
	viewer.value
		.animate({
			yaw: "-27deg",
			pitch: "-6deg",
			speed: 100
		})
		.then(() => {
			markersPlugin.value.showMarkerTooltip("new-marker1");
			autorotatePlugin.value.start();
		});
}

function handleGalleryChange() {
	document.addEventListener("click", function (e) {
		const element = document.elementFromPoint(e.clientX, e.clientY);
		let flag = false;
		let _sindex = "";
		if (element && element.dataset && element.dataset.psvGalleryItem) {
			const id = element.dataset.psvGalleryItem;
			_sindex = props.imgList.findIndex((data) => data.id == id);
			flag = true;
		} else if (element && element.className == "psv-gallery-item-thumb") {
			const eleId = element.parentElement.parentElement.dataset.psvGalleryItem;
			_sindex = props.imgList.findIndex((data) => data.id == eleId);
			flag = true;
		}
		if (flag) {
			if (currIndex.value !== _sindex) {
				currIndex.value = _sindex;
				panoramaUrl.value = props.imgList[_sindex].panorama;
				handleViewerChange("gallery");
			}
		}
	});
}

function handleViewerChange(type: string) {
	clearMarker();
	viewer.value.setPanorama(panoramaUrl.value).then(() => {
		markersPlugin.value.setMarkers(props.imgList[currIndex.value].markers);
	});
	if (type !== "gallery") {
		handleGalleryActive();
	}
}

function clearMarker() {
	markersPlugin.value.clearMarkers();
}

function randomPoints() {
	markersPlugin.value.showAllTooltips();
}

function handleGalleryActive() {
	const galleryBox = document.getElementsByClassName("psv-gallery-container")[0];
	const galleryItemEle = document.getElementsByClassName("psv-gallery-item")[currIndex.value];
	Array.from(galleryBox.childNodes).forEach((item: Element) => {
		item.className = "psv-gallery-item";
	});
	galleryItemEle.classList.add("psv-gallery-item--active");
}

// 状态
const isAddMarker = ref(false); // 是否可添加marker
const isAddMarkerFormOpen = ref(false); // 单个标记添加dialog
const addPosition = ref({ yaw: 0, pitch: 0 }); // 单标位置

// 添加标记表单
const ruleFormRef = ref();
const rules = ref({
	pop: [
		{ required: true, message: "请上传配图", trigger: "change" },
		{ type: "array", message: "配图必须是一个数组", trigger: "change" }
	]
});
const addMarkerFormData = ref({
	tooltip: "",
	to: "",
	pop: []
});

const isAltPressed = ref(false); // 是否按下alt
const altMarkerId = ref(""); // 当前正在添加的多边形标记的ID
const isAddMarkerGraphFormOpen = ref(false); // 图形dialog

// 当alt建按下时
onMounted(() => {
	const handleKeyDown = (event) => {
		if (event.altKey) {
			isAltPressed.value = true;
		}
		console.log(isAltPressed.value);
	};
	const handleKeyUp = (event) => {
		if (!event.altKey) {
			isAltPressed.value = false;
			if (altMarkerId.value === "") return;
			isAddMarkerFormOpen.value = true; // Open the form to add a graphical marker
			console.log(markersPlugin.value.getMarker(altMarkerId.value));
		}
		console.log(isAltPressed.value);
	};

	window.addEventListener("keydown", handleKeyDown);
	window.addEventListener("keyup", handleKeyUp);
	onUnmounted(() => {
		window.removeEventListener("keydown", handleKeyDown);
		window.removeEventListener("keyup", handleKeyUp);
	});
});

// 处理点击事件以添加标记
const handleClick = (event) => {
	if (!event) return;
	if (!isAddMarker.value) return;
	const yaw = event.data.yaw;
	const pitch = event.data.pitch;
	console.log(yaw, pitch);

	if (isAltPressed.value) {
		if (!markersPlugin.value) return;
		if (altMarkerId.value === "") {
			// 创建新标记
			const newMarkerId = `marker_${Date.now()}`;
			altMarkerId.value = newMarkerId;
			const newMarker = {
				id: newMarkerId,
				polygon: [
					[yaw, pitch] // 第一个点的纬度、经度
				],
				svgStyle: {
					fill: "rgba(0, 0, 0, 0)"
				},
				tooltip: {
					content: "",
					position: "top center"
				},
				mt: "graph"
			};
			markersPlugin.value.addMarker(newMarker);
		} else {
			const markerConfig = markersPlugin.value.getMarker(altMarkerId.value);
			markerConfig.config.polygon.push([yaw, pitch]);
			markersPlugin.value.updateMarker(markerConfig.config);
		}
		return;
	}
	addPosition.value = { yaw, pitch };
	isAddMarkerFormOpen.value = true;
};
const handleAddMarkerFormClose = () => {
	isAddMarkerFormOpen.value = false;
};

const handleClose = () => {
	addMarkerFormData.value = {
		tooltip: "",
		to: "",
		pop: []
	};
	if (altMarkerId.value) markersPlugin.value.removeMarker(altMarkerId.value);
	altMarkerId.value = "";
};

const handleAddMarkerGraphFormClose = () => {
	if (!markersPlugin.value) return;
	markersPlugin.value.removeMarker(altMarkerId.value);
	altMarkerId.value = "";
	isAddMarkerGraphFormOpen.value = false;
};

const handleAddMarkerFormSubmit = async (formEl: FormInstance | undefined) => {
	if (!formEl) return;
	const isValid = await formEl.validate((valid) => {
		return valid;
	});
	if (!isValid) return;

	let newMarker: any = {};
	let addMarker: any = {};
	if (altMarkerId.value) {
		const markerConfig = markersPlugin.value.getMarker(altMarkerId.value);
		if (addMarkerFormData.value.tooltip) {
			markerConfig.config.tooltip.content = addMarkerFormData.value.tooltip;
		} else {
			markerConfig.config.tooltip = undefined;
		}
		markerConfig.config.pop = addMarkerFormData.value.pop;
		newMarker = markerConfig.config;
		addMarker = {
			panoId: panoId.value,
			polygon: newMarker.polygon,
			svgStyle: {
				fill: "rgba(0, 0, 0, 0)"
			},
			tooltip: {
				content: addMarkerFormData.value.tooltip,
				position: "top center",
				trigger: "hover"
			},
			pt: "polygon",
			mt: "graph",
			visible: true,
			zIndex: 1,
			opacity: 1,
			rotation: 0,
			pop: addMarkerFormData.value.pop
		};
	} else {
		newMarker = {
			id: `marker_${Date.now()}`,
			position: addPosition.value,
			html: `<img src='${arrow}' style='width: 50px; height: 50px; transform: rotate(0deg);'/>`,
			anchor: "bottom center",
			size: { width: 50, height: 50 },
			tooltip: addMarkerFormData.value.tooltip,
			navigate: addMarkerFormData.value.to
		};
		addMarker = {
			panoId: panoId.value,
			pt: "position",
			mt: "arrow",
			tooltip: newMarker.tooltip,
			visible: true,
			navigate: newMarker.navigate,
			position: newMarker.position
		};
	}

	try {
		await service.markers.markers.add(addMarker);

		if (!markersPlugin.value) return;
		if (altMarkerId.value) {
			markersPlugin.value.updateMarker(newMarker);
		} else {
			markersPlugin.value.addMarker(newMarker);
		}
		handleAddMarkerFormClose();
		ElMessage.success("添加成功!");
		altMarkerId.value = "";
	} catch (e) {
		console.log(e);
		ElMessage.error("添加失败,请重试!");
	}
};
</script>

<style lang="scss" scoped>
#viewer {
	width: 100%;
	height: 100%;
}
.pano-btns-container {
	position: absolute;
	right: 0;
	top: 0;
	z-index: 100;
}
</style>
