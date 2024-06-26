<template>
	<!-- 参数已element-plus图片预览组件相似 -->
	<MediaViewer
		v-if="true"
		:url-list="realSrcList"
		:videoType="videoType"
		:imgType="imgType"
		@close="closeViewer"
	/>
	<!-- <div @click="openViewer" :style="`width:${realWidth};height:${realHeight};`" class="viewer">
		<el-image v-if="isImage" class="viewer-img" fit="cover" :src="realSrc" />
		<div v-if="isVideo" style="z-index: 0; position: relative">
			<video
				:src="realSrc"
				:width="width"
				style="z-index: -1; position: relative"
				:height="height"
				controls
				disablePictureInPicture="true"
				controlslist="nodownload noremoteplayback"
			></video>
		</div>
	</div> -->
</template>
<script setup lang="ts">
import { isExternal } from "../utils/validate";
import { ref, computed } from "vue";
import MediaViewer from "./mediaViewer.vue";
const props = defineProps({
	src: {
		type: String,
		default: ""
	},
	width: {
		type: [Number, String],
		default: ""
	},
	height: {
		type: [Number, String],
		default: ""
	},
	// 视频格式
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
const isShow = ref(false);
// 默认显示第一张处理
const realSrc = computed(() => {
	if (!props.src) {
		return;
	}
	let real_src = props.src.split(",")[0];
	if (isExternal(real_src)) {
		return real_src;
	}
	return import.meta.env.VITE_APP_BASE_API + real_src;
});
// 判断第一张是否为图片
const isImage = computed(() => {
	if (!realSrc.value) {
		return;
	}
	const name = realSrc.value.split(".").slice(-1)[0].toLocaleLowerCase();
	const isImg = props.imgType.find((itemVideo) => itemVideo == name);
	if (isImg) {
		return true;
	} else {
		return false;
	}
});
// 判断第一张是否为视频
const isVideo = computed(() => {
	if (!realSrc.value) {
		return;
	}
	const name = realSrc.value.split(".").slice(-1)[0].toLocaleLowerCase();
	const isVideo = props.videoType.find((itemVideo) => itemVideo == name);
	if (isVideo) {
		return true;
	} else {
		return false;
	}
});
const realSrcList = computed(() => {
	if (!props.src) {
		return;
	}
	let real_src_list = props.src.split(",");
	let srcList = [];
	real_src_list.forEach((item) => {
		if (isExternal(item)) {
			return srcList.push(item);
		}
		return srcList.push(import.meta.env.VITE_APP_BASE_API + item);
	});
	return srcList;
});
const realWidth = computed(() =>
	typeof props.width == "string" ? props.width : `${props.width}px`
);

const realHeight = computed(() =>
	typeof props.height == "string" ? props.height : `${props.height}px`
);
// 关闭预览
function closeViewer() {
	isShow.value = false;
}
//打开预览
function openViewer() {
	isShow.value = true;
}
</script>
  <style lang="scss" scoped>
.viewer {
	cursor: pointer;
	border-radius: 4px;
	overflow: hidden;

	.viewer-img {
		width: 100%;
		height: 100%;
		transition: all 0.3s;
	}

	.viewer-img:hover {
		transform: scale(1.2);
	}
}
</style>