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
					<el-button @click="viewPanos(scope.row.id)">查看</el-button>
				</template>
			</cl-table>
		</cl-row>

		<cl-row>
			<cl-flex1 />
			<!-- 分页控件 -->
			<cl-pagination />
		</cl-row>

		<!-- 新增、编辑 -->
		<cl-upsert ref="Upsert" />
	</cl-crud>
</template>

<script lang="ts" name="project-project" setup>
import { useCrud, useTable, useUpsert } from "@cool-vue/crud";
import { useCool } from "/@/cool";

const { service, router } = useCool();

// cl-upsert
const Upsert = useUpsert({
	items: [
		{
			label: "项目名称",
			prop: "name",
			component: { name: "el-input", props: { clearable: true } },
			required: true
		},
		{ label: "地图图片地址", prop: "mapSrc", component: { name: "cl-upload" } },
		// {
		// 	label: "地图宽",
		// 	prop: "mapWidth",
		// 	hook: "number",
		// 	component: { name: "el-input-number" }
		// },
		// {
		// 	label: "地图高",
		// 	prop: "mapHeight",
		// 	hook: "number",
		// 	component: { name: "el-input-number" }
		// },
		{
			label: "描述",
			prop: "description",
			component: { name: "el-input", props: { type: "textarea", rows: 4 } }
		}
	]
});

// cl-table
const Table = useTable({
	columns: [
		{ type: "selection" },
		{ label: "项目名称", prop: "name", minWidth: 140 },
		{
			label: "地图图片地址",
			prop: "mapSrc",
			minWidth: 100,
			component: { name: "cl-image", props: { size: 60 } }
		},
		// { label: "地图宽", prop: "mapWidth", minWidth: 140 },
		// { label: "地图高", prop: "mapHeight", minWidth: 140 },
		{ label: "描述", prop: "description", showOverflowTooltip: true, minWidth: 200 },
		{
			label: "创建时间",
			prop: "createTime",
			minWidth: 160,
			component: { name: "cl-date-text" }
		},
		{
			label: "更新时间",
			prop: "updateTime",
			minWidth: 160,
			component: { name: "cl-date-text" }
		},
		{ type: "op", buttons: ["edit", "delete", "slot-view"] }
	]
});

const viewPanos = (id: string) => {
	router.push("/panos/list?project_id=" + id);
};

// cl-crud
const Crud = useCrud(
	{
		service: service.project.project
	},
	(app) => {
		app.refresh();
	}
);
</script>
