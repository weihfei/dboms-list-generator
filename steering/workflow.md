# 交互式生成工作流程

本文件描述 DBOMS 列表页面生成器的完整交互流程。

## 第一阶段：需求收集

### 1.1 收集基本信息

通过对话收集以下信息：

| 信息项 | 必填 | 说明 | 示例 |
|-------|------|------|------|
| 模块中文名 | ✅ | 用于页面标题和按钮 | 采购改单申请 |
| 模块英文名 | ✅ | 用于路由、变量命名（小驼峰） | purchaseOrderAmendment |
| 列表字段 | ✅ | 需要展示的表格列 | 编号、订单号、状态、申请人、日期 |
| 搜索字段 | ✅ | 搜索框支持的筛选项 | 订单号、申请人 |
| 是否需要流程状态 | ✅ | 是否有审批流程 | 是/否 |
| 操作按钮 | ⬜ | 列表行操作（默认：编辑、删除、查看） | 编辑、删除、查看 |
| URL 配置前缀 | ⬜ | API URL 的前缀（默认取模块名缩写） | poa |

### 1.2 生成功能点清单

根据收集的信息，生成功能点清单供用户确认：

```markdown
## 功能点清单

根据你的需求，我梳理出以下功能点：

### 📋 页面结构
- [x] 1.1 顶部类型切换器 + 新建{moduleCnName}按钮
- [x] 1.2 流程状态筛选栏（全部/已完成/审批中/草稿）
- [x] 1.3 搜索框（支持{搜索字段}）
- [x] 1.4 数据表格（{列表字段}）
- [x] 1.5 分页组件

### 🔄 交互功能
- [x] 2.1 流程状态切换筛选
- [x] 2.2 搜索/重置
- [x] 2.3 新建跳转
- [x] 2.4 编辑/查看跳转（草稿/驳回→编辑，其他→查看）
- [x] 2.5 单条/批量删除
- [x] 2.6 表格多选

### 📡 数据处理
- [x] 3.1 列表数据加载
- [x] 3.2 分页切换
- [x] 3.3 删除接口调用

---

请确认以上功能点是否符合预期？
1. ✅ 确认，开始生成代码
2. ➕ 需要添加功能点
3. ➖ 需要删除某些功能点
4. ✏️ 需要修改功能点描述
```

## 第二阶段：目录检查

### 2.1 检查目录是否存在

在生成代码前，必须检查 `src/views/{moduleName}/` 目录是否存在。

**检查命令：**
```bash
ls -la src/views/{moduleName}/
```

### 2.2 根据检查结果决定生成策略

#### 情况一：目录不存在

提示用户：
```markdown
## 📁 将创建新模块

`src/views/{moduleName}/` 目录不存在，将生成完整文件结构：

### 🆕 将生成的文件
- `src/views/{moduleName}/api/index.ts` - API 接口
- `src/views/{moduleName}/config/index.ts` - 配置项
- `src/views/{moduleName}/interface/index.ts` - 类型定义
- `src/views/{moduleName}/{listName}/index.vue` - 列表页面

是否继续？
```

#### 情况二：目录已存在

提示用户：
```markdown
## 📁 检测到目录已存在

`src/views/{moduleName}/` 目录已存在，将只生成列表页面文件。

### 🆕 将生成的文件
- `src/views/{moduleName}/{listName}/index.vue`

### 📝 需要您在现有文件中补充

#### api/index.ts
请添加以下方法：
- `getListApi` - 获取列表接口
- `deleteApi` - 删除接口

#### config/index.ts
请添加以下配置：
- `WfTypes` - 流程状态数组（如已存在可复用）
- `DefaultPageInfo` - 分页默认值（如已存在可复用）
- `DefaultSearchOption` - 搜索参数默认值
- `SearchInputOptionsConfig` - 搜索框配置

#### interface/index.ts
请添加以下类型：
- `I{ModuleName}ListReqType` - 列表查询参数类型
- `I{ModuleName}ListItem` - 列表项类型

是否继续生成列表页面？
```

## 第三阶段：代码生成

### 3.1 变量替换

在生成代码时，替换以下占位符：

| 占位符 | 替换为 | 示例 |
|-------|-------|------|
| `{moduleName}` | 模块英文名（小驼峰） | purchaseOrderAmendment |
| `{ModuleName}` | 模块英文名（大驼峰） | PurchaseOrderAmendment |
| `{moduleCnName}` | 模块中文名 | 采购改单申请 |
| `{routePath}` | 路由路径（与模块英文名相同） | purchaseOrderAmendment |
| `{urlKey}` | URL 配置前缀 | poa |
| `{listName}` | 列表目录名（默认：{moduleName}List） | purchaseOrderAmendmentList |

### 3.2 生成表格列

根据用户提供的列表字段，生成对应的 `<el-table-column>`：

**普通文本列：**
```vue
<el-table-column label="{字段名}" prop="{字段属性}" min-width="120" align="center" />
```

**带链接的编号列：**
```vue
<el-table-column label="编号" prop="number" min-width="130" align="center">
	<template #default="{ row }">
		<router-link :to="getRouterLink(row)">
			<el-link type="primary" :underline="false">{{ row.number }}</el-link>
		</router-link>
	</template>
</el-table-column>
```

**日期列：**
```vue
<el-table-column label="申请日期" prop="createTime" min-width="120" align="center" />
```

### 3.3 生成搜索配置

根据用户提供的搜索字段，生成 `SearchInputOptionsConfig`：

```typescript
export const SearchInputOptionsConfig: SearchInputOptionType[] = [
	{
		innerName: 'fieldName1',
		showName: '字段显示名1',
		order: 1
	},
	{
		innerName: 'fieldName2',
		showName: '字段显示名2',
		order: 2
	}
];
```

## 第四阶段：用户补充提示

### 4.1 全新模块的补充提示

```markdown
## ✅ 代码生成完成

### 📝 需要您补充的内容

#### 1. 接口类型 (interface/index.ts)
- [ ] `I{ModuleName}ListReqType` - 补充搜索字段类型
- [ ] `I{ModuleName}ListItem` - 补充列表项字段类型

#### 2. 配置文件 (config/index.ts)
- [ ] `DefaultSearchOption` - 补充搜索字段默认值
- [ ] `SearchInputOptionsConfig` - 补充搜索框下拉选项

#### 3. API 文件 (api/index.ts)
- [ ] 替换 `$Urls.{urlKey}_getList` 为实际的 URL 配置
- [ ] 替换 `$Urls.{urlKey}_delete` 为实际的 URL 配置
- [ ] 在 `src/api/urls/modules/` 下添加对应的 URL 配置

#### 4. 列表页面 ({listName}/index.vue)
- [ ] 检查表格列是否正确
- [ ] 根据业务调整操作按钮逻辑

#### 5. 路由配置
- [ ] 在 `src/routers/` 下添加列表、编辑、详情页路由
```

### 4.2 现有模块新增列表的补充提示

```markdown
## ✅ 列表页面生成完成

已生成文件：`src/views/{moduleName}/{listName}/index.vue`

### 📝 需要您在现有文件中补充

#### 1. api/index.ts - 添加以下方法
```typescript
/**
 * @description 获取{moduleCnName}列表
 */
const get{ModuleName}ListApi = async (params: I{ModuleName}ListReqType & Global.IReqPageType) => {
	try {
		return await $Request({
			url: $Urls.{urlKey}_getList,
			method: $Method.POST,
			data: params
		});
	} catch (err) {
		return Promise.reject(err);
	}
};

/**
 * @description 删除{moduleCnName}
 */
const delete{ModuleName}Api = async (ids: string[]) => {
	try {
		return await $Request({
			url: $Urls.{urlKey}_delete,
			method: $Method.POST,
			data: ids
		});
	} catch (err) {
		return Promise.reject(err);
	}
};

// 记得在 return 中导出
return {
	// ...其他方法
	get{ModuleName}ListApi,
	delete{ModuleName}Api
};
```

#### 2. config/index.ts - 添加以下配置
```typescript
// 如果不存在，添加流程状态数组
export const WfTypes = [
	{ label: '全部', name: ProcessStatusEnum.ALL },
	{ label: '已完成', name: ProcessStatusEnum.COMPLETED },
	{ label: '审批中', name: ProcessStatusEnum.APPROVING },
	{ label: '草稿', name: ProcessStatusEnum.DRAFT }
];

// 如果不存在，添加分页默认值
export const DefaultPageInfo = () => ({
	current: 1,
	size: 10,
	total: 0
});

// 添加搜索参数默认值
export const Default{ModuleName}SearchOption = (): I{ModuleName}ListReqType => ({
	// 根据搜索字段补充
	processStatus: ProcessStatusEnum.ALL
});

// 添加搜索框配置
export const {ModuleName}SearchInputOptionsConfig: SearchInputOptionType[] = [
	// 根据搜索字段补充
];
```

#### 3. interface/index.ts - 添加以下类型
```typescript
/**
 * {moduleCnName}列表查询参数类型
 */
export interface I{ModuleName}ListReqType {
	/** 流程状态 */
	processStatus?: ProcessStatusEnum;
	// 根据搜索字段补充
	[key: string]: any;
}

/**
 * {moduleCnName}列表项类型
 */
export interface I{ModuleName}ListItem {
	/** 主键ID */
	id: string;
	/** 编号 */
	number: string;
	/** 流程状态 */
	processStatus: ProcessStatusEnum;
	// 根据列表字段补充
}
```

#### 4. 列表页面调整
- [ ] 检查导入路径是否正确
- [ ] 检查表格列是否正确
- [ ] 根据业务调整操作按钮逻辑

#### 5. 路由配置
- [ ] 在 `src/routers/` 下添加列表页路由
```

## 特殊场景处理

### 不需要流程状态的列表

如果用户表示不需要流程状态筛选：

1. 移除 `<db-status-type-switcher>` 组件
2. 移除流程状态相关的表格列
3. 移除 `changeProcessStatus` 方法
4. 简化 `searchParams` 类型定义
5. 调整操作列逻辑（不再根据流程状态判断）

### 自定义操作按钮

如果用户需要自定义操作按钮：

1. 根据需求调整操作列模板
2. 添加对应的方法定义
3. 如需要，添加相关的 API 调用

### 额外的筛选条件

如果用户需要额外的筛选条件（如日期范围、下拉选择等）：

1. 在搜索区域添加对应的组件
2. 在 `searchParams` 中添加对应字段
3. 在 `DefaultSearchOption` 中添加默认值
4. 在 `resetSearchParams` 中处理重置逻辑
