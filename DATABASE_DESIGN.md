# 数据库接入设计文档

本文档说明如何在芋道管理系统（前端）中接入自己的数据库，包括后端数据库配置、代码生成器使用、前端模块开发规范和字段映射参考。

## 一、数据库接入架构概述

本系统采用**前后端分离**架构，前端不直接连接数据库，而是通过 RESTful API 与后端服务通信。后端基于 [芋道(yudao)云框架](https://github.com/YunaiV/ruoyi-vue-pro)（Spring Boot 微服务架构），负责数据库操作和业务逻辑。

### 数据流转架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         前端应用层                               │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Views 页面 → Adapter 适配器 → API 层 → requestClient      │  │
│  └───────────────────────────┬───────────────────────────────┘  │
└──────────────────────────────┼──────────────────────────────────┘
                               │ Axios HTTP 请求 (JSON)
                               │ • Authorization: Bearer Token
                               │ • tenant-id: 租户编号
                               │ • Content-Type: application/json
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API 网关 (Gateway)                          │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  鉴权 → 限流 → 路由 → 日志 → 后端微服务                     │  │
│  └───────────────────────────┬───────────────────────────────┘  │
└──────────────────────────────┼──────────────────────────────────┘
                               │
┌──────────────────────────────┼──────────────────────────────────┐
│                      后端服务层 (Spring Boot)                    │
│  ┌───────────────────────────┴───────────────────────────────┐  │
│  │  Controller → Service → Mapper → MyBatis-Plus             │  │
│  │  (REST API)    (业务逻辑)   (数据访问)   (ORM框架)         │  │
│  └───────────────────────────┬───────────────────────────────┘  │
└──────────────────────────────┼──────────────────────────────────┘
                               │ JDBC / MyBatis
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                         数据存储层                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │  MySQL   │  │  Redis   │  │ RocketMQ │  │  MinIO   │         │
│  │ 主数据库  │  │  缓存    │  │ 消息队列  │  │ 文件存储  │         │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

### 前端 API 请求/响应格式

**统一响应结构**：

```json
{
  "code": 0,
  "data": { ... },
  "msg": ""
}
```

- `code`: 业务状态码，`0` 表示成功，`401` 表示未登录
- `data`: 业务数据
- `msg`: 提示消息

**分页请求参数**：

```
pageNo: 页码（从1开始）
pageSize: 每页条数
sortingFields[0].field: 排序字段
sortingFields[0].order: asc/desc
```

**分页响应结构**：

```json
{
  "code": 0,
  "data": {
    "list": [...],
    "total": 100
  }
}
```

---

## 二、后端数据库配置说明

> 注：本节内容涉及后端配置，前端开发者可参考配置数据源连接。

### 2.1 主数据库配置 (MySQL)

后端 `application.yaml` 中的数据库配置：

```yaml
spring:
  datasource:
    druid:
      master:
        url: jdbc:mysql://127.0.0.1:3306/yudao-vue-pro?useSSL=false&serverTimezone=Asia/Shanghai&allowPublicKeyRetrieval=true&nullCatalogMeansCurrent=true
        username: root
        password: root
        driver-class-name: com.mysql.cj.jdbc.Driver
      slave:
        # 从库配置（可选）
        enabled: false
```

### 2.2 多数据源配置

系统支持多数据源，可通过代码生成器接入不同数据库：

```yaml
# 在 infra_data_source_config 表中配置
id: 主键
name: 数据源名称
url: JDBC 连接 URL
username: 用户名
password: 密码
driver_class_name: 驱动类名
```

代码生成器支持从多个数据源读取表结构并生成代码。

### 2.3 Redis 缓存配置

```yaml
spring:
  data:
    redis:
      host: 127.0.0.1
      port: 6379
      database: 0
      password:
```

---

## 三、使用代码生成器接入新数据库表（推荐方式）

代码生成器是接入新数据库表的**最快捷方式**，可一键生成前后端完整代码。

### 3.1 代码生成流程图

```
┌────────────────────────────────────────────────────────────────┐
│                      代码生成流程                               │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. 创建数据库表                                               │
│     │                                                          │
│     ▼                                                          │
│  2. 基础设施 → 代码生成 → 导入表结构                           │
│     │  (选择数据源 → 读取数据库表列表 → 导入)                   │
│     ▼                                                          │
│  3. 配置生成信息                                               │
│     │  • 表名/表描述  • 模块名/业务名  • 类名/类描述           │
│     │  • 作者  • 父菜单  • 模板类型(单表/树表/主子表)           │
│     ▼                                                          │
│  4. 配置字段信息                                               │
│     │  • Java类型/字段名  • 字典类型  • 表单/列表显示         │
│     │  • 查询条件配置  • HTML组件类型                          │
│     ▼                                                          │
│  5. 预览/下载生成的代码                                        │
│     │  包含: Controller/Service/Mapper/Entity                  │
│     │  + Vue页面/API定义/菜单SQL                               │
│     ▼                                                          │
│  6. 将前端代码放入对应目录，重启前端服务                       │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 3.2 创建数据库表示例

以 `busi_demo`（示例业务表）为例：

```sql
CREATE TABLE `busi_demo` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `name` varchar(100) NOT NULL COMMENT '名称',
  `status` tinyint NOT NULL DEFAULT 0 COMMENT '状态（0正常 1停用）',
  `type` int DEFAULT NULL COMMENT '类型',
  `category` varchar(50) DEFAULT NULL COMMENT '分类',
  `content` text COMMENT '内容',
  `amount` decimal(12,2) DEFAULT 0 COMMENT '金额',
  `start_time` datetime DEFAULT NULL COMMENT '开始时间',
  `end_time` datetime DEFAULT NULL COMMENT '结束时间',
  `user_id` bigint DEFAULT NULL COMMENT '用户ID',
  `dept_id` bigint DEFAULT NULL COMMENT '部门ID',
  `remark` varchar(500) DEFAULT NULL COMMENT '备注',
  `creator` varchar(64) DEFAULT '' COMMENT '创建者',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updater` varchar(64) DEFAULT '' COMMENT '更新者',
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted` bit(1) NOT NULL DEFAULT b'0' COMMENT '是否删除',
  `tenant_id` bigint NOT NULL DEFAULT 0 COMMENT '租户编号',
  PRIMARY KEY (`id`),
  KEY `idx_name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='示例业务表';
```

### 3.3 代码生成器 API 参考

前端通过代码生成器 API 与后端交互：

[infra/codegen/index.ts](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/api/infra/codegen/index.ts) 中定义的核心接口：

```typescript
// 1. 获取数据库表列表（用于导入）
export function getSchemaTableList(params: any) {
  return requestClient.get<InfraCodegenApi.DatabaseTable[]>(
    '/infra/codegen/db/table/list',
    { params },
  );
}

// 2. 基于数据库表创建代码生成定义
export function createCodegenList(data: InfraCodegenApi.CodegenCreateListReqVO) {
  return requestClient.post('/infra/codegen/create-list', data);
}
// 请求参数：{ dataSourceConfigId: number, tableNames: string[] }

// 3. 查询代码生成表定义列表
export function getCodegenTablePage(params: PageParam) {
  return requestClient.get<PageResult<InfraCodegenApi.CodegenTable>>(
    '/infra/codegen/table/page',
    { params },
  );
}

// 4. 获取代码生成详情（表配置 + 字段配置）
export function getCodegenTable(tableId: number) {
  return requestClient.get<InfraCodegenApi.CodegenDetail>(
    '/infra/codegen/detail',
    { params: { tableId } },
  );
}

// 5. 更新代码生成配置（表配置 + 字段配置）
export function updateCodegenTable(data: InfraCodegenApi.CodegenUpdateReqVO) {
  return requestClient.put('/infra/codegen/update', data);
}

// 6. 同步数据库表结构（更新字段定义）
export function syncCodegenFromDB(tableId: number) {
  return requestClient.put('/infra/codegen/sync-from-db', {}, { params: { tableId } });
}

// 7. 预览生成的代码
export function previewCodegen(tableId: number) {
  return requestClient.get<InfraCodegenApi.CodegenPreview[]>(
    '/infra/codegen/preview',
    { params: { tableId } },
  );<[PLHD79_never_used_51bce0c785ca2f68081bfa7d91973934]># 数据库接入设计文档

本文档说明如何在芋道管理中台前端框架中接入自定义数据库，包含数据库配置、代码生成器使用、前端模块开发规范，以及各字段类型的代码片段和应用示例。

## 一、整体架构说明

本系统前端**不直接连接数据库**，而是通过统一的 RESTful API 层对接后端芋道(yudao) Spring Boot 微服务框架，由后端负责数据库的 CRUD 操作。数据库接入的核心流程如下：

```
┌─────────────────────────────────────────────────────────────────────┐
│                          数据库接入流程                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────┐ │
│  │ 1.创建数据库 │    │ 2.后端配置   │    │ 3.代码生成器自动生成     │ │
│  │   表结构    │───▶│ 数据源/实体  │───▶│ 后端Java代码 + 前端Vue  │ │
│  └─────────────┘    └─────────────┘    │ API/页面代码            │ │
│                                       └───────────┬─────────────┘ │
│                                                   │               │
│  ┌─────────────┐    ┌─────────────┐    ┌───────────▼─────────────┐ │
│  │ 5.功能测试  │    │ 4.前端模块   │    │ • API 接口定义          │ │
│  │   验证      │◀───│   配置菜单   │◀───│ • 表单Schema           │ │
│  └─────────────┘    └─────────────┘    │ • 表格列定义            │ │
│                                       │ • 视图页面              │ │
│                                       └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、后端数据库配置

### 2.1 数据库环境要求

| 组件 | 推荐版本 | 说明 |
|------|----------|------|
| MySQL | 8.0+ | 主数据库，支持 InnoDB 引擎、utf8mb4 字符集 |
| Redis | 6.0+ | 缓存、会话、分布式锁 |
| RocketMQ | 4.9+ | 异步消息队列（可选） |
| MinIO / OSS | - | 文件存储（可选） |

### 2.2 后端数据源配置

在后端芋道框架的 `application-local.yaml` 中配置数据源：

```yaml
spring:
  datasource:
    druid:
      master:
        url: jdbc:mysql://127.0.0.1:3306/yudao-vue-pro?useUnicode=true&characterEncoding=utf-8&serverTimezone=Asia/Shanghai
        username: root
        password: your_password
        driver-class-name: com.mysql.cj.jdbc.Driver
      # 多数据源配置（可选，用于代码生成器连接其他库）
      slave:
        enabled: false
```

### 2.3 数据库表设计规范

后端芋道框架的数据库表设计遵循以下规范：

**通用字段（每个表必备）**：

| 字段名 | 类型 | 说明 |
|--------|------|------|
| `id` | bigint | 主键，自增或雪花算法 |
| `creator` | varchar(64) | 创建者 |
| `create_time` | datetime | 创建时间 |
| `updater` | varchar(64) | 更新者 |
| `update_time` | datetime | 更新时间 |
| `deleted` | bit(1) | 逻辑删除（0未删除 1已删除） |
| `tenant_id` | bigint | 租户编号（多租户时必备） |

**建表模板**：

```sql
CREATE TABLE `your_module_biz` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `name` varchar(128) NOT NULL COMMENT '名称',
  `status` tinyint NOT NULL DEFAULT 0 COMMENT '状态（0正常 1停用）',
  `sort` int NOT NULL DEFAULT 0 COMMENT '排序',
  `remark` varchar(512) DEFAULT NULL COMMENT '备注',
  -- 以下为通用字段
  `creator` varchar(64) DEFAULT '' COMMENT '创建者',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updater` varchar(64) DEFAULT '' COMMENT '更新者',
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted` bit(1) NOT NULL DEFAULT b'0' COMMENT '是否删除',
  `tenant_id` bigint NOT NULL DEFAULT 0 COMMENT '租户编号',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='业务表';
```

---

## 三、使用代码生成器接入新表（推荐）

代码生成器是最快速的数据库接入方式，可自动生成后端Java代码和前端Vue/TypeScript代码。

### 3.1 代码生成器 API 调用流程

**步骤1：在数据库中创建表**

参照上文建表模板创建你的业务表。

**步骤2：通过代码生成器导入表**

使用 [infra/codegen/index.ts](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/api/infra/codegen/index.ts) 中的 API：

```typescript
// 1. 查询数据库中的表列表
const tables = await getSchemaTableList({
  dataSourceConfigId: 0, // 0 表示主数据源
  name: 'your_module_biz', // 表名过滤
});

// 2. 基于数据库表创建代码生成定义
await createCodegenList({
  dataSourceConfigId: 0,
  tableNames: ['your_module_biz'],
});
```

**步骤3：配置生成信息并更新**

代码生成器的数据结构（[CodegenTable](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/api/infra/codegen/index.ts#L7-L25)）：

```typescript
export interface CodegenTable {
  id: number;
  tableName: string;           // 数据库表名
  tableComment: string;        // 表注释
  moduleName: string;          // 模块名（如: system, pay, member）
  businessName: string;        // 业务名（如: user, order）
  className: string;           // 类名（如: User, Order）
  classComment: string;        // 类注释
  author: string;              // 作者
  templateType: number;        // 模板类型（1单表 2树表 3主子表）
  parentMenuId: number;        // 父菜单ID
  frontType: number;           // 前端类型
  scene: number;               // 场景
}

export interface CodegenColumn {
  id: number;
  columnName: string;          // 数据库列名
  dataType: string;            // 数据库数据类型
  columnComment: string;       // 列注释
  javaType: string;            // Java类型（String, Integer, Long, Date等）
  javaField: string;           // Java字段名
  dictType: string;            // 字典类型
  htmlType: string;            // HTML组件类型
  createOperation: number;     // 是否在新增表单中显示
  updateOperation: number;     // 是否在编辑表单中显示
  listOperation: number;       // 是否在列表中显示
  listOperationCondition: string;  // 列表查询条件类型（eq/like/between等）
  listOperationResult: number; // 是否在列表中显示为查询结果
  primaryKey: number;          // 是否主键
  nullable: number;            // 是否可为空
}
```

配置完成后调用：

```typescript
// 获取代码生成详情（表 + 字段）
const detail = await getCodegenTable(tableId);

// 更新配置
await updateCodegenTable({
  table: { ...detail.table, moduleName: 'yourmodule', businessName: 'biz' },
  columns: detail.columns.map(col => ({ ...col, createOperation: 1 })),
});

// 预览生成代码
const previewResult = await previewCodegen(tableId);

// 下载代码ZIP
await downloadCodegen(tableId);
```

### 3.2 生成的前端代码结构

代码生成器会生成以下前端文件（放入对应目录）：

```
apps/web-antd/src/
├── api/
│   └── yourmodule/
│       └── biz/
│           └── index.ts       # API 接口定义
└── views/
    └── yourmodule/
        └── biz/
            ├── data.ts        # 表单Schema + 表格列定义
            └── index.vue      # 页面主文件
```

---

## 四、手动接入数据库：前端模块开发规范

如果需要手动开发前端模块对接数据库（API已存在），需遵循以下规范。

### 4.1 API 接口层定义

创建文件 `src/api/yourmodule/biz/index.ts`，参照 [pay/app/index.ts](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/api/pay/app/index.ts) 的模式：

```typescript
import type { PageParam, PageResult } from '@vben/request';
import { requestClient } from '#/api/request';

export namespace YourBizApi {
  /** 业务实体接口定义 - 对应数据库表字段 */
  export interface Biz {
    id?: number;
    name: string;
    status: number;
    sort: number;
    remark?: string;
    createTime?: Date;
  }

  /** 更新状态请求 */
  export interface BizUpdateStatusReqVO {
    id: number;
    status: number;
  }
}

/** 分页查询列表 */
export function getBizPage(params: PageParam) {
  return requestClient.get<PageResult<YourBizApi.Biz>>('/yourmodule/biz/page', { params });
}

/** 查询详情 */
export function getBiz(id: number) {
  return requestClient.get<YourBizApi.Biz>(`/yourmodule/biz/get?id=${id}`);
}

/** 新增 */
export function createBiz(data: YourBizApi.Biz) {
  return requestClient.post('/yourmodule/biz/create', data);
}

/** 修改 */
export function updateBiz(data: YourBizApi.Biz) {
  return requestClient.put('/yourmodule/biz/update', data);
}

/** 删除 */
export function deleteBiz(id: number) {
  return requestClient.delete(`/yourmodule/biz/delete?id=${id}`);
}

/** 导出Excel */
export function exportBiz(params: any) {
  return requestClient.download('/yourmodule/biz/export-excel', { params });
}
```

**关键引用说明**：

| 代码片段 | 引用位置 | 作用 |
|---------|---------|------|
| `requestClient` | [api/request.ts](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/api/request.ts) | 统一HTTP客户端，自动携带Token、租户ID、加密解密 |
| `PageParam` / `PageResult` | [effects/request/src](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/packages/effects/request/src) | 分页参数和响应类型 |
| `requestClient.get/post/put/delete` | [request-client.ts](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/packages/effects/request/src/request-client/request-client.ts#L100-L162) | HTTP方法封装 |
| `requestClient.download` | [downloader.ts](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/packages/effects/request/src/request-client/modules/downloader.ts) | 文件下载 |
| `requestClient.upload` | [uploader.ts](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/packages/effects/request/src/request-client/modules/uploader.ts) | 文件上传 |

### 4.2 数据定义层（data.ts）

创建文件 `src/views/yourmodule/biz/data.ts`，参照 [views/system/user/data.ts](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/views/system/user/data.ts)：

```typescript
import type { VbenFormSchema } from '#/adapter/form';
import type { VxeTableGridOptions } from '#/adapter/vxe-table';
import type { YourBizApi } from '#/api/yourmodule/biz';

import { CommonStatusEnum, DICT_TYPE } from '@vben/constants';
import { getDictOptions } from '@vben/hooks';
import { z } from '#/adapter/form';
import { getRangePickerDefaultProps } from '#/utils';

/** 新增/修改表单 Schema - 对应数据库字段的表单组件 */
export function useFormSchema(): VbenFormSchema[] {
  return [
    // 隐藏字段 ID（编辑时使用）
    {
      component: 'Input',
      fieldName: 'id',
      dependencies: { triggerFields: [''], show: () => false },
    },
    // 名称字段 - 对应 name VARCHAR 列
    {
      fieldName: 'name',
      label: '名称',
      component: 'Input',
      componentProps: { placeholder: '请输入名称' },
      rules: 'required',
    },
    // 状态字段 - 对应 status TINYINT 列，使用单选按钮组
    {
      fieldName: 'status',
      label: '状态',
      component: 'RadioGroup',
      componentProps: {
        options: getDictOptions(DICT_TYPE.COMMON_STATUS, 'number'),
        buttonStyle: 'solid',
        optionType: 'button',
      },
      rules: z.number().default(CommonStatusEnum.ENABLE),
    },
    // 排序字段 - 对应 sort INT 列，使用数字输入
    {
      fieldName: 'sort',
      label: '排序',
      component: 'InputNumber',
      componentProps: { placeholder: '请输入排序' },
      defaultValue: 0,
    },
    // 备注字段 - 对应 remark VARCHAR/TEXT 列
    {
      fieldName: 'remark',
      label: '备注',
      component: 'Textarea',
      componentProps: { placeholder: '请输入备注', rows: 3 },
    },
  ];
}

/** 搜索表单 Schema - 对应列表查询条件字段 */
export function useGridFormSchema(): VbenFormSchema[] {
  return [
    {
      fieldName: 'name',
      label: '名称',
      component: 'Input',
      componentProps: { placeholder: '请输入名称', allowClear: true },
    },
    {
      fieldName: 'status',
      label: '状态',
      component: 'Select',
      componentProps: {
        placeholder: '请选择状态',
        options: getDictOptions(DICT_TYPE.COMMON_STATUS, 'number'),
        allowClear: true,
      },
    },
    {
      fieldName: 'createTime',
      label: '创建时间',
      component: 'RangePicker',
      componentProps: { ...getRangePickerDefaultProps(), allowClear: true },
    },
  ];
}

/** 表格列定义 - 对应数据库字段在列表中的展示 */
export function useGridColumns(
  onStatusChange?: (newStatus: number, row: YourBizApi.Biz) => PromiseLike<boolean | undefined>,
): VxeTableGridOptions['columns'] {
  return [
    { type: 'checkbox', width: 40 },
    { field: 'id', title: '编号', minWidth: 80 },
    { field: 'name', title: '名称', minWidth: 120 },
    {
      field: 'status',
      title: '状态',
      minWidth: 100,
      align: 'center',
      cellRender: {
        attrs: { beforeChange: onStatusChange },
        name: 'CellSwitch',
        props: {
          checkedValue: CommonStatusEnum.ENABLE,
          unCheckedValue: CommonStatusEnum.DISABLE,
        },
      },
    },
    { field: 'sort', title: '排序', minWidth: 80 },
    { field: 'remark', title: '备注', minWidth: 120 },
    {
      field: 'createTime',
      title: '创建时间',
      minWidth: 180,
      formatter: 'formatDateTime',
    },
    {
      title: '操作',
      width: 180,
      fixed: 'right',
      slots: { default: 'actions' },
    },
  ];
}
```

**关键引用说明**：

| 代码片段 | 引用位置 | 作用 |
|---------|---------|------|
| `VbenFormSchema` | [adapter/form.ts](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/adapter/form.ts#L69) | 表单Schema类型定义 |
| `VxeTableGridOptions` | [adapter/vxe-table.ts](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/adapter/vxe-table.ts#L1) | Vxe表格配置类型 |
| `getDictOptions` | @vben/hooks | 获取字典数据选项 |
| `DICT_TYPE` | [constants/src/dict-enum.ts](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/packages/constants/src/dict-enum.ts) | 字典类型常量 |
| `CommonStatusEnum` | @vben/constants | 通用状态枚举 |
| `CellSwitch` | [adapter/vxe-table.ts#L169](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/adapter/vxe-table.ts#L169) | 开关渲染器 |
| `CellDict` | [adapter/vxe-table.ts#L152](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/adapter/vxe-table.ts#L152) | 字典标签渲染器 |
| `z` | [adapter/form.ts](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/adapter/form.ts#L66) | Zod 验证库 |

### 4.3 视图页面（index.vue）

创建文件 `src/views/yourmodule/biz/index.vue`：

```vue
<script setup lang="ts">
import type { YourBizApi } from '#/api/yourmodule/biz';
import type { VbenFormSchema } from '#/adapter/form';
import type { VbenVxeGrid } from '#/adapter/vxe-table';

import { Page, useVbenModal } from '@vben/common-ui';
import { useMessage } from '@vben/hooks';

import {
  createBiz,
  deleteBiz,
  exportBiz,
  getBiz,
  getBizPage,
  updateBiz,
  updateBizStatus,
} from '#/api/yourmodule/biz';
import { useVbenVxeGrid } from '#/adapter/vxe-table';
import { useFormSchema, useGridColumns, useGridFormSchema } from './data';

const { createConfirm, createMessage } = useMessage();

// 弹窗实例
const [Modal, modalApi] = useVbenModal({
  connectedComponent: 'VbenModal',
  onConfirm: async () => {
    const { values } = await modalApi.getData();
    if (values.id) {
      await updateBiz(values);
      createMessage.success('修改成功');
    } else {
      await createBiz(values);
      createMessage.success('新增成功');
    }
    gridApi.query();
    return true;
  },
});

// 表格配置
const [Grid, gridApi] = useVbenVxeGrid({
  formConfig: { schema: useGridFormSchema() as VbenFormSchema[] },
  gridOptions: {
    columns: useGridColumns(async (status, row) => {
      await createConfirm(`确认要${status === 1 ? '启用' : '停用'}吗?`);
      await updateBizStatus({ id: row.id, status });
      return true;
    }),
    proxyConfig: {
      ajax: {
        query: async ({ page }, formValues) => {
          return await getBizPage({
            pageNo: page.currentPage,
            pageSize: page.pageSize,
            ...formValues,
          });
        },
      },
    },
    toolbarConfig: {
      buttons: [
        {
          code: 'add',
          name: '新增',
          status: 'primary',
          click: () => {
            modalApi.setData({});
            modalApi.open();
          },
        },
      ],
    },
  },
});

// 编辑
const onEdit = async (row: YourBizApi.Biz) => {
  const data = await getBiz(row.id!);
  modalApi.setData(data);
  modalApi.open();
};

// 删除
const onDelete = async (row: YourBizApi.Biz) => {
  await createConfirm(`确认删除【${row.name}】吗?`);
  await deleteBiz(row.id!);
  createMessage.success('删除成功');
  gridApi.query();
};

// 导出
const onExport = () => {
  gridApi.exportData({
    filename: '业务数据',
    exportMethod: async (params) => {
      await exportBiz(params);
    },
  });
};
</script>

<template>
  <Page auto-content-height>
    <Modal :title="`业务${modalApi.getData()?.id ? '编辑' : '新增'}`">
      <template #default="{ values }">
        <VbenForm :schema="useFormSchema()" :values="values" />
      </template>
    </Modal>
    <Grid>
      <template #actions="{ row }">
        <TableAction :actions="[
          { label: '编辑', onClick: () => onEdit(row) },
          { label: '删除', color: 'error', onClick: () => onDelete(row) },
        ]" />
      </template>
    </Grid>
  </Page>
</template>
```

### 4.4 数据库字段到前端组件映射

以下是数据库字段类型与前端组件的映射参考：

| 数据库类型 | Java类型 | TypeScript类型 | HTML组件 | 应用示例 |
|-----------|---------|----------------|---------|---------|
| `bigint` | `Long` | `number` | `Input` / `InputNumber` | 主键ID、用户ID [system/user](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/views/system/user/data.ts#L301-L305) |
| `varchar` | `String` | `string` | `Input` / `Textarea` | 名称、标题、备注 [pay/app/data.ts#L193-L200](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/views/pay/app/data.ts#L193-L200) |
| `int` / `tinyint` | `Integer` | `number` | `RadioGroup` / `Select` / `Switch` | 状态、类型 [pay/app/data.ts#L211-L219](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/views/pay/app/data.ts#L211-L219) |
| `decimal` | `BigDecimal` | `number` | `InputNumber` | 金额、费率 [pay/app/data.ts#L289-L299](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/views/pay/app/data.ts#L289-L299) |
| `datetime` | `Date` | `Date` / `number` | `DatePicker` / `RangePicker` | 创建时间、有效期 [system/user/data.ts#L281-L289](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/views/system/user/data.ts#L281-L289) |
| `text` / `longtext` | `String` | `string` | `Textarea` / `Editor` | 富文本内容、备注 |
| `bit(1)` / `boolean` | `Boolean` | `boolean` | `Switch` / `Checkbox` | 是否删除、是否启用 |
| `json` | `String` | `any` | `Input` / 自定义 | 配置JSON、扩展字段 |
| 关联外键 | `Long` | `number` | `ApiSelect` / `ApiTreeSelect` | 部门ID、角色ID [system/user/data.ts#L57-L82](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/views/system/user/data.ts#L57-L82) |
| 文件路径 | `String` | `string` | `Upload` | 头像、证书 [pay/app/data.ts#L409-L417](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/views/pay/app/data.ts#L409-L417) |
| 字典字段 | `String` | `string` | `Select` / `RadioGroup` + `getDictOptions` | 数据字典 [system/user/data.ts#L102-L110](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/views/system/user/data.ts#L102-L110) |

### 4.5 特殊字段处理代码片段

**关联外键字段（ApiSelect）**：

```typescript
// 对应数据库字段：dept_id (bigint) - 关联 system_dept 表
{
  fieldName: 'deptId',
  label: '归属部门',
  component: 'ApiTreeSelect',
  componentProps: {
    api: async () => {
      const data = await getDeptList();
      return handleTree(data); // 转换为树形结构
    },
    labelField: 'name',    // 显示字段
    valueField: 'id',      // 值字段
    childrenField: 'children',
    placeholder: '请选择归属部门',
    treeDefaultExpandAll: true,
  },
},
```

**字典字段（getDictOptions）**：

```typescript
// 对应数据库字段：sex (tinyint) - 使用 system_user_sex 字典
{
  fieldName: 'sex',
  label: '用户性别',
  component: 'RadioGroup',
  componentProps: {
    options: getDictOptions(DICT_TYPE.SYSTEM_USER_SEX, 'number'),
    // 第二个参数 'number' 表示值类型为数字（后端存储为int/tinyint）
    // 如果后端存储为字符串，使用 'string'
    buttonStyle: 'solid',
    optionType: 'button',
  },
  rules: z.number().default(1),
},
```

表格中展示字典标签使用 `CellDict` 渲染器：

```typescript
{
  field: 'sex',
  title: '性别',
  minWidth: 80,
  cellRender: {
    name: 'CellDict',
    props: { type: DICT_TYPE.SYSTEM_USER_SEX },
  },
},
```

**状态开关字段（CellSwitch）**：

```typescript
// 对应数据库字段：status (tinyint) - 0启用/1停用
{
  field: 'status',
  title: '状态',
  cellRender: {
    attrs: {
      beforeChange: async (newStatus, row) => {
        // 状态变更前的确认和API调用
        await updateBizStatus({ id: row.id, status: newStatus });
        return true; // 返回true表示允许变更，false阻止
      },
    },
    name: 'CellSwitch',
    props: {
      checkedValue: CommonStatusEnum.ENABLE,   // 0
      unCheckedValue: CommonStatusEnum.DISABLE, // 1
      checkedChildren: '启用',
      unCheckedChildren: '停用',
    },
  },
},
```

**金额字段（分转元）**：

数据库金额存储单位为**分**（int/bigint），前端显示需转为**元**：

```typescript
// 表格中使用自定义格式化
{
  field: 'amount',
  title: '金额',
  minWidth: 120,
  formatter: 'formatFenToYuanAmount', // 分转元，保留2位小数
},
// 格式化器定义在 adapter/vxe-table.ts 中
// vxeUI.formats.add('formatFenToYuanAmount', ...)
// 引用: adapter/vxe-table.ts#L352
```

**日期时间范围查询**：

```typescript
// 对应数据库字段：create_time (datetime)
{
  fieldName: 'createTime',
  label: '创建时间',
  component: 'RangePicker',
  componentProps: {
    ...getRangePickerDefaultProps(), // 默认属性配置
    allowClear: true,
  },
},
// 后端会自动解析为 beginCreateTime 和 endCreateTime 两个参数
// 引用: utils/rangePickerProps.ts
```

**文件上传字段**：

```typescript
// 对应数据库字段：cert_content (text) - 存储证书文件内容
{
  label: '证书文件',
  fieldName: 'config.privateKeyContent',
  component: h(InputUpload, {
    inputType: 'textarea',
    textareaProps: { rows: 3, placeholder: '请上传证书文件' },
    fileUploadProps: {
      accept: ['pem', 'p12', 'crt'], // 允许的文件类型
    },
  }),
  rules: 'required',
},
// 引用: components/upload/index.ts
```

---

## 五、前端环境变量配置

修改 [.env.development](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/.env.development) 配置后端API地址：

```bash
# 后端服务地址
VITE_BASE_URL=http://127.0.0.1:48080
# API前缀（芋道后端默认 /admin-api）
VITE_GLOB_API_URL=/admin-api
# 文件上传类型：server - 后端上传，client - 前端直连S3
VITE_UPLOAD_TYPE=server
# 租户开关（如果后端启用了多租户）
VITE_APP_TENANT_ENABLE=true
# API加解密开关（需要与后端密钥一致）
VITE_APP_API_ENCRYPT_ENABLE=true
VITE_APP_API_ENCRYPT_ALGORITHM=AES
VITE_APP_API_ENCRYPT_REQUEST_KEY=your_request_key
VITE_APP_API_ENCRYPT_RESPONSE_KEY=your_response_key
```

Vite代理配置在 [vite.config.ts](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/vite.config.ts) 中：

```typescript
proxy: {
  '/admin-api': {
    changeOrigin: true,
    rewrite: (path) => path.replace(/^\/admin-api/, ''),
    target: 'http://localhost:48080/admin-api',
    ws: true,
  },
},
```

---

## 六、新增字典数据

如果你的数据库字段使用了字典枚举（如状态、类型），需要在后端 `system_dict_type` 和 `system_dict_data` 表中配置字典，然后在前端 [dict-enum.ts](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/packages/constants/src/dict-enum.ts) 中添加常量：

```typescript
// 在 dict-enum.ts 中添加你的字典类型常量
const YOUR_MODULE_DICT = {
  YOUR_BIZ_STATUS: 'your_biz_status',   // 对应后端字典类型
  YOUR_BIZ_TYPE: 'your_biz_type',
} as const;
```

使用方式：

```typescript
import { DICT_TYPE } from '@vben/constants';
import { getDictOptions } from '@vben/hooks';

// 在表单中使用
const options = getDictOptions(DICT_TYPE.YOUR_BIZ_STATUS, 'number');

// 在表格中使用
{
  field: 'status',
  title: '状态',
  cellRender: { name: 'CellDict', props: { type: DICT_TYPE.YOUR_BIZ_STATUS } },
}
```

---

## 七、请求拦截器与数据处理

[request.ts](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/api/request.ts) 中已内置以下处理，无需手动处理：

| 功能 | 代码位置 | 说明 |
|------|---------|------|
| Token 自动携带 | [request.ts#L75-L106](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/api/request.ts#L75-L106) | `Authorization: Bearer {token}` |
| 租户ID传递 | [request.ts#L82-L88](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/api/request.ts#L82-L88) | `tenant-id` 请求头 |
| Token 自动刷新 | [request.ts#L54-L68](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/api/request.ts#L54-L68) | 401时自动刷新Token |
| API 请求加密 | [request.ts#L91-L103](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/api/request.ts#L91-L103) | AES加密请求体 |
| API 响应解密 | [request.ts#L109-L129](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/api/request.ts#L109-L129) | AES解密响应体 |
| 统一响应解析 | [request.ts#L175-L181](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/api/request.ts#L175-L181) | 自动提取 `{code:0, data:{}, msg:""}` |
| Blob错误处理 | [request.ts#L136-L172](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/api/request.ts#L136-L172) | 下载文件时的401处理 |
| 国际化 | [request.ts#L80](file:///Users/tog/Desktop/code/gsb/gsb-723/xyj-723-4/xyj-723-4_Charmander/apps/web-antd/src/api/request.ts#L80) | `Accept-Language` 请求头 |

---

## 八、完整接入 Checklist

接入新数据库表的完整步骤：

- [ ] 1. 在 MySQL 中创建业务表（遵循建表规范）
- [ ] 2. 后端配置数据源连接（如非默认库）
- [ ] 3. 使用代码生成器导入表结构并生成代码
- [ ] 4. 将生成的后端代码放入对应模块并启动后端
- [ ] 5. 将生成的前端代码放入 `apps/web-antd/src/api/` 和 `apps/web-antd/src/views/`
- [ ] 6. 配置菜单权限（后端菜单管理中添加菜单）
- [ ] 7. 如有字典字段，在字典管理中配置字典数据
- [ ] 8. 启动前端 `pnpm dev:antd`，测试 CRUD 功能
- [ ] 9. 验证分页查询、条件搜索、导出功能
- [ ] 10. 验证新增/编辑表单验证、状态切换等交互
