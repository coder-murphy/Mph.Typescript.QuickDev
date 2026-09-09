# Mph.VueProject.QuickDev

> 基于 Vue 的个性化快速前端基础设施，开箱即用，专注把常见后台管理/CRUD场景里的重复工作收敛成最小、最易用的封装。

Personalized convenient development of small functions.

---

## 简介

本项目是一套轻量级 Vue3 + TypeScript 前端基础设施，沉淀了后台管理系统开发中最常见的基础能力：

- **HTTP 请求封装**（axios 拦截器、统一错误处理、Token 自动携带）
- **安全存储**（避免 `localStorage` 被浏览器跟踪防护拦截导致的崩溃）
- **登录 / 权限**（用户信息缓存、权限校验）
- **通用 API 包装器**（按实体名一键生成增删改查与分页查询）
- **通用模型**（基础模型、数据追溯、分页模型、结果模型）
- **交互辅助**（表格选中状态同步、表单重置、文件读取）
- **文件服务**（上传、下载）

配合 **Element Plus** 使用，适合快速搭建后台管理类页面。

---

## 技术栈

| 依赖 | 用途 |
| --- | --- |
| [Vue 3](https://vuejs.org/) | 响应式核心、`Ref` 支持 |
| [TypeScript](https://www.typescriptlang.org/) | 类型安全、泛型封装 |
| [Axios](https://axios-http.com/) | HTTP 请求 |
| [Element Plus](https://element-plus.org/) | 表格、表单、消息提示（`ElMessage`） |

---

## 目录结构

```
Mph.VueProject.QuickDev/
├── Core.ts                   # 核心服务：用户信息、权限校验
├── CommonModel.ts            # 基础模型: ModelBase / RetrospectModel
├── QueryPageModel.ts         # 分页查询模型
├── Constants.ts              # 常量（默认请求头等）
├── Interactvity.ts           # UI 交互辅助（表格选中、表单重置等）
├── Authentication/
│   └── LoginInfo.ts          # 登录信息模型
├── Common/
│   └── DisplayItem.ts        # 通用显示项模型
└── Http/
    ├── Request.ts            # axios 实例 + 拦截器
    ├── HttpResult.ts         # 通用结果模型 / 分页结果模型
    ├── ApiWarpper.ts         # 通用 API 包装器
    ├── SafeLocalStorage.ts   # 安全的 localStorage 封装 + token 工具
    └── FileService.ts        # 文件上传/下载
```

---

## 快速开始

### 安装依赖

```bash
npm install axios element-plus
```

### 环境配置

`Http/Request.ts` 中默认使用 `baseURL: '/api'`，可通过 Vite / Webpack 的 `proxy` 或将 `baseURL` 改为环境变量进行配置：

```ts
const service = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || '/api',
  timeout: 10000,
})
```

---

## 核心能力

### 1. HTTP 请求封装（`Http/Request.ts`）

导出默认的 axios 实例，统一处理：

- **请求拦截器**：自动读取 Token 并注入 `Authorization: Bearer <token>`。
- **响应拦截器**：直接返回后端 `data`，简化调用；统一错误提示。
  - 仅当 `401` 且 message 命中「TOKEN / 过期 / 未授权 / 登录已失效」时才清理 Token 并跳转登录页；
  - 普通 `401`（权限/跨域预检）不清理、不跳转，避免误伤；
  - `400 / 403 / 404 / 500` 等通过 `ElMessage` 统一提示。

```ts
import request from './Http/Request'

const data = await request.get('/some/api')
```

### 2. 安全存储（`Http/SafeLocalStorage.ts`）

`safeLocalStorage` 是对原生 `localStorage` 的 try-catch 封装，当浏览器跟踪防护拦截存储访问时**不会抛异常导致页面崩溃**，而是返回 `false` / `null` 并在控制台警告。

同时提供 Token 快捷工具：

```ts
import { safeLocalStorage, getToken, setToken, removeToken } from './Http/SafeLocalStorage'

safeLocalStorage.setItem('user', JSON.stringify(user))
setToken('my-token')
```

> 如需过期时间、命名空间等更复杂能力，建议改用 `localForage` / `store.js`。

### 3. 登录与权限（`Core.ts` + `Authentication/LoginInfo.ts`）

`Core` 提供静态方法管理当前用户与权限：

```ts
import { Core } from './Core'

// 获取/设置当前用户
const user = Core.getCurrentUser()
Core.setCurrentUser(loginInfo)

// 刷新页面后从 localStorage 恢复用户
Core.restoreUserFromStorage()

// 校验用户是否有效（未登录/无 token 时返回 401 Result）
const invalid = Core.getUserInvalidResult()

// 权限校验（大小写不敏感）
const canEdit = Core.hasPermission('user:edit')
```

### 4. 通用模型（`CommonModel.ts` / `QueryPageModel.ts` / `Http/HttpResult.ts`）

- `ModelBase`：实体基类（`id`、前端展示用 `isSelected`）。
- `RetrospectModel`：数据追溯模型（创建/更新/删除人、时间、备注）。
- `QueryPageModel`：分页查询参数（`pageNum / pageSize / total / pageCount`）。
- `HttpResult`：通用接口结果（`isSuccess / message / code / data`）。
- `QueryPageResultModel<T>`：分页查询结果（继承 `HttpResult`，含 `dataList`、`pageInfo`）。

```ts
export interface RetrospectModel extends ModelBase {
  createdAt?: Date
  createdBy?: number
  updatedAt?: Date
  updatedBy?: number
  deletedAt?: Date
  deletedBy?: number
  remarks?: string
}
```

### 5. 通用 API 包装器（`Http/ApiWarpper.ts`）

按实体自动生成列表查询、新增、更新、删除、批量删除等接口，自动处理登录校验与操作人填充：

```ts
import { ApiWarpper } from './Http/ApiWarpper'
import type { RetrospectModel } from './CommonModel'

interface User extends RetrospectModel { userName: string }

const userApi = new ApiWarpper<User>('user')

// 分页查询
const result = await userApi.getList(1, 20)

// 新增 / 更新（自动填充 createdBy / updatedBy）
await userApi.add(newUser)
await userApi.update(user)

// 删除 / 批量删除
await userApi.delete(1)
await userApi.batchDelete([1, 2, 3])
```

约定请求地址前缀为 `/api/{entityName}`，如 `/api/user/query`。若后端规范不同，可基于本类扩展。

### 6. 交互辅助（`Interactvity.ts`）

面向 Element Plus 表格/表单的常用操作：

```ts
import { Interactvity } from './Interactvity'

// 将数据中的 isSelected 同步到表格选中状态
Interactvity.syncRowsToTable(tableRef, items)

// 监听表格选择变化，回写 isSelected（含取消全选清空逻辑）
Interactvity.refreshSelectedRows(tableData, selectedItems)

// 清空表单
Interactvity.resetForm(formRef)

// 读取本地文件
const file = Interactvity.handleReadFile(e)
```

### 7. 文件服务（`Http/FileService.ts`）

```ts
import { FileService } from './Http/FileService'

// 上传
const res = await FileService.uploadFile(file)

// 下载（自动触发浏览器下载并释放内存）
await FileService.downloadFile('reports/report.pdf')

// 下载并自定义文件名
await FileService.downloadCustomFile('reports/report.pdf', '我的报告.pdf')
```

---

## 目录结构说明

| 模块 | 说明 |
| --- | --- |
| `CommonModel.ts` | 实体基础接口 `ModelBase`、数据追溯接口 `RetrospectModel` |
| `Http` | 请求实例、结果模型、API 包装、安全存储、文件服务 |
| `Authentication` | 登录信息模型 |
| `Common` | 跨场景复用的通用模型（如 `DisplayItem`） |
| `Interactvity.ts` | 无 UI 依赖的交互辅助方法 |

---

## 自定义与扩展

- **调整后端接口规范**：可基于 `ApiWarpper` 的增删改查结构自定义新的模板。
- **更换 UI 库**：`Interactvity` 中 `syncRowsToTable` / `resetForm` 依赖 Element Plus API，若使用其他组件库需对应调整；`Request.ts` 中的 `ElMessage` 同理。
- **多环境配置**：将 `Request.ts` 的 `baseURL` 抽取为环境变量。

---

## License

[MIT](./LICENSE)