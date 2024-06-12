<template>
	<transition name="fade">
		<div
			v-if="isSceneSelect"
			class="image-scroll-container"
			style="overflow-x: auto; white-space: nowrap"
		>
			<div class="image-container">
				<el-tooltip
					v-for="(v, index) in panoList"
					:key="index"
					:content="v.title"
					placement="top"
					effect="dark"
				>
					<div
						:style="{
							padding: '3px',
							margin: '3px 0',
							background: sceneId == v.id ? '#F6B64C' : 'white',
							cursor: 'pointer',
							position: 'relative'
						}"
						@click="goToScene(v.id)"
					>
						<img
							:src="v.thumb"
							:alt="v.title"
							style="width: 86px; height: auto; display: block"
						/>
						<div class="title">{{ v.title }}</div>
					</div>
				</el-tooltip>
			</div>
		</div>
	</transition>
</template>

<script lang="ts" setup>
import Vue from "vue";
import { useCool } from "/@/cool";
const props = defineProps<{
	isSceneSelect: boolean;
	projectId: any;
	sceneId: any;
	panoList: any;
	toPath: string;
}>();

const { router } = useCool();
const goToScene = (id: any) => {
	router.push(`${props.toPath}?pano_id=${id}&project_id=${props.projectId}`);
};
</script>
<style scoped>
.image-scroll-container {
	display: flex;
	justify-content: center;
	background-color: rgba(128, 128, 128, 0.5);
	position: absolute;
	bottom: 100px;
	width: 100%;
	z-index: 100;
	padding: 10px 0;
}

.image-container {
	gap: 10px;
	display: flex;
	justify-content: flex-start;
	align-items: flex-start;
	overflow-x: auto;
	overflow-y: hidden;
	-webkit-overflow-scrolling: touch;
	position: relative;
	overflow: hidden;
}

.container-fade {
}

.title {
	position: absolute;
	background-color: rgba(128, 128, 128, 0.5);
	bottom: 3px;
	font-size: 12px;
	color: white;
	width: calc(100% - 6px);
	text-align: center;
	left: 3px;
}

/* 定义进入和离开过渡的样式 */
/* .fade-enter-active,
.fade-leave-active {
	opacity: 1;
	transition: opacity 0.5s;
}
.fade-enter,
.fade-leave-to {
	opacity: 0;
} */
/* 定义进入和离开过渡的样式 */
.fade-enter-active,
.fade-leave-active {
	transition: opacity 0.5s;
}

.fade-enter, .fade-leave-to /* .fade-leave-active for <=2.1.8 */ {
	opacity: 0;
}

/* 添加进入开始的状态 */
.fade-enter-from {
	opacity: 0;
}

/* 添加进入结束的状态 */
.fade-enter-to {
	opacity: 1;
}
</style>