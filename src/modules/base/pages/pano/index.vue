<template>
	<el-image
		id="preview_img_list"
		style="width: 100px; height: 100px; position: absolute; left: -10000px"
		:src="imageList[0]"
		:zoom-rate="1.2"
		:max-scale="7"
		:min-scale="0.2"
		:preview-src-list="imageList"
		:initial-index="4"
		fit="cover"
	/>
	<div id="viewer"></div>
	<el-dialog
		:before-close="handleClose"
		v-model="isAddMarkerFormOpen"
		:title="selectedMarker.id ? '修改' : '添加'"
		width="500"
		center
	>
		<el-form ref="ruleFormRef" :model="addMarkerFormData" :rules="rules" label-width="auto">
			<el-form-item label="提示" prop="tooltip">
				<el-input v-model="addMarkerFormData.tooltip" />
			</el-form-item>
			<el-form-item
				v-if="altMarkerId || selectedMarker.mt === 'graph'"
				label="配图"
				prop="pop"
			>
				<cl-upload v-model="addMarkerFormData.pop" multiple />
			</el-form-item>
			<el-form-item v-else label="跳转" prop="to">
				<el-select v-model="addMarkerFormData.to" placeholder="选择" style="width: 100%">
					<el-option
						v-for="item in panosNavigateOps"
						:key="item.value"
						:label="item.label"
						:value="item.value"
					/>
				</el-select>
			</el-form-item>
			<el-form-item>
				<el-button type="primary" @click="handleAddMarkerFormSubmit(ruleFormRef)">
					{{ selectedMarker.id ? "保存" : "创建" }}
				</el-button>
				<el-button @click="handleAddMarkerFormClose">取消</el-button>
				<el-button v-if="selectedMarker.id" type="danger" @click="handleRemoveMarker"
					>删除</el-button
				>
			</el-form-item>
		</el-form>
	</el-dialog>
</template>

<script lang="ts" setup>
import { ref, onMounted, onUnmounted, watch, nextTick } from "vue";
import { Viewer } from "@photo-sphere-viewer/core";
import { MarkersPlugin } from "@photo-sphere-viewer/markers-plugin";
import { GalleryPlugin } from "@photo-sphere-viewer/gallery-plugin";
import { AutorotatePlugin } from "@photo-sphere-viewer/autorotate-plugin";
import "@photo-sphere-viewer/core/index.css";
import "@photo-sphere-viewer/markers-plugin/index.css";
import "@photo-sphere-viewer/gallery-plugin/index.css";
import arrow from "../static/icon/arrow.gif";
import { useCool } from "/@/cool";
import { ElMessage, ElNotification } from "element-plus";

const { service, route, router } = useCool();

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
const defaultUrl = "";
const panoInfo = ref<any>();
const panoId = ref("");
const projectId = ref("");

const panosNavigateOps = ref([]);

const isImagePreviewOpen = ref(false);
const imageList = ref<string[]>([]);

// 初始 获取pano_id，panoInfo，
onMounted(async () => {
	if (route.query?.pano_id) {
		panoId.value = route.query.pano_id;
	}
	if (route.query?.project_id) {
		projectId.value = route.query.project_id;
	}
	await initPano();

	initViewer();
	handleGalleryChange();
});

watch(
	() => route.query,
	async () => {
		nextTick(async () => {
			await initPano();
			viewer.value.setPanorama(panoramaUrl.value).then((res) => {
				handleViewerReady();
			});
		});
	}
);

const initPano = async () => {
	try {
		const promise = [];
		currentMarkers.value = [];
		promise.push(
			// service.panos.panos.info({ id: panoId.value }).then((res) => (panoInfo.value = res))
			service.base.open
				.getPanos({
					panoId: panoId.value,
					projectId: projectId.value
				})
				.then((res) => {
					if (!res.panoDetail) router.push("/404");
					panoInfo.value = res.panoDetail;
					panosNavigateOps.value = res.panosList.map((v) => ({
						label: v.title,
						value: v.id + ""
					}));
					if (res.markersList.length) {
						currentMarkers.value = res.markersList.map((v) => ({
							id: v.id,
							[v.pt]: v[v.pt],
							html: v.html,
							anchor: "bottom center",
							svgStyle: v.svgStyle,
							size: { width: 50, height: 50 },
							tooltip: v.tooltip,
							navigate: v.navigate,
							mt: v.mt,
							pop: v.pop
						}));
					}
				})
				.catch((err) => router.push("/404"))
		);

		await Promise.all(promise);
		panoramaUrl.value = panoInfo.value?.panoSrc ?? defaultUrl;
		console.log(panoramaUrl.value, panoId.value);
	} catch (e) {
		console.log(e);
	}
};

watch(
	() => props.imgList,
	(newVal, oldVal) => {
		if (galleryPlugin.value && props.hasGallery) {
			galleryPlugin.value.setItems(newVal);
		}
	}
);

// 点击标记显示marker弹窗
const selectedMarker = ref({});

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
			// {
			// 	title: "Change points",
			// 	content: "🔄",
			// 	onClick: randomPoints
			// },
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
		if (e.marker?.config?.isAdding) return;
		if (e.marker?.config?.mt === "graph") {
			const config = e.marker.config;
			config.svgStyle = {
				fill: "rgba(255, 165, 0, 0.3)" // 半透明橙色填充
			};
			markersPlugin.value.updateMarker(config);
		}
	});

	markersPlugin.value.addEventListener("leave-marker", (e) => {
		if (e.marker?.config?.isAdding) return;
		if (e.marker?.config?.mt === "graph") {
			const config = e.marker.config;
			config.svgStyle = {
				fill: "rgba(0, 0, 0, 0)"
			};
			markersPlugin.value.updateMarker(config);
		}
	});
	markersPlugin.value.addEventListener("select-marker", async (e) => {
		if (e) {
			if (isAddMarker.value) {
				selectedMarker.value = e.marker.config;
				addMarkerFormData.value.tooltip = selectedMarker.value.tooltip.content;
				addMarkerFormData.value.to = selectedMarker.value.navigate;
				addMarkerFormData.value.pop = selectedMarker.value.pop;
				isAddMarkerFormOpen.value = true;
			} else {
				if (e.marker.config.mt === "arrow") {
					panoId.value = e.marker.config.navigate;
					clearMarker();
					router.push(
						`/pano?pano_id=${e.marker.config.navigate}&project_id=${projectId.value}`
					);
				} else if (e.marker.config.mt === "graph") {
					imageList.value = e.marker.config.pop;
					nextTick(() => {
						document.querySelector("#preview_img_list")?.click();
					});
				}
			}
		}
	});
	viewer.value.addEventListener("ready", handleViewerReady);
	viewer.value.addEventListener("click", handleClick);
}

const closeImagePreview = () => {
	isImagePreviewOpen.value = false;
};

function handleViewerReady() {
	console.log("READY");
	markersPlugin.value.setMarkers(currentMarkers.value);
	markersPlugin.value.showAllTooltips();
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
const changeIsAddMarker = () => {
	isAddMarker.value = !isAddMarker.value;
	if (isAddMarker.value) {
		ElNotification({
			title: "提示",
			message: "点击页面添加箭头，按住alt键连续点击添加图形",
			position: "bottom-right"
		});
	}
};
const isAddMarkerFormOpen = ref(false); // 单个标记添加dialog
const addPosition = ref({ yaw: 0, pitch: 0 }); // 单标位置

// 添加标记表单
const ruleFormRef = ref();
const rules = ref({
	tooltip: [{ required: true, message: "请输入提示", trigger: "change" }],
	to: [{ required: true, message: "请选择跳转位置", trigger: "change" }],
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
			console.log(1);
			const markerConfig = markersPlugin.value.getMarker(altMarkerId.value);
			console.log(markerConfig.config.polyline);
			if (markerConfig.config.polyline.length < 5) {
				ElMessage.error("请选择至少三个点");
				markersPlugin.value.removeMarker(altMarkerId.value);
				altMarkerId.value = "";
				return;
			}
			console.log(3);
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
				polyline: [
					[yaw - 0.001, pitch - 0.001],
					[yaw, pitch],
					[yaw + 0.001, pitch + 0.001] // 第一个点的纬度、经度
				],
				dot: [[yaw, pitch]],
				svgStyle: {
					fill: "rgba(255, 165, 0, 0.3)",
					stroke: "rgba(200, 0, 50, 0.8)",
					strokeWidth: "5px"
				},
				tooltip: {
					content: "",
					position: "top center"
				},
				mt: "graph",
				isAdding: true
			};
			markersPlugin.value.addMarker(newMarker);
		} else {
			const markerConfig = markersPlugin.value.getMarker(altMarkerId.value);
			markerConfig.config.polyline.push([yaw, pitch]);
			markerConfig.config.dot.push([yaw, pitch]);
			markersPlugin.value.updateMarker(markerConfig.config);
		}
		return;
	}
	addPosition.value = { yaw, pitch };
	isAddMarkerFormOpen.value = true;
};
const handleAddMarkerFormClose = () => {
	isAddMarkerFormOpen.value = false;
	addMarkerFormData.value = {
		tooltip: "",
		to: "",
		pop: []
	};
	if (altMarkerId.value) markersPlugin.value.removeMarker(altMarkerId.value);
	altMarkerId.value = "";
	selectedMarker.value = {};
};

const handleClose = (done) => {
	handleAddMarkerFormClose();
	done();
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

	if (selectedMarker.value.id) {
		console.log(selectedMarker.value);
		selectedMarker.value.navigate = addMarkerFormData.value.to;
		selectedMarker.value.pop = addMarkerFormData.value.pop;
		selectedMarker.value.tooltip.content = addMarkerFormData.value.tooltip;

		try {
			await service.markers.markers.update(selectedMarker.value);

			if (!markersPlugin.value) return;
			markersPlugin.value.updateMarker(selectedMarker.value);
			ElMessage.success("更新成功!");
			isAddMarkerFormOpen.value = false;
			addMarkerFormData.value = {
				tooltip: "",
				to: "",
				pop: []
			};
			altMarkerId.value = "";
			selectedMarker.value = {};
		} catch (e) {
			console.log(e);
			ElMessage.error("更新失败,请重试!");
		}
	} else {
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
			newMarker.pop = addMarkerFormData.value.pop;
			addMarker = {
				id: `marker_${Date.now()}`,
				panoId: panoId.value,
				polygon: newMarker.dot,
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
				pop: addMarkerFormData.value.pop,
				isAdding: false
			};
		} else {
			newMarker = {
				id: `marker_${Date.now()}`,
				position: addPosition.value,
				html: `<img src='http://qiniu-misc.hua10.com/1717154078197-9ff50574b17240db8de946ad02635fe6_arrow.gif' style='width: 50px; height: 50px; transform: rotate(0deg);'/>`,
				anchor: "bottom center",
				size: { width: 50, height: 50 },
				tooltip: {
					content: addMarkerFormData.value.tooltip,
					position: "top center",
					trigger: "hover"
				},
				navigate: addMarkerFormData.value.to,
				mt: "arrow"
			};
			addMarker = {
				panoId: panoId.value,
				pt: "position",
				mt: "arrow",
				tooltip: newMarker.tooltip,
				visible: true,
				navigate: newMarker.navigate,
				position: newMarker.position,
				html: newMarker.html
			};
		}

		try {
			await service.markers.markers.add(addMarker);

			if (!markersPlugin.value) return;
			console.log(altMarkerId.value, newMarker);
			if (altMarkerId.value) {
				markersPlugin.value.removeMarker(altMarkerId.value);
				markersPlugin.value.addMarker(addMarker);
			} else {
				markersPlugin.value.addMarker(newMarker);
			}
			ElMessage.success("添加成功!");
			isAddMarkerFormOpen.value = false;
			addMarkerFormData.value = {
				tooltip: "",
				to: "",
				pop: []
			};
			altMarkerId.value = "";
			selectedMarker.value = {};
		} catch (e) {
			console.log(e);
			ElMessage.error("添加失败,请重试!");
		}
	}
};

const handleRemoveMarker = async () => {
	try {
		await service.markers.markers.delete({ ids: [selectedMarker.value.id] });
		markersPlugin.value.removeMarker(selectedMarker.value.id);
		ElMessage.success("删除成功!");
		isAddMarkerFormOpen.value = false;
		addMarkerFormData.value = {
			tooltip: "",
			to: "",
			pop: []
		};
		altMarkerId.value = "";
		selectedMarker.value = {};
	} catch (e) {
		console.log(e);
		ElMessage.error("删除失败,请重试!");
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
	right: 10px;
	top: 10px;
	z-index: 100;
}

.image-preview-modal {
	position: fixed;
	top: 0;
	left: 0;
	width: 100%;
	height: 100%;
	background: rgba(0, 0, 0, 0.8);
	display: flex;
	justify-content: center;
	align-items: center;
	z-index: 1000;
}

.image-preview-content {
	position: relative;
	background: transparent;
	border-radius: 10px;
	padding: 20px;
	box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.image-preview-container {
	display: flex;
	flex-direction: column;
	align-items: center;
}

.image-preview-item {
	margin-bottom: 20px;
}

.preview-image {
	max-width: 80vw;
	max-height: 80vh;
	object-fit: contain;
	border-radius: 10px;
	box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
	transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.preview-image:hover {
	transform: scale(1.05);
	box-shadow: 0 8px 16px rgba(0, 0, 0, 0.2);
}

.no-image {
	display: flex;
	justify-content: center;
	align-items: center;
	height: 100%;
	color: #fff;
	font-size: 18px;
}

.close-button {
	position: absolute;
	top: 10px;
	right: 10px;
	background: transparent;
	border: none;
	color: #fff;
	font-size: 24px;
	cursor: pointer;
}
</style>
