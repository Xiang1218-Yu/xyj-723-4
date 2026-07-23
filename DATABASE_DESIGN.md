# 数据库设计文档 —— 接入自有数据库指南

> 本文档说明芋道中台框架的数据库设计规范、核心表结构、字段约定，以及如何将自有数据库接入本系统。包含建表 SQL 片段、Java/TS 类型映射代码、前端字段使用引用。

---

## 目录

- [1. 数据库设计规范](#1-数据库设计规范)
- [2. 公共字段约定](#2-公共字段约定)
- [3. 核心模块表结构](#3-核心模块表结构)
- [4. 字典与枚举映射](#4-字典与枚举映射)
- [5. 接入自有数据库步骤](#5-接入自有数据库步骤)
- [6. 代码生成器使用](#6-代码生成器使用)
- [7. 前后端字段映射速查表](#7-前后端字段映射速查表)

---

## 1. 数据库设计规范

### 1.1 命名约定

| 对象 | 命名规则 | 示例 |
|------|----------|------|
| 表名 | `系统模块_业务实体`，全小写下划线 | `system_users`、`pay_order`、`member_user` |
| 字段名 | 小写下划线命名 | `user_name`、`create_time`、`dept_id` |
| 主键 | `id` BIGINT 自增/雪花 | `id` |
| 外键关联 | `业务_id` | `dept_id`、`role_id`、`tenant_id` |
| 布尔字段 | `tinyint` + 注释说明 | `status`(0开启/1禁用)、`visible`(0显示/1隐藏) |
| 时间字段 | `datetime` | `create_time`、`update_time`、`login_date` |
| 金额字段 | `int` 单位**分**（避免浮点精度问题） | `price`、`refund_price`、`pay_amount` |
| 索引 | `idx_字段名`（普通索引）、`uk_字段名`（唯一索引） | `idx_username`、`uk_app_key` |

### 1.2 多租户设计

所有业务表均包含 `tenant_id` 字段（BIGINT），用于 SaaS 多租户数据隔离。系统在请求头中通过 `tenant-id` 传递当前租户编号，后端 MyBatis 拦截器自动拼接租户过滤条件。

相关前端代码参考：[request.ts](apps/web-antd/src/api/request.ts#L82-L84) 中租户 ID 的注入逻辑：

```typescript
// 添加租户编号
config.headers['tenant-id'] = tenantEnable
  ? accessStore.tenantId
  : undefined;
// 只有登录时，才设置 visit-tenant-id 访问租户
config.headers['visit-tenant-id'] = tenantEnable
  ? accessStore.visitTenantId
  : undefined;
```

### 1.3 逻辑删除

使用 `deleted` 字段（BIT(1)/TINYINT）实现逻辑删除，默认值 `0`（未删除），删除后置 `1`。后端框架自动过滤已删除记录。

### 1.4 自动填充字段

| 字段 | 类型 | 说明 | 自动填充时机 |
|------|------|------|-------------|
| `creator` | VARCHAR(64) | 创建者 | INSERT 时自动填充当前用户 ID |
| `create_time` | DATETIME | 创建时间 | INSERT 时自动填充当前时间 |
| `updater` | VARCHAR(64) | 更新者 | INSERT/UPDATE 时自动填充 |
| `update_time` | DATETIME | 更新时间 | INSERT/UPDATE 时自动填充 |
| `deleted` | BIT(1) | 逻辑删除 | 默认 b'0' |

---

## 2. 公共字段约定

每张业务表都应包含以下公共字段：

```sql
CREATE TABLE `your_table_name` (
  `id`          BIGINT       NOT NULL AUTO_INCREMENT COMMENT '主键编号',
  -- ===== 业务字段 =====

  -- ===== 公共字段 =====
  `creator`     VARCHAR(64)  NOT NULL DEFAULT '' COMMENT '创建者',
  `create_time` DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updater`     VARCHAR(64)  NOT NULL DEFAULT '' COMMENT '更新者',
  `update_time` DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted`     BIT(1)       NOT NULL DEFAULT b'0' COMMENT '是否删除',
  `tenant_id`   BIGINT       NOT NULL DEFAULT 0 COMMENT '租户编号',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='你的业务表注释';
```

### 2.1 状态字段统一约定

系统中 `status` 字段的通用枚举值定义在 [biz-system-enum.ts](packages/constants/src/biz-system-enum.ts#L3-L6)：

```typescript
// 全局通用状态枚举
export const CommonStatusEnum = {
  ENABLE: 0,   // 开启
  DISABLE: 1,  // 禁用
};
```

在前端表单中，状态字段的使用方式参考 [data.ts](apps/web-antd/src/views/system/user/data.ts#L112-L121)：

```typescript
{
  fieldName: 'status',
  label: '用户状态',
  component: 'RadioGroup',
  componentProps: {
    options: getDictOptions(DICT_TYPE.COMMON_STATUS, 'number'),
    buttonStyle: 'solid',
    optionType: 'button',
  },
  rules: z.number().default(CommonStatusEnum.ENABLE),
}
```

在表格列中渲染状态参考 [data.ts](apps/web-antd/src/views/pay/app/data.ts#L67-L79)：

```typescript
{
  field: 'status',
  title: '状态',
  cellRender: {
    name: 'CellSwitch',
    props: {
      checkedValue: CommonStatusEnum.ENABLE,    // 0 = 开启
      unCheckedValue: CommonStatusEnum.DISABLE, // 1 = 禁用
    },
  },
}
```

### 2.2 分页查询标准接口

所有列表查询统一使用分页接口，前端 PageParam/PageResult 类型如下。参考 [order/index.ts](apps/web-antd/src/api/pay/order/index.ts#L44-L48)：

```typescript
import type { PageParam, PageResult } from '@vben/request';

/** 查询支付订单列表 */
export function getOrderPage(params: PageParam) {
  return requestClient.get<PageResult<PayOrderApi.Order>>('/pay/order/page', {
    params,
  });
}
```

对应后端 Controller 标准签名：

```java
@GetMapping("/page")
public CommonResult<PageResult<OrderRespVO>> getOrderPage(@Valid PageParam pageParam) {
    // 返回 PageResult<OrderRespVO>
}
```

### 2.3 标准 CRUD 接口约定

| 操作 | HTTP 方法 | URL 模式 | 说明 |
|------|-----------|----------|------|
| 分页查询 | GET | `/{module}/page` | 支持条件筛选 |
| 详情查询 | GET | `/{module}/get?id=` | 按主键查询 |
| 列表查询 | GET | `/{module}/list` | 不分页列表 |
| 精简列表 | GET | `/{module}/simple-list` | 只返回 id + name |
| 新增 | POST | `/{module}/create` | Body 传实体 |
| 修改 | PUT | `/{module}/update` | Body 传实体 |
| 删除 | DELETE | `/{module}/delete?id=` | 逻辑删除 |
| 批量删除 | DELETE | `/{module}/delete-list?ids=1,2,3` | 批量逻辑删除 |
| 导出 | GET | `/{module}/export-excel` | Blob 下载 |

参考实现：[user/index.ts](apps/web-antd/src/api/system/user/index.ts) 是最完整的 CRUD 示例。

---

## 3. 核心模块表结构

### 3.1 系统用户表 `system_users`

对应前端类型定义：[user/index.ts](apps/web-antd/src/api/system/user/index.ts#L7-L23)

```sql
CREATE TABLE `system_users` (
  `id`           BIGINT       NOT NULL AUTO_INCREMENT COMMENT '用户编号',
  `username`     VARCHAR(30)  NOT NULL COMMENT '用户账号',
  `password`     VARCHAR(100) NOT NULL DEFAULT '' COMMENT '密码',
  `nickname`     VARCHAR(30)  NOT NULL COMMENT '用户昵称',
  `remark`       VARCHAR(500)  DEFAULT NULL COMMENT '备注',
  `dept_id`      BIGINT        DEFAULT NULL COMMENT '部门编号',
  `post_ids`     VARCHAR(255)  DEFAULT NULL COMMENT '岗位编号数组（逗号分隔）',
  `email`        VARCHAR(50)   DEFAULT '' COMMENT '用户邮箱',
  `mobile`       VARCHAR(20)   DEFAULT '' COMMENT '手机号码',
  `sex`          TINYINT      NOT NULL DEFAULT 0 COMMENT '用户性别（0未知 1男 2女）',
  `avatar`       VARCHAR(512)  DEFAULT '' COMMENT '头像地址',
  `login_ip`     VARCHAR(50)   DEFAULT '' COMMENT '最后登录IP',
  `login_date`   DATETIME      DEFAULT NULL COMMENT '最后登录时间',
  `status`       TINYINT      NOT NULL DEFAULT 0 COMMENT '状态（0开启 1禁用）',
  -- 公共字段
  `creator`      VARCHAR(64)  NOT NULL DEFAULT '' COMMENT '创建者',
  `create_time`  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updater`      VARCHAR(64)  NOT NULL DEFAULT '' COMMENT '更新者',
  `update_time`  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted`      BIT(1)       NOT NULL DEFAULT b'0' COMMENT '是否删除',
  `tenant_id`    BIGINT       NOT NULL DEFAULT 0 COMMENT '租户编号',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_username` (`username`, `tenant_id`, `deleted`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户信息表';
```

**字段在前端的使用片段** — 引用自 [user/index.ts](apps/web-antd/src/api/system/user/index.ts#L7-L23)：

```typescript
export namespace SystemUserApi {
  /** 用户信息 */
  export interface User {
    id?: number;
    username: string;      // → 对应 system_users.username
    nickname: string;      // → 对应 system_users.nickname
    deptId: number;        // → 对应 system_users.dept_id（驼峰转换）
    deptName?: string;     // → 关联查询字段（JOIN system_dept.name）
    postIds: string[];     // → 对应 system_users.post_ids（逗号分隔→数组）
    email: string;
    mobile: string;
    sex: number;           // → 对应 system_users.sex
    avatar: string;
    loginIp: string;       // → 对应 system_users.login_ip
    loginDate?: Date;      // → 对应 system_users.login_date
    status: number;        // → 对应 system_users.status
    remark: string;
    createTime?: Date;     // → 对应 system_users.create_time
  }
}
```

**搜索表单字段映射** — 引用自 [data.ts](apps/web-antd/src/views/system/user/data.ts)（列表搜索→查询条件）：

```typescript
// 搜索字段 → SQL WHERE 条件
// username  → WHERE username LIKE CONCAT('%', #{username}, '%')
// mobile    → WHERE mobile LIKE CONCAT('%', #{mobile}, '%')
// status    → WHERE status = #{status}
// deptId    → WHERE dept_id = #{deptId}
// createTime→ WHERE create_time BETWEEN #{beginTime} AND #{endTime}
```

### 3.2 支付订单表 `pay_order`

对应前端类型定义：[order/index.ts](apps/web-antd/src/api/pay/order/index.ts#L7-L40)

```sql
CREATE TABLE `pay_order` (
  `id`                    BIGINT       NOT NULL AUTO_INCREMENT COMMENT '订单编号',
  `no`                    VARCHAR(64)  NOT NULL COMMENT '支付单号',
  `merchant_order_id`     VARCHAR(64)  NOT NULL COMMENT '商户单号',
  `app_id`                BIGINT       NOT NULL COMMENT '应用编号',
  `channel_id`            BIGINT        DEFAULT NULL COMMENT '渠道编号',
  `channel_code`          VARCHAR(32)   DEFAULT NULL COMMENT '渠道编码',
  `merchant_id`           BIGINT        DEFAULT NULL COMMENT '商户编号',
  `subject`               VARCHAR(255) NOT NULL COMMENT '商品标题',
  `body`                  VARCHAR(500)  DEFAULT NULL COMMENT '商品描述',
  `notify_url`            VARCHAR(512) NOT NULL COMMENT '异步通知地址',
  `amount`                INT          NOT NULL COMMENT '支付金额，单位：分',
  `price`                 INT          NOT NULL COMMENT '支付金额，单位：分',
  `channel_fee_rate`      DECIMAL(10,6) DEFAULT 0 COMMENT '渠道手续费率',
  `channel_fee_amount`    INT           DEFAULT 0 COMMENT '渠道手续费，单位：分',
  `channel_fee_price`     INT           DEFAULT 0 COMMENT '手续金额，单位：分',
  `refund_price`          INT           DEFAULT 0 COMMENT '退款金额，单位：分',
  `status`                TINYINT      NOT NULL DEFAULT 0 COMMENT '支付状态（0未支付 10已支付 20已关闭）',
  `user_ip`               VARCHAR(50)   DEFAULT NULL COMMENT '用户IP',
  `expire_time`           DATETIME      DEFAULT NULL COMMENT '订单失效时间',
  `success_time`          DATETIME      DEFAULT NULL COMMENT '订单成功时间',
  `notify_time`           DATETIME      DEFAULT NULL COMMENT '订单通知时间',
  `notify_status`         TINYINT       DEFAULT 0 COMMENT '通知状态（0未通知 1通知成功 2通知失败）',
  `refund_status`         TINYINT       DEFAULT 0 COMMENT '退款状态',
  `refund_times`          INT           DEFAULT 0 COMMENT '退款次数',
  `channel_user_id`       VARCHAR(64)   DEFAULT NULL COMMENT '渠道用户标识',
  `channel_order_no`      VARCHAR(64)   DEFAULT NULL COMMENT '渠道订单号',
  `channel_notify_data`   TEXT          DEFAULT NULL COMMENT '异步回调原始报文',
  -- 公共字段
  `creator`               VARCHAR(64)  NOT NULL DEFAULT '',
  `create_time`           DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updater`               VARCHAR(64)  NOT NULL DEFAULT '',
  `update_time`           DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deleted`               BIT(1)       NOT NULL DEFAULT b'0',
  `tenant_id`             BIGINT       NOT NULL DEFAULT 0,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_no` (`no`),
  UNIQUE KEY `uk_merchant_order_id` (`merchant_id`, `merchant_order_id`, `deleted`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='支付订单';
```

**金额字段的前端处理** — 引用自 [data.ts](apps/web-antd/src/views/pay/order/data.ts#L96-L106)：

```typescript
// 金额字段以分为单位存储，前端展示时通过 formatter 转换为元
{
  field: 'price',
  title: '支付金额',
  minWidth: 120,
  formatter: 'formatAmount2',  // VxeTable 内置格式化：分 → 元（除以100，保留2位小数）
},
```

**状态枚举值定义** — 引用自 [biz-pay-enum.ts](packages/constants/src/biz-pay-enum.ts#L92-L105)：

```typescript
export const PayOrderStatusEnum = {
  WAITING: { status: 0,  name: '未支付' },
  SUCCESS: { status: 10, name: '已支付' },
  CLOSED:  { status: 20, name: '已关闭' },
};
```

**状态在详情页中的渲染** — 引用自 [data.ts](apps/web-antd/src/views/pay/order/data.ts#L186-L191)：

```typescript
{
  field: 'status',
  label: '支付状态',
  render: (val) =>
    h(DictTag, {
      type: DICT_TYPE.PAY_ORDER_STATUS,  // → 对应字典 pay_order_status
      value: val,                         // → 0/10/20
    }),
},
```

### 3.3 支付应用表 `pay_app`

对应前端类型定义：[app/index.ts](apps/web-antd/src/api/pay/app/index.ts#L7-L20)

```sql
CREATE TABLE `pay_app` (
  `id`                  BIGINT       NOT NULL AUTO_INCREMENT COMMENT '应用编号',
  `app_key`             VARCHAR(64)  NOT NULL COMMENT '应用标识（AppKey）',
  `name`                VARCHAR(64)  NOT NULL COMMENT '应用名称',
  `merchant_id`         BIGINT       NOT NULL COMMENT '商户编号',
  `status`              TINYINT      NOT NULL DEFAULT 0 COMMENT '状态（0开启 1禁用）',
  `remark`              VARCHAR(255)  DEFAULT NULL COMMENT '备注',
  `pay_notify_url`      VARCHAR(512) NOT NULL COMMENT '支付结果回调地址',
  `refund_notify_url`   VARCHAR(512) NOT NULL DEFAULT '' COMMENT '退款结果回调地址',
  `transfer_notify_url` VARCHAR(512) NOT NULL DEFAULT '' COMMENT '转账结果回调地址',
  `channel_codes`       VARCHAR(512)  DEFAULT NULL COMMENT '关联渠道编码（逗号分隔）',
  -- 公共字段
  `creator`             VARCHAR(64)  NOT NULL DEFAULT '',
  `create_time`         DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updater`             VARCHAR(64)  NOT NULL DEFAULT '',
  `update_time`         DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deleted`             BIT(1)       NOT NULL DEFAULT b'0',
  `tenant_id`           BIGINT       NOT NULL DEFAULT 0,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_app_key` (`app_key`, `deleted`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='支付应用';
```

**CRUD 接口实现** — 引用自 [app/index.ts](apps/web-antd/src/api/pay/app/index.ts#L29-L64)：

```typescript
/** 查询支付应用列表 */
export function getAppPage(params: PageParam) {
  return requestClient.get<PageResult<PayAppApi.App>>('/pay/app/page', { params });
}
/** 查询支付应用详情 */
export function getApp(id: number) {
  return requestClient.get<PayAppApi.App>(`/pay/app/get?id=${id}`);
}
/** 新增支付应用 */
export function createApp(data: PayAppApi.App) {
  return requestClient.post('/pay/app/create', data);
}
/** 修改支付应用 */
export function updateApp(data: PayAppApi.App) {
  return requestClient.put('/pay/app/update', data);
}
/** 删除支付应用 */
export function deleteApp(id: number) {
  return requestClient.delete(`/pay/app/delete?id=${id}`);
}
```

### 3.4 会员用户表 `member_user`

对应前端类型定义：[user/index.ts](apps/web-antd/src/api/member/user/index.ts#L7-L31)

```sql
CREATE TABLE `member_user` (
  `id`              BIGINT       NOT NULL AUTO_INCREMENT COMMENT '会员编号',
  `mobile`          VARCHAR(20)   DEFAULT '' COMMENT '手机号',
  `email`           VARCHAR(50)   DEFAULT '' COMMENT '邮箱',
  `password`        VARCHAR(100)  DEFAULT '' COMMENT '密码（加密存储）',
  `nickname`        VARCHAR(30)  NOT NULL COMMENT '昵称',
  `name`            VARCHAR(30)   DEFAULT '' COMMENT '真实名字',
  `avatar`          VARCHAR(512)  DEFAULT '' COMMENT '头像',
  `sex`             TINYINT      NOT NULL DEFAULT 0 COMMENT '性别（0未知 1男 2女）',
  `birthday`        DATETIME      DEFAULT NULL COMMENT '出生日期',
  `area_id`         BIGINT        DEFAULT NULL COMMENT '所在地编号',
  `status`          TINYINT      NOT NULL DEFAULT 0 COMMENT '状态（0开启 1禁用）',
  `mark`            VARCHAR(255)  DEFAULT NULL COMMENT '用户备注（管理员标记）',
  `tag_ids`         VARCHAR(255)  DEFAULT NULL COMMENT '标签编号（逗号分隔）',
  `group_id`        BIGINT        DEFAULT NULL COMMENT '分组编号',
  `level_id`        BIGINT        DEFAULT NULL COMMENT '等级编号',
  `point`           INT           DEFAULT 0 COMMENT '当前积分',
  `total_point`     INT           DEFAULT 0 COMMENT '累计积分',
  `experience`      INT           DEFAULT 0 COMMENT '当前经验值',
  `register_ip`     VARCHAR(50)   DEFAULT '' COMMENT '注册IP',
  `login_ip`        VARCHAR(50)   DEFAULT '' COMMENT '最后登录IP',
  `login_date`      DATETIME      DEFAULT NULL COMMENT '最后登录时间',
  -- 公共字段
  `creator`         VARCHAR(64)  NOT NULL DEFAULT '',
  `create_time`     DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updater`         VARCHAR(64)  NOT NULL DEFAULT '',
  `update_time`     DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deleted`         BIT(1)       NOT NULL DEFAULT b'0',
  `tenant_id`       BIGINT       NOT NULL DEFAULT 0,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_mobile` (`mobile`, `tenant_id`, `deleted`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='会员用户';
```

**字段在前端表单中的使用** — 引用自 [data.ts](apps/web-antd/src/views/member/user/data.ts#L30-L113)：

```typescript
// 手机号字段 → 必填校验
{
  fieldName: 'mobile',
  label: '手机号',
  component: 'Input',
  rules: 'required',    // 必填规则
},
// 邮箱字段 → Zod schema 校验
{
  fieldName: 'email',
  label: '邮箱',
  component: 'Input',
  rules: z.string().email('邮箱格式不正确').or(z.literal('')).optional(),
},
// 头像字段 → 图片上传组件
{
  fieldName: 'avatar',
  label: '头像',
  component: 'ImageUpload',
},
// 标签字段 → API 远程下拉（多选）
{
  fieldName: 'tagIds',
  label: '用户标签',
  component: 'ApiSelect',
  componentProps: {
    api: getSimpleTagList,    // 调用 /member/tag/list-all-simple
    labelField: 'name',
    valueField: 'id',
    mode: 'multiple',         // 对应 member_user.tag_ids（逗号分隔存储）
  },
},
// 等级字段 → API 远程下拉
{
  fieldName: 'levelId',
  label: '会员等级',
  component: 'ApiSelect',
  componentProps: {
    api: getSimpleLevelList,
    labelField: 'name',
    valueField: 'id',
  },
},
```

### 3.5 系统部门表 `system_dept`（树形结构）

对应前端类型定义：[dept/index.ts](apps/web-antd/src/api/system/dept/index.ts#L5-L16)

```sql
CREATE TABLE `system_dept` (
  `id`             BIGINT       NOT NULL AUTO_INCREMENT COMMENT '部门编号',
  `name`           VARCHAR(30)  NOT NULL COMMENT '部门名称',
  `parent_id`      BIGINT       NOT NULL DEFAULT 0 COMMENT '父部门编号（0=顶级）',
  `sort`           INT          NOT NULL DEFAULT 0 COMMENT '显示顺序',
  `leader_user_id` BIGINT        DEFAULT NULL COMMENT '负责人用户编号',
  `phone`          VARCHAR(20)   DEFAULT NULL COMMENT '联系电话',
  `email`          VARCHAR(50)   DEFAULT NULL COMMENT '邮箱',
  `status`         TINYINT      NOT NULL DEFAULT 0 COMMENT '状态（0开启 1禁用）',
  -- 公共字段
  `creator`        VARCHAR(64)  NOT NULL DEFAULT '',
  `create_time`    DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updater`        VARCHAR(64)  NOT NULL DEFAULT '',
  `update_time`    DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deleted`        BIT(1)       NOT NULL DEFAULT b'0',
  `tenant_id`      BIGINT       NOT NULL DEFAULT 0,
  PRIMARY KEY (`id`),
  KEY `idx_parent_id` (`parent_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='部门表';
```

**树形数据在前端的处理** — 引用自 [data.ts](apps/web-antd/src/views/system/user/data.ts#L59-L69)：

```typescript
{
  fieldName: 'deptId',
  label: '归属部门',
  component: 'ApiTreeSelect',     // 树形下拉选择器
  componentProps: {
    api: async () => {
      const data = await getDeptList();
      return handleTree(data);    // 将扁平数组转换为树形结构（parentId 关联）
    },
    labelField: 'name',           // → system_dept.name
    valueField: 'id',             // → system_dept.id
    childrenField: 'children',    // → 子节点字段名
  },
}
```

### 3.6 代码生成表 `infra_codegen_table` / `infra_codegen_column`

对应前端类型定义：[codegen/index.ts](apps/web-antd/src/api/infra/codegen/index.ts#L7-L89)

```sql
-- 代码生成表定义
CREATE TABLE `infra_codegen_table` (
  `id`                       BIGINT       NOT NULL AUTO_INCREMENT COMMENT '编号',
  `table_id`                 BIGINT        DEFAULT NULL COMMENT '数据源表编号',
  `data_source_config_id`    BIGINT        DEFAULT NULL COMMENT '数据源配置编号',
  `scene`                    TINYINT      NOT NULL DEFAULT 1 COMMENT '场景（1单表 2树表 3主子表）',
  `table_name`               VARCHAR(200) NOT NULL COMMENT '表名',
  `table_comment`            VARCHAR(500) NOT NULL DEFAULT '' COMMENT '表描述',
  `class_name`               VARCHAR(100) NOT NULL DEFAULT '' COMMENT '实体类名',
  `class_comment`            VARCHAR(500) NOT NULL DEFAULT '' COMMENT '类描述',
  `module_name`              VARCHAR(30)  NOT NULL COMMENT '模块名（如 system）',
  `business_name`            VARCHAR(30)  NOT NULL COMMENT '业务名（如 user）',
  `author`                   VARCHAR(30)  NOT NULL DEFAULT '' COMMENT '作者',
  `template_type`            TINYINT      NOT NULL DEFAULT 1 COMMENT '模板类型（1单表CRUD 2树表CRUD 3主子表CRUD）',
  `front_type`               TINYINT       DEFAULT NULL COMMENT '前端类型（10Vue3 20Vue2）',
  `parent_menu_id`           BIGINT        DEFAULT NULL COMMENT '父菜单编号',
  `gen_type`                 VARCHAR(20)   DEFAULT 'zip' COMMENT '生成方式（zip压缩包/ custom自定义路径）',
  `gen_path`                 VARCHAR(200)  DEFAULT '/' COMMENT '自定义路径',
  `remark`                   VARCHAR(500)  DEFAULT NULL COMMENT '备注',
  -- 公共字段
  `creator`                  VARCHAR(64)  NOT NULL DEFAULT '',
  `create_time`              DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updater`                  VARCHAR(64)  NOT NULL DEFAULT '',
  `update_time`              DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deleted`                  BIT(1)       NOT NULL DEFAULT b'0',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='代码生成表';

-- 代码生成字段定义
CREATE TABLE `infra_codegen_column` (
  `id`                       BIGINT       NOT NULL AUTO_INCREMENT COMMENT '编号',
  `table_id`                 BIGINT       NOT NULL COMMENT '归属表编号',
  `column_name`              VARCHAR(200) NOT NULL COMMENT '字段名',
  `data_type`                VARCHAR(100)  DEFAULT NULL COMMENT '字段类型（数据库）',
  `column_comment`           VARCHAR(500) NOT NULL DEFAULT '' COMMENT '字段描述',
  `nullable`                 BIT(1)       NOT NULL DEFAULT b'1' COMMENT '是否可为空',
  `primary_key`              BIT(1)       NOT NULL DEFAULT b'0' COMMENT '是否主键',
  `ordinal_position`         INT          NOT NULL COMMENT '排序',
  `java_type`                VARCHAR(32)  NOT NULL DEFAULT 'String' COMMENT 'Java 属性类型',
  `java_field`               VARCHAR(64)  NOT NULL COMMENT 'Java 属性名',
  `dict_type`                VARCHAR(200)  DEFAULT '' COMMENT '字典类型',
  `example`                  VARCHAR(500)  DEFAULT NULL COMMENT '数据示例',
  `create_operation`         BIT(1)       NOT NULL DEFAULT b'1' COMMENT '是否新增字段',
  `update_operation`         BIT(1)       NOT NULL DEFAULT b'1' COMMENT '是否修改字段',
  `list_operation`           BIT(1)       NOT NULL DEFAULT b'1' COMMENT '是否列表字段',
  `list_operation_condition` VARCHAR(32)  NOT NULL DEFAULT '=' COMMENT '列表查询条件（=/LIKE/BETWEEN/>=/<=）',
  `list_operation_result`    BIT(1)       NOT NULL DEFAULT b'1' COMMENT '是否列表返回字段',
  `html_type`                VARCHAR(32)  NOT NULL DEFAULT 'input' COMMENT '前端组件（input/select/radio等）',
  -- 公共字段
  `creator`                  VARCHAR(64)  NOT NULL DEFAULT '',
  `create_time`              DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updater`                  VARCHAR(64)  NOT NULL DEFAULT '',
  `update_time`              DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deleted`                  BIT(1)       NOT NULL DEFAULT b'0',
  PRIMARY KEY (`id`),
  KEY `idx_table_id` (`table_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='代码生成字段表';
```

**字段类型映射** — 引用自 [codegen/index.ts](apps/web-antd/src/api/infra/codegen/index.ts#L40-L59)：

```typescript
export interface CodegenColumn {
  id: number;
  tableId: number;
  columnName: string;           // 数据库字段名（蛇形）
  dataType: string;             // 数据库类型：varchar/bigint/datetime/tinyint...
  columnComment: string;        // 字段注释
  nullable: number;             // 0不可为空 / 1可为空
  primaryKey: number;           // 是否主键（0否/1是）
  ordinalPosition: number;      // 字段顺序
  javaType: string;             // Java类型：String/Long/Integer/Date/BigDecimal...
  javaField: string;            // Java字段名（驼峰）：userName
  dictType: string;             // 字典类型：system_user_sex
  example: string;              // 示例值
  createOperation: number;      // 是否在新增表单显示
  updateOperation: number;      // 是否在编辑表单显示
  listOperation: number;        // 是否在搜索条件显示
  listOperationCondition: string; // 查询方式：=、LIKE、BETWEEN
  listOperationResult: number;  // 是否在列表表格显示
  htmlType: string;             // 前端组件：input/select/radio/switch/datetime...
}
```

### 3.7 退款订单表 `pay_refund`

对应前端类型定义：[refund/index.ts](apps/web-antd/src/api/pay/refund/index.ts#L7-L38)

```sql
CREATE TABLE `pay_refund` (
  `id`                    BIGINT       NOT NULL AUTO_INCREMENT COMMENT '退款编号',
  `no`                    VARCHAR(64)  NOT NULL COMMENT '退款单号',
  `merchant_refund_no`    VARCHAR(64)  NOT NULL COMMENT '商户退款单号',
  `merchant_refund_id`    VARCHAR(64)   DEFAULT NULL COMMENT '商户退款单编号',
  `app_id`                BIGINT       NOT NULL COMMENT '应用编号',
  `channel_id`            BIGINT        DEFAULT NULL COMMENT '渠道编号',
  `channel_code`          VARCHAR(32)   DEFAULT NULL COMMENT '渠道编码',
  `order_id`              BIGINT        DEFAULT NULL COMMENT '支付订单编号',
  `merchant_order_id`     VARCHAR(64)   DEFAULT NULL COMMENT '商户订单号',
  `trade_no`              VARCHAR(64)   DEFAULT NULL COMMENT '支付流水号',
  `notify_url`            VARCHAR(512) NOT NULL COMMENT '异步通知地址',
  `notify_status`         TINYINT      NOT NULL DEFAULT 0 COMMENT '通知状态',
  `status`                TINYINT      NOT NULL DEFAULT 0 COMMENT '退款状态（0等待 10成功 20失败）',
  `pay_price`             INT          NOT NULL COMMENT '支付金额（分）',
  `refund_price`          INT          NOT NULL COMMENT '退款金额（分）',
  `reason`                VARCHAR(255)  DEFAULT NULL COMMENT '退款原因',
  `type`                  TINYINT       DEFAULT NULL COMMENT '退款类型',
  `user_ip`               VARCHAR(50)   DEFAULT NULL COMMENT '用户IP',
  `channel_order_no`      VARCHAR(64)   DEFAULT NULL COMMENT '渠道订单号',
  `channel_refund_no`     VARCHAR(64)   DEFAULT NULL COMMENT '渠道退款单号',
  `channel_error_code`    VARCHAR(128)  DEFAULT NULL COMMENT '渠道错误码',
  `channel_error_msg`     VARCHAR(512)  DEFAULT NULL COMMENT '渠道错误描述',
  `channel_extras`        TEXT          DEFAULT NULL COMMENT '渠道附加数据',
  `expire_time`           DATETIME      DEFAULT NULL COMMENT '失效时间',
  `success_time`          DATETIME      DEFAULT NULL COMMENT '成功时间',
  `notify_time`           DATETIME      DEFAULT NULL COMMENT '通知时间',
  -- 公共字段
  `creator`               VARCHAR(64)  NOT NULL DEFAULT '',
  `create_time`           DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updater`               VARCHAR(64)  NOT NULL DEFAULT '',
  `update_time`           DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deleted`               BIT(1)       NOT NULL DEFAULT b'0',
  `tenant_id`             BIGINT       NOT NULL DEFAULT 0,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_no` (`no`),
  UNIQUE KEY `uk_merchant_refund_no` (`merchant_refund_no`, `deleted`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='退款订单';
```

### 3.8 定时任务表 `infra_job`

对应前端类型定义：[job/index.ts](apps/web-antd/src/api/infra/job/index.ts#L7-L19)

```sql
CREATE TABLE `infra_job` (
  `id`               BIGINT       NOT NULL AUTO_INCREMENT COMMENT '任务编号',
  `name`             VARCHAR(32)  NOT NULL COMMENT '任务名称',
  `status`           TINYINT      NOT NULL DEFAULT 1 COMMENT '任务状态（1初始化 2运行中 3暂停）',
  `handler_name`     VARCHAR(64)  NOT NULL COMMENT '处理器名称',
  `handler_param`    VARCHAR(255)  DEFAULT NULL COMMENT '处理器参数',
  `cron_expression`  VARCHAR(32)  NOT NULL COMMENT 'CRON 表达式',
  `retry_count`      INT          NOT NULL DEFAULT 0 COMMENT '重试次数',
  `retry_interval`   INT          NOT NULL DEFAULT 0 COMMENT '重试间隔（毫秒）',
  `monitor_timeout`  INT          NOT NULL DEFAULT 0 COMMENT '监控超时时间（毫秒）',
  -- 公共字段
  `creator`          VARCHAR(64)  NOT NULL DEFAULT '',
  `create_time`      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updater`          VARCHAR(64)  NOT NULL DEFAULT '',
  `update_time`      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deleted`          BIT(1)       NOT NULL DEFAULT b'0',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_handler_name` (`handler_name`, `handler_param`, `deleted`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='定时任务表';
```

### 3.9 租户表 `system_tenant`

对应前端类型定义：[tenant/index.ts](apps/web-antd/src/api/system/tenant/index.ts#L7-L17)

```sql
CREATE TABLE `system_tenant` (
  `id`              BIGINT       NOT NULL AUTO_INCREMENT COMMENT '租户编号',
  `name`            VARCHAR(30)  NOT NULL COMMENT '租户名称',
  `package_id`      BIGINT       NOT NULL COMMENT '租户套餐编号',
  `contact_name`    VARCHAR(30)  NOT NULL DEFAULT '' COMMENT '联系人',
  `contact_mobile`  VARCHAR(20)  NOT NULL DEFAULT '' COMMENT '联系电话',
  `account_count`   INT          NOT NULL DEFAULT 0 COMMENT '账号配额（-1不限）',
  `expire_time`     DATETIME     NOT NULL COMMENT '过期时间',
  `websites`        VARCHAR(255)  DEFAULT NULL COMMENT '绑定域名（逗号分隔多个）',
  `status`          TINYINT      NOT NULL DEFAULT 0 COMMENT '状态（0开启 1禁用）',
  -- 公共字段
  `creator`         VARCHAR(64)  NOT NULL DEFAULT '',
  `create_time`     DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updater`         VARCHAR(64)  NOT NULL DEFAULT '',
  `update_time`     DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deleted`         BIT(1)       NOT NULL DEFAULT b'0',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='租户表';
```

**租户域名绑定** — 前端通过域名自动识别租户，参考 [auth.ts](apps/web-antd/src/api/core/auth.ts#L109-L113)：

```typescript
/** 使用租户域名，获得租户信息 */
export async function getTenantByWebsite(website: string) {
  return requestClient.get<AuthApi.TenantResult>(
    `/system/tenant/get-by-website?website=${website}`,
  );
}
```

### 3.10 CRM 客户表 `crm_customer`

对应前端类型定义：[customer/index.ts](apps/web-antd/src/api/crm/customer/index.ts#L9-L39)

```sql
CREATE TABLE `crm_customer` (
  `id`                    BIGINT       NOT NULL AUTO_INCREMENT COMMENT '客户编号',
  `name`                  VARCHAR(100) NOT NULL COMMENT '客户名称',
  `follow_up_status`      BIT(1)       NOT NULL DEFAULT b'0' COMMENT '跟进状态',
  `contact_last_time`     DATETIME      DEFAULT NULL COMMENT '最后跟进时间',
  `contact_last_content`  VARCHAR(500)  DEFAULT NULL COMMENT '最后跟进内容',
  `contact_next_time`     DATETIME      DEFAULT NULL COMMENT '下次联系时间',
  `owner_user_id`         BIGINT       NOT NULL COMMENT '负责人用户编号',
  `lock_status`           BIT(1)       NOT NULL DEFAULT b'0' COMMENT '锁定状态（0未锁定 1已锁定）',
  `deal_status`           BIT(1)       NOT NULL DEFAULT b'0' COMMENT '成交状态',
  `mobile`                VARCHAR(20)   DEFAULT '' COMMENT '手机号',
  `telephone`             VARCHAR(20)   DEFAULT '' COMMENT '电话',
  `qq`                    VARCHAR(20)   DEFAULT '' COMMENT 'QQ',
  `wechat`                VARCHAR(50)   DEFAULT '' COMMENT '微信',
  `email`                 VARCHAR(50)   DEFAULT '' COMMENT '邮箱',
  `area_id`               BIGINT        DEFAULT NULL COMMENT '所在地编号',
  `detail_address`        VARCHAR(255)  DEFAULT '' COMMENT '详细地址',
  `industry_id`           INT           DEFAULT NULL COMMENT '所属行业',
  `level`                 TINYINT       DEFAULT NULL COMMENT '客户等级',
  `source`                TINYINT       DEFAULT NULL COMMENT '客户来源',
  `remark`                VARCHAR(500)  DEFAULT NULL COMMENT '备注',
  `pool_day`              INT           DEFAULT NULL COMMENT '进入公海天数',
  -- 公共字段
  `creator`               VARCHAR(64)  NOT NULL DEFAULT '',
  `create_time`           DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updater`               VARCHAR(64)  NOT NULL DEFAULT '',
  `update_time`           DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deleted`               BIT(1)       NOT NULL DEFAULT b'0',
  `tenant_id`             BIGINT       NOT NULL DEFAULT 0,
  PRIMARY KEY (`id`),
  KEY `idx_owner_user_id` (`owner_user_id`),
  KEY `idx_industry_id` (`industry_id`),
  KEY `idx_level` (`level`),
  KEY `idx_source` (`source`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='CRM 客户';
```

---

## 4. 字典与枚举映射

系统通过**数据字典**管理枚举值，字典类型定义在 [dict-enum.ts](packages/constants/src/dict-enum.ts)。

### 4.1 字典表结构

```sql
-- 字典类型表
CREATE TABLE `system_dict_type` (
  `id`          BIGINT       NOT NULL AUTO_INCREMENT,
  `name`        VARCHAR(100) NOT NULL DEFAULT '' COMMENT '字典名称',
  `type`        VARCHAR(100) NOT NULL DEFAULT '' COMMENT '字典类型（唯一标识）',
  `status`      TINYINT      NOT NULL DEFAULT 0 COMMENT '状态（0开启 1禁用）',
  `remark`      VARCHAR(500)  DEFAULT NULL,
  -- 公共字段
  `creator`     VARCHAR(64)  NOT NULL DEFAULT '',
  `create_time` DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updater`     VARCHAR(64)  NOT NULL DEFAULT '',
  `update_time` DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deleted`     BIT(1)       NOT NULL DEFAULT b'0',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_type` (`type`, `deleted`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='字典类型';

-- 字典数据表
CREATE TABLE `system_dict_data` (
  `id`          BIGINT       NOT NULL AUTO_INCREMENT,
  `sort`        INT          NOT NULL DEFAULT 0 COMMENT '显示顺序',
  `label`       VARCHAR(100) NOT NULL DEFAULT '' COMMENT '字典标签',
  `value`       VARCHAR(100) NOT NULL DEFAULT '' COMMENT '字典键值',
  `dict_type`   VARCHAR(100) NOT NULL DEFAULT '' COMMENT '字典类型',
  `css_class`   VARCHAR(100)  DEFAULT NULL COMMENT '样式属性',
  `list_class`  VARCHAR(100)  DEFAULT NULL COMMENT '表格回显样式',
  `status`      TINYINT      NOT NULL DEFAULT 0,
  `remark`      VARCHAR(500)  DEFAULT NULL,
  -- 公共字段
  `creator`     VARCHAR(64)  NOT NULL DEFAULT '',
  `create_time` DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updater`     VARCHAR(64)  NOT NULL DEFAULT '',
  `update_time` DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deleted`     BIT(1)       NOT NULL DEFAULT b'0',
  PRIMARY KEY (`id`),
  KEY `idx_dict_type` (`dict_type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='字典数据';
```

### 4.2 字典在前端的使用

字典类型常量统一在 [dict-enum.ts](packages/constants/src/dict-enum.ts#L284-L300) 导出：

```typescript
const DICT_TYPE = {
  ...PAY_DICT,       // PAY_CHANNEL_CODE, PAY_ORDER_STATUS, PAY_REFUND_STATUS...
  ...SYSTEM_DICT,    // SYSTEM_USER_SEX, SYSTEM_MENU_TYPE, SYSTEM_ROLE_TYPE...
  ...MEMBER_DICT,    // MEMBER_EXPERIENCE_BIZ_TYPE, MEMBER_POINT_BIZ_TYPE
  ...CRM_DICT,       // CRM_CUSTOMER_LEVEL, CRM_CUSTOMER_SOURCE...
  ...BPM_DICT,       // BPM_TASK_STATUS, BPM_PROCESS_INSTANCE_STATUS...
  ...INFRA_DICT,     // INFRA_JOB_STATUS, INFRA_FILE_STORAGE...
  // ... 更多模块
} as const;

export { DICT_TYPE };
```

**在表单中使用字典作为下拉选项** — 参考 [order/data.ts](apps/web-antd/src/views/pay/order/data.ts#L29-L37)：

```typescript
{
  fieldName: 'channelCode',
  label: '支付渠道',
  component: 'Select',
  componentProps: {
    options: getDictOptions(DICT_TYPE.PAY_CHANNEL_CODE, 'string'),
    // getDictOptions 从后端字典缓存中获取 [{label, value}] 列表
    // 第二个参数 'string' 表示 value 为字符串类型（默认 'number'）
  },
},
```

**在表格中使用字典渲染标签** — 参考 [order/data.ts](apps/web-antd/src/views/pay/order/data.ts#L122-L129)：

```typescript
{
  field: 'status',
  title: '支付状态',
  cellRender: {
    name: 'CellDict',
    props: { type: DICT_TYPE.PAY_ORDER_STATUS },
    // CellDict 组件根据 value（0/10/20）自动映射为字典 label + 颜色标签
  },
},
```

### 4.3 模块字典类型速查

| 字典类型常量 | 字典 Key | 适用字段 | 选项值示例 |
|-------------|----------|----------|-----------|
| `COMMON_STATUS` | `common_status` | `status` | 0=开启, 1=禁用 |
| `SYSTEM_USER_SEX` | `system_user_sex` | `sex` | 0=未知, 1=男, 2=女 |
| `SYSTEM_MENU_TYPE` | `system_menu_type` | `type` | 1=目录, 2=菜单, 3=按钮 |
| `SYSTEM_DATA_SCOPE` | `system_data_scope` | `dataScope` | 1=全部, 2=指定部门, 3=本部门... |
| `PAY_CHANNEL_CODE` | `pay_channel_code` | `channelCode` | wx_pub/wx_lite/alipay_pc... |
| `PAY_ORDER_STATUS` | `pay_order_status` | `status` | 0=未支付, 10=已支付, 20=已关闭 |
| `PAY_REFUND_STATUS` | `pay_refund_status` | `status` | 0=退款中, 10=退款成功, 20=退款失败 |
| `INFRA_JOB_STATUS` | `infra_job_status` | `status` | 1=初始化, 2=运行中, 3=暂停 |
| `INFRA_FILE_STORAGE` | `infra_file_storage` | `storage` | 1=数据库, 10=MinIO, 20=S3... |
| `MEMBER_POINT_BIZ_TYPE` | `member_point_biz_type` | `bizType` | 积分业务类型 |
| `CRM_CUSTOMER_LEVEL` | `crm_customer_level` | `level` | 客户等级 |
| `CRM_CUSTOMER_SOURCE` | `crm_customer_source` | `source` | 客户来源 |

---

## 5. 接入自有数据库步骤

### 5.1 整体流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    接入自有数据库流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  步骤 1: 创建数据库 & 配置数据源                                   │
│    ↓                                                              │
│  步骤 2: 按照规范创建业务表                                        │
│    ↓                                                              │
│  步骤 3: 后端配置数据源（Nacos/yml）                               │
│    ↓                                                              │
│  步骤 4: 使用代码生成器生成前后端代码                               │
│    ↓                                                              │
│  步骤 5: 前端创建 API 接口 & 视图页面                              │
│    ↓                                                              │
│  步骤 6: 配置菜单 & 权限                                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 步骤 1：创建数据库

```sql
-- 创建数据库
CREATE DATABASE IF NOT EXISTS `your_database`
  DEFAULT CHARACTER SET utf8mb4
  DEFAULT COLLATE utf8mb4_general_ci;

USE `your_database`;
```

### 5.3 步骤 2：按照规范建表

按照本文件 [第 2 节](#2-公共字段约定) 的公共字段规范创建业务表。以自定义业务模块为例：

```sql
CREATE TABLE `your_module_your_entity` (
  `id`            BIGINT       NOT NULL AUTO_INCREMENT COMMENT '主键',
  `name`          VARCHAR(100) NOT NULL COMMENT '名称',
  `type`          TINYINT       DEFAULT 0 COMMENT '类型（关联字典 your_type）',
  `status`        TINYINT      NOT NULL DEFAULT 0 COMMENT '状态（0开启 1禁用）',
  `sort`          INT          NOT NULL DEFAULT 0 COMMENT '排序',
  `remark`        VARCHAR(500)  DEFAULT NULL COMMENT '备注',
  -- 公共字段（必须包含）
  `creator`       VARCHAR(64)  NOT NULL DEFAULT '' COMMENT '创建者',
  `create_time`   DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updater`       VARCHAR(64)  NOT NULL DEFAULT '' COMMENT '更新者',
  `update_time`   DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted`       BIT(1)       NOT NULL DEFAULT b'0' COMMENT '是否删除',
  `tenant_id`     BIGINT       NOT NULL DEFAULT 0 COMMENT '租户编号',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='你的业务实体';
```

### 5.4 步骤 3：后端数据源配置

在 Nacos 配置中心或 `application.yaml` 中配置数据源：

```yaml
spring:
  datasource:
    dynamic:
      datasource:
        master:
          url: jdbc:mysql://127.0.0.1:3306/your_database?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai
          username: root
          password: your_password
          driver-class-name: com.mysql.cj.jdbc.Driver
        # 如果需要多数据源
        slave:
          url: jdbc:mysql://127.0.0.1:3306/your_other_db?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai
          username: root
          password: your_password
```

### 5.5 步骤 4：使用代码生成器

在基础设施 → 代码生成 菜单中：

1. **导入表**：选择数据源和表名，点击「导入」
2. **配置生成信息**：
   - 模块名（moduleName）：如 `yourmodule`
   - 业务名（businessName）：如 `yourentity`
   - 类名（className）：如 `YourEntity`
   - 作者（author）
   - 父菜单（parentMenuId）
3. **配置字段**：为每个字段设置 Java 类型、前端组件、是否显示在新增/编辑/列表/搜索中
4. **预览/下载代码**：生成后端（Controller/Service/Mapper/DO/VO）和前端（API/views）代码

API 参考：[codegen/index.ts](apps/web-antd/src/api/infra/codegen/index.ts#L162-L166)：

```typescript
/** 基于数据库的表结构，创建代码生成器的表定义 */
export function createCodegenList(data: InfraCodegenApi.CodegenCreateListReqVO) {
  return requestClient.post('/infra/codegen/create-list', data);
}
```

### 5.6 步骤 5：前端接入

在前端 [apps/web-antd/src/api/](apps/web-antd/src/api/) 目录下创建模块目录，按以下模板编写 API 文件：

```typescript
// 文件位置：apps/web-antd/src/api/yourmodule/yourentity/index.ts
import type { PageParam, PageResult } from '@vben/request';
import { requestClient } from '#/api/request';

export namespace YourModuleApi {
  export interface YourEntity {
    id?: number;
    name: string;
    type: number;
    status: number;
    sort: number;
    remark: string;
    createTime?: Date;
  }
}

export function getPage(params: PageParam) {
  return requestClient.get<PageResult<YourModuleApi.YourEntity>>(
    '/yourmodule/yourentity/page',
    { params },
  );
}
export function get(id: number) {
  return requestClient.get<YourModuleApi.YourEntity>(`/yourmodule/yourentity/get?id=${id}`);
}
export function create(data: YourModuleApi.YourEntity) {
  return requestClient.post('/yourmodule/yourentity/create', data);
}
export function update(data: YourModuleApi.YourEntity) {
  return requestClient.put('/yourmodule/yourentity/update', data);
}
export function remove(id: number) {
  return requestClient.delete(`/yourmodule/yourentity/delete?id=${id}`);
}
```

在 [apps/web-antd/src/views/](apps/web-antd/src/views/) 下创建视图目录，包含 `data.ts`（Schema 定义）和 `index.vue`（页面组件）。

### 5.7 步骤 6：配置菜单

在系统管理 → 菜单管理中新增菜单：

| 字段 | 值 |
|------|-----|
| 菜单类型 | 目录 / 菜单 / 按钮 |
| 菜单名称 | 你的菜单名 |
| 路由地址 | `/yourmodule/yourentity` |
| 组件路径 | `yourmodule/yourentity/index` |
| 权限标识 | `yourmodule:yourentity:query` |

菜单数据存储在 `system_menu` 表中，前端类型定义参考 [menu/index.ts](apps/web-antd/src/api/system/menu/index.ts#L5-L21)：

```typescript
export interface Menu {
  id: number;
  name: string;           // 菜单名称
  permission: string;     // 权限标识（如 yourmodule:yourentity:create）
  type: number;           // 1=目录 2=菜单 3=按钮
  sort: number;
  parentId: number;       // 父菜单 ID
  path: string;           // 路由路径
  icon: string;
  component: string;      // 前端组件路径（如 yourmodule/yourentity/index）
  status: number;
  visible: boolean;       // 是否可见
  keepAlive: boolean;     // 是否缓存
  createTime: Date;
}
```

---

## 6. 代码生成器使用

### 6.1 支持的场景

代码生成器（[codegen/index.ts](apps/web-antd/src/api/infra/codegen/index.ts)）支持三种场景：

| scene 值 | 场景 | 说明 |
|----------|------|------|
| 1 | 单表 CRUD | 标准列表 + 新增/编辑/删除/导出 |
| 2 | 树表 CRUD | 树形结构表（如部门、分类），使用 parent_id |
| 3 | 主子表 CRUD | 主表 + 子表（一对多），如订单 + 订单项 |

### 6.2 字段 htmlType 与前端组件映射

| htmlType 值 | 前端组件 | 适用字段类型 |
|-------------|----------|-------------|
| `input` | Input 文本框 | VARCHAR 普通文本 |
| `textarea` | Textarea 文本域 | VARCHAR/TEXT 长文本 |
| `select` | Select 下拉选择 | 关联字典的字段（如 type/status） |
| `radio` | RadioGroup 单选 | status/sex 等枚举字段 |
| `checkbox` | CheckboxGroup 多选 | 数组/逗号分隔字符串 |
| `datetime` | DatePicker 日期时间 | DATETIME 类型 |
| `imageUpload` | 图片上传 | VARCHAR(512) 图片 URL |
| `fileUpload` | 文件上传 | VARCHAR(512) 文件 URL |
| `editor` | 富文本编辑器 (TinyMCE) | TEXT 富文本内容 |
| `switch` | Switch 开关 | TINYINT 布尔字段 |
| `number` | InputNumber 数字输入 | INT/BIGINT 数值字段 |

### 6.3 查询条件（listOperationCondition）映射

| 值 | SQL 条件 | 适用场景 |
|----|----------|----------|
| `=` | `WHERE field = #{value}` | 精确匹配（status, type） |
| `!=` | `WHERE field != #{value}` | 排除匹配 |
| `LIKE` | `WHERE field LIKE CONCAT('%', #{value}, '%')` | 模糊搜索（name, title） |
| `BETWEEN` | `WHERE field BETWEEN #{begin} AND #{end}` | 时间范围（create_time） |
| `>=` | `WHERE field >= #{value}` | 大于等于 |
| `<=` | `WHERE field <= #{value}` | 小于等于 |

### 6.4 同步数据库变更

当表结构变更后，可通过同步功能更新代码生成配置：

```typescript
// 引用自 codegen/index.ts
/** 基于数据库的表结构，同步数据库的表和字段定义 */
export function syncCodegenFromDB(tableId: number) {
  return requestClient.put('/infra/codegen/sync-from-db', {}, {
    params: { tableId },
  });
}
```

---

## 7. 前后端字段映射速查表

### 7.1 数据库类型 → Java 类型 → TypeScript 类型

| 数据库类型 | Java 类型 | TS 类型 | 示例字段 |
|-----------|-----------|---------|----------|
| BIGINT | Long | `number` | `id`, `dept_id`, `app_id` |
| INT | Integer | `number` | `status`, `sort`, `price`（分） |
| TINYINT | Integer | `number` | `sex`, `status`, `type` |
| VARCHAR | String | `string` | `username`, `name`, `mobile` |
| TEXT | String | `string` | `remark`, `channel_notify_data` |
| DATETIME | LocalDateTime | `Date` | `create_time`, `login_date` |
| DECIMAL(10,6) | BigDecimal | `number` | `channel_fee_rate` |
| BIT(1) | Boolean | `boolean` | `deleted`, `lock_status` |

### 7.2 字段命名转换规则

| 数据库字段（蛇形） | Java 字段（驼峰） | TS 字段（驼峰） |
|-------------------|------------------|----------------|
| `user_name` | `userName` | `userName` |
| `dept_id` | `deptId` | `deptId` |
| `create_time` | `createTime` | `createTime` |
| `channel_code` | `channelCode` | `channelCode` |
| `notify_url` | `notifyUrl` | `notifyUrl` |
| `merchant_order_id` | `merchantOrderId` | `merchantOrderId` |
| `channel_fee_rate` | `channelFeeRate` | `channelFeeRate` |

### 7.3 特殊字段处理

| 数据库存储方式 | 前端类型 | 转换说明 |
|---------------|---------|----------|
| `post_ids` VARCHAR(255) 逗号分隔 | `string[]` / `number[]` | 后端拆分为数组返回前端 |
| `tag_ids` VARCHAR(255) 逗号分隔 | `number[]` | 同上 |
| `websites` VARCHAR(255) 逗号分隔 | `string[]` | 多域名绑定数组 |
| `channel_codes` VARCHAR(512) 逗号分隔 | `string[]` | 支付渠道编码数组 |
| `price` INT（分） | `number` | 展示时除以100转为元 |
| `config` TEXT（JSON 字符串） | 解析后的 Object | 后端序列化/反序列化 |
| `avatar` VARCHAR(512) URL | `string` | 前端 ImageUpload 组件 |

---

## 附录：API 加密配置

系统支持 API 请求/响应 AES 加解密，配置在 [.env](apps/web-antd/.env#L28-L35)：

```env
# API 加解密开关
VITE_APP_API_ENCRYPT_ENABLE = true
VITE_APP_API_ENCRYPT_HEADER = X-Api-Encrypt
VITE_APP_API_ENCRYPT_ALGORITHM = AES
# AES 密钥（32位）
VITE_APP_API_ENCRYPT_REQUEST_KEY = 52549111389893486934626385991395
VITE_APP_API_ENCRYPT_RESPONSE_KEY = 96103715984234343991809655248883
```

加密逻辑在 [request.ts](apps/web-antd/src/api/request.ts#L90-L128) 中实现。若接入自有数据库不需要此功能，可在 .env 中设置 `VITE_APP_API_ENCRYPT_ENABLE = false`。

---

> 本文档基于芋道中台框架 v5.7.0 (Vue Vben Admin) 整理，后端对接芋道 yudao-cloud Spring Boot 微服务。
