# 列表页面模板

本文件包含 DBOMS 列表页面的完整代码模板和规范。

## 列表页面模板 (index.vue)

```vue
<template>
	<div class="app-container" v-loading="loading" :element-loading-text="LoadingTextEnum.LOADING">
		<!-- 顶部 新建按钮 -->
		<db-type-switcher class="w-100" :tabs="moduleTab" v-model="tabActiveName">
			<template #custom-button>
				<e-button type="primary" :icon="'Plus'" @click="handleCreate">新建{moduleCnName}</e-button>
			</template>
		</db-type-switcher>

		<!-- 顶部搜索 -->
		<div class="flx-justify-between">
			<!-- 左侧流程状态 -->
			<db-status-type-switcher v-model="searchParams.processStatus" :types="WfTypes" @btn-click="changeProcessStatus" />

			<!-- 右侧搜索 重置 批量删除 -->
			<div class="search-wrapper-right">
				<transition-group name="slide-fade" tag="div" class="button-container">
					<db-custom-input-search
						key="search"
						v-model="searchInputQuery"
						:options="searchInputOptions"
						@change:values="changeSearchInput"
					/>
					<e-button key="reset" @click="resetSearchParams">重置</e-button>
					<e-button key="batch-delete" type="danger" text v-if="isShowBatchDelBtn" @click="batchDelete">
						批量删除
					</e-button>
				</transition-group>
			</div>
		</div>

		<!-- 列表 -->
		<e-table
			border
			:max-height="tableMaxHeight"
			:table-data="dataList"
			@selection-change="handleSelectionChange"
			show-overflow-tooltip
			class="table-container"
		>
			<!-- 选择列 -->
			<el-table-column type="selection" width="45" />

			<!-- TODO: 根据需求添加表格列 -->
			<!-- 示例：编号列（带链接跳转） -->
			<el-table-column label="编号" prop="number" min-width="130" align="center">
				<template #default="{ row }">
					<router-link :to="getRouterLink(row)">
						<el-link type="primary" :underline="false">{{ row.number }}</el-link>
					</router-link>
				</template>
			</el-table-column>

			<!-- 流程状态列（条件显示） -->
			<el-table-column
				label="流程状态"
				prop="processStatus"
				min-width="130"
				align="center"
				v-if="[ProcessStatusEnum.ALL, ProcessStatusEnum.DRAFT].includes(searchParams.processStatus as ProcessStatusEnum)"
			>
				<template #default="{ row }">
					{{ ProcessStatusEnumDescription[row.processStatus as ProcessStatusEnum] }}
				</template>
			</el-table-column>

			<!-- 当前审批环节列（条件显示） -->
			<el-table-column
				label="当前审批环节"
				prop="currentApprovalStep"
				min-width="142"
				align="center"
				v-if="[ProcessStatusEnum.ALL, ProcessStatusEnum.APPROVING].includes(searchParams.processStatus as ProcessStatusEnum)"
			>
				<template #default="{ row }">
					<el-tooltip
						placement="top-start"
						:content="row.nodeApproveUsers ? row.nodeApproveUsers.join(',') : ''"
						popper-class="poper-max-width"
						v-if="row.nodeApproveUsers"
					>
						<e-button size="small" :type="'default'">{{ row.nodeName }}</e-button>
					</el-tooltip>
				</template>
			</el-table-column>

			<!-- 操作列 -->
			<el-table-column label="操作" width="140" fixed="right" :show-overflow-tooltip="false">
				<template #default="{ row }">
					<div v-if="[ProcessStatusEnum.DRAFT, ProcessStatusEnum.REJECTED].includes(row.processStatus)">
						<e-button type="primary" text bg size="small" @click="goDetail(row)" style="margin-right: 10px">编辑</e-button>
						<e-button type="danger" text bg size="small" @click="batchDelete(row)">删除</e-button>
					</div>
					<div v-else>
						<e-button type="primary" text bg size="small" @click="goDetail(row)">查看</e-button>
					</div>
				</template>
			</el-table-column>
		</e-table>

		<!-- 分页 -->
		<e-pagination
			background
			layout="total, sizes, prev, pager, next, jumper"
			:page-info="pageInfo"
			@page-event="handlePageChange"
		></e-pagination>
	</div>
</template>

<script setup lang="ts">
// ==================== 1. 导入声明 ====================
// 1.1 Vue 核心
import { ref, onMounted, onBeforeUnmount, computed, getCurrentInstance } from 'vue';
// 1.2 路由相关
import { useRouter } from 'vue-router';
// 1.3 枚举导入
import { ProcessStatusEnum, ProcessStatusEnumDescription } from '@/enums/processStatusEnum';
import { LoadingTextEnum } from '@/enums/loadingTextEnum';
// 1.4 配置导入
import { WfTypes, DefaultPageInfo, DefaultSearchOption, SearchInputOptionsConfig } from '../config/index';
// 1.5 类型导入
import type { I{ModuleName}ListReqType, I{ModuleName}ListItem } from '../interface/index';
import type { SearchInputOptionType } from '@/interface/inputSearchOptionConfigResponseType';
// 1.6 基础配置
import { WEB_MODE, DEFAULT_HEIGHT } from '@/config/config';
// 1.7 API 导入
import { {ModuleName}Request } from '../api/index';

// ==================== 2. API 解构 ====================
const { getListApi, deleteApi } = {ModuleName}Request();

// ==================== 3. 全局变量 ====================
const {
	proxy: { $Modal }
} = getCurrentInstance() as any;

// ==================== 4. 路由实例 ====================
const router = useRouter();

// ==================== 5. 响应式数据 ====================
// 5.1 Tab 配置
const moduleTab = ref<{ label: string; name: string }[]>([
	{ label: '{moduleCnName}', name: '{moduleName}' }
]);
const tabActiveName = ref<string>('{moduleName}');

// 5.2 搜索相关
const searchParams = ref<I{ModuleName}ListReqType>(DefaultSearchOption());
const searchInputQuery = ref<string[]>([]);
const searchInputOptions = ref<SearchInputOptionType[]>(SearchInputOptionsConfig);

// 5.3 状态
const loading = ref(false);
const tableMaxHeight = ref(0);

// 5.4 列表数据
const selectedRows = ref<I{ModuleName}ListItem[]>([]);
const pageInfo = ref<Global.IPageInfoType>(DefaultPageInfo());
const dataList = ref<I{ModuleName}ListItem[]>([]);

// ==================== 6. 计算属性 ====================
const isShowBatchDelBtn = computed(() => {
	return selectedRows.value.length > 0;
});

// ==================== 7. 方法定义 ====================
// 7.1 流程状态切换
const changeProcessStatus = (val: ProcessStatusEnum) => {
	searchParams.value.processStatus = val;
	pageInfo.value = DefaultPageInfo();
	getList();
};

// 7.2 清空搜索条件
const clearInputSearch = () => {
	searchInputOptions.value.forEach((item: SearchInputOptionType) => {
		searchParams.value[item.innerName] = DefaultSearchOption()[item.innerName];
	});
};

// 7.3 搜索框变化
const changeSearchInput = (selectedItems: any[]) => {
	if (!selectedItems.length) {
		const originalProcessStatus = searchParams.value.processStatus;
		searchParams.value = DefaultSearchOption();
		searchParams.value.processStatus = originalProcessStatus;
	} else {
		clearInputSearch();
		selectedItems.forEach(item => {
			const [key, value] = item.split(':');
			searchParams.value[key] = value;
		});
	}
	getList();
};

// 7.4 重置搜索
const resetSearchParams = () => {
	searchParams.value = DefaultSearchOption();
	searchInputQuery.value = [];
	pageInfo.value = DefaultPageInfo();
	getList();
};

// 7.5 批量删除
const batchDelete = async (row?: I{ModuleName}ListItem) => {
	try {
		if (await $Modal.confirm(`您确定要删除?`)) {
			const ids = row?.id ? [row?.id] : selectedRows.value.map((item: I{ModuleName}ListItem) => item.id);
			await deleteApi(ids);
			selectedRows.value = [];
			getList();
			$Modal.msgSuccess('删除成功');
		}
	} catch (err) {
		console.warn(err);
	}
};

// 7.6 新建
const handleCreate = () => {
	router.push('/{routePath}/edit');
};

// 7.7 获取路由链接
const getRouterLink = (row: I{ModuleName}ListItem) => {
	if (row.processStatus === ProcessStatusEnum.DRAFT || row.processStatus === ProcessStatusEnum.REJECTED) {
		return `/{routePath}/edit?id=${row.id}`;
	} else {
		return `/{routePath}/detail?id=${row.id}`;
	}
};

// 7.8 跳转详情
const goDetail = (row: I{ModuleName}ListItem) => {
	router.push(getRouterLink(row));
};

// 7.9 表格选择
const handleSelectionChange = (selection: I{ModuleName}ListItem[]) => {
	selectedRows.value = selection;
};

// 7.10 分页变化
const handlePageChange = (newPageInfo: Global.IPageInfoType) => {
	pageInfo.value = newPageInfo;
	getList();
};

// 7.11 计算表格高度
const calculateMaxHeight = () => {
	let height = DEFAULT_HEIGHT;
	if (WEB_MODE === 'vertical') {
		height += 85;
	}
	tableMaxHeight.value = window.innerHeight - height;
};

// 7.12 格式化查询参数
const formatSearchParams = (): I{ModuleName}ListReqType => {
	const _searchParams: I{ModuleName}ListReqType = { ...searchParams.value };
	_searchParams.processStatus = _searchParams.processStatus === ProcessStatusEnum.ALL ? undefined : _searchParams.processStatus;
	return _searchParams;
};

// 7.13 获取列表数据
const getList = async () => {
	loading.value = true;
	const params = formatSearchParams();
	
	try {
		const resData: Global.IResultType<any> = await getListApi({
			...params,
			...pageInfo.value
		});
		dataList.value = resData?.data?.result || [];
		pageInfo.value.total = resData?.data?.total || 0;
	} catch (err) {
		console.error('获取列表失败:', err);
		$Modal.msgError('获取列表失败');
	} finally {
		loading.value = false;
	}
};

// ==================== 8. 生命周期 ====================
onMounted(() => {
	getList();
	calculateMaxHeight();
	window.addEventListener('resize', calculateMaxHeight);
});

onBeforeUnmount(() => {
	window.removeEventListener('resize', calculateMaxHeight);
});
</script>

<style lang="scss" scoped>
.search-wrapper-right {
	display: flex;
	justify-content: end;
	width: 65%;
	flex: 1;
}

.table-container {
	margin-top: 10px;
}

.slide-fade-enter-active {
	transition: all 0.3s ease;
}

.slide-fade-leave-active {
	transition: all 0.3s ease;
	position: absolute;
}

.slide-fade-enter-from,
.slide-fade-leave-to {
	transform: translateX(20px);
	opacity: 0;
}

.slide-fade-move {
	transition: transform 0.3s ease;
}

.button-container {
	display: flex;
	gap: 0px;
	position: relative;
	width: 100%;
	justify-content: flex-end;
	align-items: center;
}
</style>
```

## 配置文件模板 (config/index.ts)

```typescript
/**
 * @description {moduleCnName}相关配置
 */
import { ProcessStatusEnum } from '@/enums/processStatusEnum';
import type { SearchInputOptionType } from '@/interface/inputSearchOptionConfigResponseType';
import type { I{ModuleName}ListReqType } from '../interface/index';

// 流程状态数组
export const WfTypes = [
	{ label: '全部', name: ProcessStatusEnum.ALL },
	{ label: '已完成', name: ProcessStatusEnum.COMPLETED },
	{ label: '审批中', name: ProcessStatusEnum.APPROVING },
	{ label: '草稿', name: ProcessStatusEnum.DRAFT }
];

// 分页信息初始值
export const DefaultPageInfo = () => ({
	current: 1,
	size: 10,
	total: 0
});

// 列表搜索参数初始化
export const DefaultSearchOption = (): I{ModuleName}ListReqType => ({
	// TODO: 根据实际搜索字段补充
	processStatus: ProcessStatusEnum.ALL
});

// 搜索框配置
export const SearchInputOptionsConfig: SearchInputOptionType[] = [
	// TODO: 根据实际搜索字段补充
	// {
	// 	innerName: 'fieldName',
	// 	showName: '字段显示名',
	// 	order: 1
	// }
];
```

## 接口类型模板 (interface/index.ts)

```typescript
/**
 * @description {moduleCnName}相关类型定义
 */
import { ProcessStatusEnum } from '@/enums/processStatusEnum';

/**
 * 列表查询条件
 */
export interface I{ModuleName}ListReqType {
	/** 流程状态 */
	processStatus?: ProcessStatusEnum;
	// TODO: 根据实际搜索字段补充
	[key: string]: any;
}

/**
 * 列表项类型
 */
export interface I{ModuleName}ListItem {
	/** 主键ID */
	id: string;
	/** 编号 */
	number: string;
	/** 流程状态 */
	processStatus: ProcessStatusEnum;
	/** 当前审批环节节点名称 */
	nodeName?: string;
	/** 当前审批环节节点审批人 */
	nodeApproveUsers?: string[];
	// TODO: 根据实际列表字段补充
}
```

## API 模板 (api/index.ts)

```typescript
/**
 * @description {moduleCnName} API
 */
import { getCurrentInstance } from 'vue';
import type { I{ModuleName}ListReqType } from '../interface/index';

export const {ModuleName}Request = () => {
	const {
		proxy: { $Urls, $Request, $Method }
	} = getCurrentInstance() as any;

	/**
	 * @description 获取列表
	 */
	const getListApi = async (params: I{ModuleName}ListReqType & Global.IReqPageType) => {
		try {
			return await $Request({
				url: $Urls.{urlKey}_getList, // TODO: 替换为实际URL配置
				method: $Method.POST,
				data: params
			});
		} catch (err) {
			return Promise.reject(err);
		}
	};

	/**
	 * @description 删除
	 */
	const deleteApi = async (ids: string[]) => {
		try {
			return await $Request({
				url: $Urls.{urlKey}_delete, // TODO: 替换为实际URL配置
				method: $Method.POST,
				data: ids
			});
		} catch (err) {
			return Promise.reject(err);
		}
	};

	return {
		getListApi,
		deleteApi
	};
};
```

## 代码块分类顺序规范

生成的代码必须按以下顺序组织：

1. **导入声明** - Vue 核心、路由、枚举、配置、类型、API
2. **API 解构** - 从 Request 函数中解构需要的方法
3. **全局变量** - $Modal 等全局挂载
4. **路由实例** - useRouter()
5. **响应式数据** - Tab、搜索、状态、列表数据
6. **计算属性** - computed
7. **方法定义** - 按功能分组（状态切换、搜索、删除、跳转、分页、数据加载）
8. **生命周期** - onMounted、onBeforeUnmount

## 组件使用规范

**必须使用封装组件（自动导入）：**
- `<e-button>` 替代 `<el-button>`
- `<e-input>` 替代 `<el-input>`
- `<e-select>` 替代 `<el-select>`
- `<e-table>` 替代 `<el-table>`
- `<e-dialog>` 替代 `<el-dialog>`
- `<e-pagination>` 替代 `<el-pagination>`

**可以直接使用 Element UI 的组件：**
- `<el-form>` / `<el-form-item>`
- `<el-table-column>`
- `<el-card>`
- `<el-tooltip>`
- `<el-link>`

## TypeScript 规范

**禁止使用 `any`（除全局变量外）：**
```typescript
// ❌ 错误
const data: any = ref(null);

// ✅ 正确
interface DataItem {
  id: number;
  name: string;
}
const data = ref<DataItem | null>(null);
```
