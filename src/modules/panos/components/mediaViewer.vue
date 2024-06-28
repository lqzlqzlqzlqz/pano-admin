<template>
	<teleport to="body">
		<transition name="viewer-fade">
			<div ref="wrapper" :tabindex="-1" class="el-image-viewer__wrapper" :style="{ zIndex }">
				<div class="el-image-viewer__mask" @click.self="hideOnClickModal && hide()"></div>
				<!-- CLOSE -->
				<span class="el-image-viewer__btn el-image-viewer__close btn-bgd" @click="hide">
					<el-icon color="black">
						<Close />
					</el-icon>
				</span>
				<!-- ARROW -->
				<template v-if="!isSingle">
					<span
						class="el-image-viewer__btn el-image-viewer__prev btn-bgd"
						:class="{ 'is-disabled': !infinite && isFirst }"
						@click="prev"
					>
						<el-icon color="black">
							<ArrowLeft />
						</el-icon>
					</span>
					<span
						class="el-image-viewer__btn el-image-viewer__next btn-bgd"
						:class="{ 'is-disabled': !infinite && isLast }"
						@click="next"
					>
						<el-icon color="black">
							<ArrowRight />
						</el-icon>
					</span>
				</template>
				<!-- ACTIONS -->
				<div v-if="isImage" class="el-image-viewer__btn el-image-viewer__actions btn-bgd">
					<div class="el-image-viewer__actions__inner">
						<el-icon @click="handleActions('zoomOut')" color="black">
							<ZoomOut />
						</el-icon>
						<el-icon @click="handleActions('zoomIn')" color="black">
							<ZoomIn />
						</el-icon>
						<i class="el-image-viewer__actions__divider"></i>

						<el-icon @click="toggleMode" color="black">
							<component :is="mode.icon"></component>
							<FullScreen />
						</el-icon>
						<i class="el-image-viewer__actions__divider"></i>

						<el-icon @click="handleActions('anticlocelise')" color="black">
							<RefreshLeft />
						</el-icon>

						<el-icon @click="handleActions('clocelise')" color="black">
							<RefreshRight />
						</el-icon>
					</div>
				</div>
				<!-- CANVAS -->
				<div class="el-image-viewer__canvas">
					<template v-for="(url, i) in urlList">
						<img
							v-if="i === index && isImage"
							ref="media"
							:key="url"
							:src="url"
							:style="mediaStyle"
							class="el-image-viewer__img"
							@load="handleMediaLoad"
							@error="handleMediaError"
							@mousedown="handleMouseDown"
						/>
						<video
							controls="controls"
							v-if="i === index && isVideo"
							ref="media"
							:key="url"
							:src="url"
							:style="mediaStyle"
							class="el-image-viewer__img"
							@load="handleMediaLoad"
							@error="handleMediaError"
							@mousedown="handleMouseDown"
						></video>
					</template>
				</div>
			</div>
		</transition>
	</teleport>
</template>
   
<script setup lang="ts">
import { computed, ref, onMounted, watch, nextTick } from "vue";
import {
	ArrowLeft,
	ArrowRight,
	Close,
	RefreshLeft,
	RefreshRight,
	ZoomOut,
	ZoomIn,
	FullScreen
} from "@element-plus/icons-vue";

const props = defineProps({
	urlList: {
		type: Array,
		default: () => []
	},
	zIndex: {
		type: Number,
		default: 9999
	},
	initialIndex: {
		type: Number,
		default: 0
	},
	infinite: {
		type: Boolean,
		default: true
	},
	hideOnClickModal: {
		type: Boolean,
		default: false
	}, // 视频格式
	videoType: {
		type: Array,
		default: [
			"avi",
			"wmv",
			"mpg",
			"mpeg",
			"mov",
			"rm",
			"ram",
			"swf",
			"flv",
			"mp4",
			"mp3",
			"wma",
			"avi",
			"rm",
			"rmvb",
			"flv",
			"mpg",
			"mkv"
		]
	},
	//图片格式
	imgType: {
		type: Array,
		default: [
			"svgz",
			"pjp",
			"png",
			"ico",
			"avif",
			"tiff",
			"tif",
			"jfif",
			"svg",
			"xbm",
			"pjpeg",
			"webp",
			"jpg",
			"jpeg",
			"bmp",
			"gif"
		]
	}
});
const emits = defineEmits(["close", "switch"]);
const EVENT_CODE = {
	tab: "Tab",
	enter: "Enter",
	space: "Space",
	left: "ArrowLeft", // 37
	up: "ArrowUp", // 38
	right: "ArrowRight", // 39
	down: "ArrowDown", // 40
	esc: "Escape",
	delete: "Delete",
	backspace: "Backspace"
};

const isFirefox = function () {
	return !!window.navigator.userAgent.match(/firefox/i);
};

const rafThrottle = function (fn) {
	let locked = false;
	return function (...args) {
		if (locked) return;
		locked = true;
		window.requestAnimationFrame(() => {
			fn.apply(this, args);
			locked = false;
		});
	};
};

const Mode = {
	CONTAIN: {
		name: "contain",
		icon: "FullScreen"
	},
	ORIGINAL: {
		name: "original",
		icon: "ScaleToOriginal"
	}
};

const mousewheelEventName = isFirefox() ? "DOMMouseScroll" : "mousewheel";

let _keyDownHandler = null;
let _mouseWheelHandler = null;
let _dragHandler = null;
const loading = ref(true);
const index = ref(props.initialIndex);
const wrapper = ref(null);
const media = ref(null);
const mode = ref(Mode.CONTAIN);
const transform = ref({
	scale: 1,
	deg: 0,
	offsetX: 0,
	offsetY: 0,
	enableTransition: false
});

const isSingle = computed(() => {
	const { urlList } = props;
	return urlList.length <= 1;
});

const isFirst = computed(() => {
	return index.value === 0;
});

const isLast = computed(() => {
	return index.value === props.urlList.length - 1;
});

const currentMedia = computed(() => {
	return props.urlList[index.value];
});

const isVideo = computed(() => {
	const currentUrl = props.urlList[index.value];
	const name = currentUrl.split(".").slice(-1)[0].toLocaleLowerCase();
	const isVideo = props.videoType.find((itemVideo) => itemVideo == name);
	if (isVideo) {
		return true;
	} else {
		return false;
	}
});

const isImage = computed(() => {
	const currentUrl = props.urlList[index.value];
	const name = currentUrl.split(".").slice(-1)[0].toLocaleLowerCase();
	const isImg = props.imgType.find((itemVideo) => itemVideo == name);
	if (isImg) {
		return true;
	} else {
		return false;
	}
});

const mediaStyle = computed(() => {
	const { scale, deg, offsetX, offsetY, enableTransition } = transform.value;
	const style = {
		transform: `scale(${scale}) rotate(${deg}deg)`,
		transition: enableTransition ? "transform .3s" : "",
		marginLeft: `${offsetX}px`,
		marginTop: `${offsetY}px`
	};
	if (mode.value.name === Mode.CONTAIN.name) {
		style.maxWidth = style.maxHeight = "100%";
	}
	return style;
});
function hide() {
	deviceSupportUninstall();
	emits("close");
}

function deviceSupportInstall() {
	_keyDownHandler = rafThrottle((e) => {
		switch (e.code) {
			// ESC
			case EVENT_CODE.esc:
				hide();
				break;
			// SPACE
			case EVENT_CODE.space:
				toggleMode();
				break;
			// LEFT_ARROW
			case EVENT_CODE.left:
				prev();
				break;
			// UP_ARROW
			case EVENT_CODE.up:
				handleActions("zoomIn");
				break;
			// RIGHT_ARROW
			case EVENT_CODE.right:
				next();
				break;
			// DOWN_ARROW
			case EVENT_CODE.down:
				handleActions("zoomOut");
				break;
		}
	});

	_mouseWheelHandler = rafThrottle((e) => {
		const delta = e.wheelDelta ? e.wheelDelta : -e.detail;
		if (delta > 0) {
			handleActions("zoomIn", {
				zoomRate: 0.015,
				enableTransition: false
			});
		} else {
			handleActions("zoomOut", {
				zoomRate: 0.015,
				enableTransition: false
			});
		}
	});

	document.addEventListener("keydown", _keyDownHandler, false);
	document.addEventListener(mousewheelEventName, _mouseWheelHandler, false);
}

function deviceSupportUninstall() {
	document.removeEventListener("keydown", _keyDownHandler, false);
	document.removeEventListener(mousewheelEventName, _mouseWheelHandler, false);
	_keyDownHandler = null;
	_mouseWheelHandler = null;
}

function handleMediaLoad() {
	loading.value = false;
}

function handleMediaError(e) {
	loading.value = false;
}

function handleMouseDown(e) {
	if (loading.value || e.button !== 0) return;

	const { offsetX, offsetY } = transform.value;
	const startX = e.pageX;
	const startY = e.pageY;

	const divLeft = wrapper.value.clientLeft;
	const divRight = wrapper.value.clientLeft + wrapper.value.clientWidth;
	const divTop = wrapper.value.clientTop;
	const divBottom = wrapper.value.clientTop + wrapper.value.clientHeight;

	_dragHandler = rafThrottle((ev) => {
		transform.value = {
			...transform.value,
			offsetX: offsetX + ev.pageX - startX,
			offsetY: offsetY + ev.pageY - startY
		};
	});
	document.addEventListener("mousemove", _dragHandler, false);
	document.addEventListener(
		"mouseup",
		(e) => {
			const mouseX = e.pageX;
			const mouseY = e.pageY;
			if (mouseX < divLeft || mouseX > divRight || mouseY < divTop || mouseY > divBottom) {
				reset();
			}
			document.removeEventListener("mousemove", _dragHandler, false);
		},
		false
	);

	e.preventDefault();
}

function reset() {
	transform.value = {
		scale: 1,
		deg: 0,
		offsetX: 0,
		offsetY: 0,
		enableTransition: false
	};
}

function toggleMode() {
	if (loading.value) return;

	const modeNames = Object.keys(Mode);
	const modeValues = Object.values(Mode);
	const currentMode = mode.value.name;
	const index = modeValues.findIndex((i) => i.name === currentMode);
	const nextIndex = (index + 1) % modeNames.length;
	mode.value = Mode[modeNames[nextIndex]];
	reset();
}

function prev() {
	if (isFirst.value && !props.infinite) return;
	const len = props.urlList.length;
	index.value = (index.value - 1 + len) % len;
}

function next() {
	if (isLast.value && !props.infinite) return;
	const len = props.urlList.length;
	index.value = (index.value + 1) % len;
}

function handleActions(action, options = {}) {
	if (loading.value) return;
	const { zoomRate, rotateDeg, enableTransition } = {
		zoomRate: 0.2,
		rotateDeg: 90,
		enableTransition: true,
		...options
	};
	switch (action) {
		case "zoomOut":
			if (transform.value.scale > 0.2) {
				transform.value.scale = parseFloat((transform.value.scale - zoomRate).toFixed(3));
			}
			break;
		case "zoomIn":
			transform.value.scale = parseFloat((transform.value.scale + zoomRate).toFixed(3));
			break;
		case "clocelise":
			transform.value.deg += rotateDeg;
			break;
		case "anticlocelise":
			transform.value.deg -= rotateDeg;
			break;
	}
	transform.value.enableTransition = enableTransition;
}

watch(currentMedia, () => {
	nextTick(() => {
		const $media = media.value;
		if (!$media.complete) {
			loading.value = true;
		}
	});
});

watch(index, (val) => {
	reset();
	emits("switch", val);
});

onMounted(() => {
	deviceSupportInstall();
	// add tabindex then wrapper can be focusable via Javascript
	// focus wrapper so arrow key can't cause inner scroll behavior underneath
	wrapper.value?.focus?.();
});
</script>

<style scoped lang="scss">
.btn-bgd {
	background: white;
}
</style>