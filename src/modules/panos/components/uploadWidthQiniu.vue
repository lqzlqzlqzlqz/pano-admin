<template>
	<div class="upload-info">
		<el-upload
			style="margin: 20px auto"
			:action="domain"
			:data="QiniuData"
			:on-remove="handleRemove"
			:on-error="uploadError"
			:on-change="onChange"
			:on-success="uploadSuccess"
			:before-upload="beforeAvatarUpload"
			:on-preview="handlePictureCardPreview"
			:file-list="fileList"
			list-type="picture-card"
			:limit="limit"
			ref="uploadRef"
			multiple
			accept="image/*,video/*"
		>
			<el-icon><Plus /></el-icon>
			<template #file="{ file }">
				<div v-if="isImage(file.url)">
					<img
						class="el-upload-list__item-thumbnail upload_img_fit"
						:src="file.url"
						alt=""
					/>
					<span class="el-upload-list__item-actions">
						<span
							class="el-upload-list__item-preview"
							@click="handlePictureCardPreview(file)"
						>
							<el-icon><zoom-in /></el-icon>
						</span>
						<!-- <span
							v-if="!disabled"
							class="el-upload-list__item-delete"
							@click="handleDownload(file)"
						>
							<el-icon><Download /></el-icon>
						</span> -->
						<span
							v-if="!disabled"
							class="el-upload-list__item-delete"
							@click="handleRemove(file.url)"
						>
							<el-icon><Delete /></el-icon>
						</span>
					</span>
				</div>
				<div v-else>
					<video width="100%" height="100%">
						<source :src="file.url" type="video/mp4" />
						Your browser does not support the video tag.
					</video>
					<span class="el-upload-list__item-actions">
						<span
							class="el-upload-list__item-preview"
							@click="handlePictureCardPreview(file)"
						>
							<el-icon><zoom-in /></el-icon>
						</span>
						<!-- <span
							v-if="!disabled"
							class="el-upload-list__item-delete"
							@click="handleDownload(file)"
						>
							<el-icon><Download /></el-icon>
						</span> -->
						<span
							v-if="!disabled"
							class="el-upload-list__item-delete"
							@click="handleRemove(file.url)"
						>
							<el-icon><Delete /></el-icon>
						</span>
					</span>
				</div>
			</template>
		</el-upload>
		<!-- <el-dialog v-model="dialogVisible" style="margin: 5vh auto">
			<img v-if="isImage(dialogImageUrl)" width="100%" :src="dialogImageUrl" alt="" />
			<video v-else width="100%" controls>
				<source :src="dialogImageUrl" type="video/mp4" />
				Your browser does not support the video tag.
			</video>
		</el-dialog> -->
	</div>

	<media-viewer
		v-if="dialogVisible"
		:url-list="fileEcho"
		:initialIndex="initialIndex"
		:infinite="true"
		:hideOnClickModal="true"
		@close="closeViewer"
	/>
</template>
  
  <script lang="ts" setup>
import { ElMessage, UploadInstance, UploadFile, UploadFiles } from "element-plus";
import { ref, reactive, watch, onMounted } from "vue";
import { useCool } from "/@/cool";
import { Plus, ZoomIn, Download, Delete } from "@element-plus/icons-vue";
import MediaViewer from "./mediaViewer.vue";
const { service } = useCool();

const uploadRef = ref<UploadInstance>();

const emit = defineEmits(["setImgList"]);

interface QiniuData {
	key: string;
	token: string;
	data: any;
}

const props = defineProps<{
	limit: number;
	fileEcho: any[];
	isPreview: boolean;
	disabled: boolean;
}>();

const fileList = ref<any[]>([]);
const dialogImageUrl = ref("");
const dialogVisible = ref(false);
const domain = ref(""); // 七牛云的上传地址（华东区）
const QiniuData = reactive<QiniuData>({
	key: "",
	token: "",
	data: {}
});
const publicDomain = ref("");

// watch(
// 	() => props.fileEcho,
// 	() => {
// 		initImgs();
// 	}
// );

onMounted(() => {
	getQiniuToken();
	initImgs();
});

function initImgs() {
	if (props.fileEcho) {
		try {
			fileList.value = props.fileEcho.map((v: string, index: number) => ({
				name: v,
				url: v,
				uid: v
			}));
		} catch (e) {
			console.log(e);
		}
	} else {
		fileList.value = [];
	}
}

const initialIndex = ref(0);
const handlePictureCardPreview = (file: any) => {
	dialogImageUrl.value = file.url;
	const findIndex = props.fileEcho.findIndex((v) => file.url === v);
	console.log(props.fileEcho, "-", findIndex, "-", file.url);
	if (findIndex !== -1) {
		initialIndex.value = findIndex;
		dialogVisible.value = true;
	}
};

const handleRemove = (fileUrl: string) => {
	fileList.value = fileList.value.filter((v) => v.url !== fileUrl);
	emit(
		"setImgList",
		fileList.value.map((v) => v.url)
	);
};

const beforeAvatarUpload = (file: any) => {
	QiniuData.data = file;
	QiniuData.key = `${file.name}`;
};

const uploadSuccess = (response: any, file: any, fl: any[]) => {
	const url = `${publicDomain.value}/${response.key}`;
	file.url = url;
	emit(
		"setImgList",
		fl.map((v) => v.url)
	);
};

const uploadError = (err: any, file: any, fl: any[]) => {
	ElMessage({
		message: "上传出错，请重试！",
		type: "error",
		center: true
	});
};

const getQiniuToken = async () => {
	const uploadConfig = await service.base.comm.qiniuUpload();
	QiniuData.token = uploadConfig.token;
	domain.value = uploadConfig.uploadUrl;
	publicDomain.value = uploadConfig.publicDomain;
};

const isImage = (url: string) => {
	if (url.startsWith("blob")) return true;
	return /\.(jpg|jpeg|png|gif)$/.test(url);
};

const onChange = (uploadFile: UploadFile, uploadFiles: UploadFiles) => {};

const closeViewer = () => {
	dialogVisible.value = false;
};
</script>
  
<style scoped lang="scss">
.upload-info {
	/* margin: 20px auto; */
}

.upload_img_fit {
	object-fit: cover; /* 确保图片覆盖整个容器 */
	object-position: center; /* 确保图片居中 */
}

:deep() {
	.el-upload-list__item {
		display: flex;
		justify-content: center;
		align-items: center;
	}
}
</style>