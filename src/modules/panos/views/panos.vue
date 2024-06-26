<template>
	<cl-crud ref="Crud">
		<cl-row>
			<!-- 刷新按钮 -->
			<cl-refresh-btn />
			<!-- 新增按钮 -->
			<cl-add-btn />
			<!-- 删除按钮 -->
			<cl-multi-delete-btn />
			<cl-flex1 />
			<!-- 关键字搜索 -->
			<cl-search-key />
		</cl-row>

		<cl-row>
			<!-- 数据表格 -->
			<cl-table ref="Table">
				<template #slot-view="{ scope }">
					<el-button @click="toPano(scope)">查看</el-button>
				</template>
				<template #column-position="{ scope }">
					<el-button @click="viewMapPosition(scope)" :disabled="!scope.row.position"
						>查看</el-button
					>
				</template>
				<template #column-url="{ scope }">
					<div>
						<el-link
							:href="`${origin}/pano?pano_id=${scope.row.id}&project_id=${scope.row.projectId}`"
							>{{
								`${origin}/pano?pano_id=${scope.row.id}&project_id=${scope.row.projectId}`
							}}</el-link
						>
					</div>
				</template>
			</cl-table>
		</cl-row>

		<cl-row>
			<cl-flex1 />
			<!-- 分页控件 -->
			<cl-pagination />
		</cl-row>

		<!-- 新增、编辑 -->
		<cl-upsert ref="Upsert">
			<template #slot-change-map-position="{ scope }">
				<el-button
					:disabled="!scope.projectId || !findMapSrc(scope.projectId)"
					@click="changePosition(scope)"
					>Edit position</el-button
				>
			</template>
		</cl-upsert>
		<cl-dialog
			class="change-map-position-dialog"
			:align-center="true"
			title="Change position"
			width="500px"
			:controls="['close']"
			v-model="isChangePositionDialogOpen"
		>
			<div style="background-color: white; position: relative" @click="addPositionNode">
				<img
					:src="findMapSrc(mapScope?.projectId)"
					alt=""
					style="width: 100%; display: block"
				/>
				<div
					v-if="temporaryPosition"
					:style="{
						width: dottedSize + 'px',
						height: dottedSize + 'px',
						position: 'absolute',
						left: temporaryPosition?.left + '%',
						top: temporaryPosition?.top + '%',
						zIndex: 180,
						background: 'blue',
						borderRadius: '50%'
					}"
				></div>
			</div>
			<div style="display: flex; justify-content: flex-end">
				<el-button @click="isChangePositionDialogOpen = false">取消</el-button>
				<el-button type="primary" @click="confirmPositionChange">完成</el-button>
			</div>
		</cl-dialog>
		<cl-dialog
			class="view-map-position-dialog"
			:align-center="true"
			title="View position"
			width="500px"
			:controls="['close']"
			v-model="viewPositionDialog.visible"
		>
			<div style="background-color: white; position: relative">
				<img
					:src="viewPositionDialog.mapSrc ?? ''"
					alt=""
					style="width: 100%; display: block"
				/>
				<div
					v-if="viewPositionDialog.currentPosition"
					:style="{
						width: dottedSize + 'px',
						height: dottedSize + 'px',
						position: 'absolute',
						left: viewPositionDialog.currentPosition?.left + '%',
						top: viewPositionDialog.currentPosition?.top + '%',
						zIndex: 180,
						background: 'blue',
						borderRadius: '50%'
					}"
				></div>
			</div>
		</cl-dialog>
	</cl-crud>
</template>

<script lang="ts" name="panos-panos" setup>
import { useCrud, useTable, useUpsert } from "@cool-vue/crud";
import { useCool } from "/@/cool";
import { ref } from "vue";
const { service, router, route } = useCool();

const projectId = ref(route.query?.project_id || null);
const projectsList = ref<any>([]);
// 设置全景图在地图中的位置路由
const findMapSrc = (id: number) => {
	const mapItem = projectsList.value.find((v: any) => v.id === id);
	if (mapItem) {
		return mapItem.mapSrc;
	}
	return null;
};
const mapScope = ref<any>(null);
const changePosition = (scope: any) => {
	mapScope.value = scope;
	if (scope.position) temporaryPosition.value = scope.position;
	isChangePositionDialogOpen.value = true;
};
const dottedSize = 10;
const isChangePositionDialogOpen = ref(false);
const temporaryPosition = ref<any>(null);
const addPositionNode = (e: any) => {
	const parentWidth = e.currentTarget.offsetWidth;
	const parentHeight = e.currentTarget.offsetHeight;
	const position = {
		left: (e.offsetX / parentWidth) * 100,
		top: (e.offsetY / parentHeight) * 100
	};
	temporaryPosition.value = position;
};
const confirmPositionChange = () => {
	isChangePositionDialogOpen.value = false;
	mapScope.value.position = temporaryPosition.value;
	temporaryPosition.value = null;
};

// 查看全景图在地图中的位置
const viewPositionDialog = ref<{
	visible: boolean;
	currentPosition: { left: number; top: number } | null;
	mapSrc: string | null;
}>({
	visible: false,
	currentPosition: null,
	mapSrc: null
});
const viewMapPosition = (scope: any) => {
	viewPositionDialog.value.currentPosition = scope.row.position;
	viewPositionDialog.value.mapSrc = scope.row.mapSrc;
	viewPositionDialog.value.visible = true;
};

// 跳往pano
const toPano = (scope: any) => {
	router.push(`/panos/view?pano_id=${scope.row.id}&project_id=${scope.row.projectId}`);
};

// cl-upsert
const Upsert = useUpsert({
	props: {
		labelWidth: "200px"
	},
	items: [
		{
			label: "标题",
			prop: "title",
			component: { name: "el-input", props: { clearable: true } },
			required: true
		},
		{
			label: "全景图文件地址",
			prop: "panoSrc",
			component: { name: "cl-upload", props: { type: "file", limit: 1, limitSize: 1000 } },
			required: true
		},
		{
			prop: "projectId",
			label: "项目编号",
			value: "",
			component: {
				name: "el-select",
				options: [],
				props: {
					placeholder: "Please choose department"
				}
			}
		},
		{
			label: "序号",
			prop: "no",
			component: { name: "el-input", props: { type: "number" } },
			required: false
		},
		{
			label: "相对于地图的位置",
			prop: "position",
			component: { name: "slot-change-map-position" }
		},
		{
			label: "缩略图[200x200]",
			prop: "thumb",
			component: { name: "cl-upload" },
			required: true
		},
		{
			label: "描述",
			prop: "description",
			component: { name: "el-input", props: { type: "textarea", rows: 4 } }
		},
		{
			label: "背景音乐地址",
			prop: "music",
			component: { name: "cl-upload", props: { type: "file", limit: 1 } }
		}
	],
	async onOpen() {
		projectsList.value = await service.project.project.list();
		// 设置项目列表
		Upsert.value?.setOptions(
			"projectId",
			projectsList.value.map((e) => {
				return {
					label: e.name || "",
					value: e.id
				};
			})
		);
	}
});
const origin = window.location.origin;
// cl-table
const Table = useTable({
	columns: [
		{ type: "selection" },
		{ label: "标题", prop: "title", minWidth: 140 },
		{
			label: "全景图文件地址",
			prop: "panoSrc",
			minWidth: 120,
			component: { name: "cl-link" }
		},
		{ label: "项目", prop: "name", minWidth: 140 },
		{
			label: "相对于地图的位置",
			prop: "position",
			minWidth: 120
		},
		{
			label: "缩略图",
			prop: "thumb",
			minWidth: 100,
			component: { name: "cl-image", props: { size: 60 } }
		},
		{ label: "序号", prop: "no" },
		{
			label: "链接",
			prop: "url",
			minWidth: 180,
			showOverflowTooltip: true,

			formatter: (row, column) =>
				`${origin}/pano?pano_id=${row.id}&project_id=${row.projectId}`
		},
		{ label: "描述", prop: "description", showOverflowTooltip: true, minWidth: 200 },
		{ label: "背景音乐地址", prop: "music", minWidth: 120, component: { name: "cl-link" } },
		{
			label: "创建时间",
			prop: "createTime",
			minWidth: 160,
			component: { name: "cl-date-text" }
		},
		{ type: "op", buttons: ["edit", "delete", "slot-view"] }
	]
});

// cl-crud
const Crud = useCrud(
	{
		service: service.panos?.panos
	},
	(app) => {
		app.refresh({
			projectId: projectId.value ? projectId.value : undefined,
			prop: "no",
			sort: "desc"
		});
	}
);
</script>
