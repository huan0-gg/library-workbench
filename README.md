# 资料库在线工作台 · Library Workbench

一个**零依赖、单文件**的在线工作台 HTML 页面。部署到 [WorkBuddy 资料库](https://www.workbuddy.cn)空间后，自动绑定资料库**数据表**，实现：

- **在线持久化存储** —— 所有数据写入资料库数据表，云端保存，关页面、换设备都不丢
- **页面 ↔ 数据表双向同步** —— 页面上的增删改即时写入数据表；数据表被任何人/任何端修改后，页面自动收到变更通知并刷新（`db.onUpdated` 订阅）
- **多端实时协作** —— 两个浏览器窗口同时打开，一边添加任务，另一边 300ms 内自动出现

> 纯 HTML + 内联 CSS/JS，**无任何外部依赖**（无 CDN、无框架、无字体/图表库），单文件 `index.html` 即全部代码。

---

## 功能一览

| 模块 | 能力 | 背后的数据表操作 |
|---|---|---|
| KPI 统计卡 | 全部 / 待办 / 进行中 / 已完成 / 已逾期 五个实时指标 | `db.aggregate`（count + groupBy + 条件过滤） |
| 新建任务表单 | 标题（必填）/ 分类 / 优先级 / 截止日期 / 备注 | `db.addRecord` |
| 任务清单列表 | 全量分页拉取渲染；状态胶囊、优先级配色、逾期标红 | `db.query`（分页 `startCursor` / `hasMore`） |
| 行内操作 | 每条任务可切换状态（待办/进行中/已完成）、删除（二次确认） | `db.updateRecord` / `db.deleteRecord` |
| 双向同步 | 顶栏同步状态指示（已同步/同步中/离线）；表被外部修改时自动刷新 | `db.onUpdated` 订阅 + 300ms 防抖 |
| 容错 | 表单输入本地缓存（失败重试不丢输入）；加载/空态/错误条 + 一键重试 | — |

## 在线体验

作者的已部署示例（数据即作者本人的示例表）：

- **协作编辑态**：`https://www.workbuddy.cn/space/d/c3av6cSWVRbbS459gZwIva`
- **公开发布态**（可直接分享查看）：`https://workbuddy.link/p/c3av6cSWVRbbS459gZwIva`

> 发布态页面查看无需登录；**写入数据需要登录 WorkBuddy 账号**（平台安全策略，未登录提交会被拦截引导登录）。

## 部署到你自己的资料库（三步）

### 第 1 步：创建数据表

在 WorkBuddy 中让智能体帮你建表（把 `database-schema.json` 的内容发给它，或直接说「按 database-schema.json 帮我建一张表」），也可以在资料库界面手动创建同构的表：

- 表名：`工作台任务清单`
- 字段：`标题`(文本)、`分类`(单选：工作/学习/生活/其他)、`优先级`(单选：P0 紧急/P1 重要/P2 一般)、`状态`(单选：待办/进行中/已完成)、`截止日期`(日期)、`备注`(文本)

建好后从表的访问链接中拿到**数据表 ID**：链接形如 `https://www.workbuddy.cn/space/d/<数据表ID>`。

### 第 2 步：替换页面中的 DATABASE_ID

打开 `index.html`，把所有 `REPLACE_WITH_YOUR_DATABASE_ID` 替换为你的数据表 ID（共 6 处：1 处 JS 常量 + 5 处 KPI 卡片的 `data-sp-database-id` 属性）。

### 第 3 步：导入页面并绑定数据表

在 WorkBuddy 中对智能体说：

> 把 index.html 导入到我的资料库，并绑定数据表 `<你的数据表ID>`

导入后即得到协作编辑链接；如需公开分享，再让它「发布这个页面」即可获得 `workbuddy.link` 链接。**页面与数据表的绑定关系**（读写授权、变更订阅）由资料库平台在导入时建立。

## 本地预览

直接双击 `index.html` 用浏览器打开即可看到界面 —— 此时没有资料库数据 SDK 环境（`window.__SMART_PAGE__.database` 不存在），页面会**安全降级为静态模式**并显示提示横幅，不会报错。这是刻意设计的防御行为。

也可以起个静态服务：

```bash
npm run preview     # npx serve .
```

## 技术要点

- **数据 SDK**：`window.__SMART_PAGE__.database`（资料库页面运行环境自动注入），提供 `query` / `addRecord` / `updateRecord` / `deleteRecord` / `getSchema` / `aggregate` / `onUpdated` 等方法
- **分页完整性**：`db.query` 单次只返回一页（最大 200 条），页面用 `loadAll` 递归翻页（`startCursor` + `hasMore`），内置三重防死循环保障（硬上限断路 / cursor 必须推进 / 空页停止）
- **QPS 友好**：所有 SDK 调用串行执行 + `onUpdated` 回调 300ms 防抖 + 刷新互斥锁（页面级 20 QPS / 表级 30 QPS 限制）
- **下拉选项动态化**：分类/优先级/状态选项在运行时从 `db.getSchema` 动态渲染，不硬编码（表结构改了页面自适应）
- **XSS 防护**：所有动态文本一律 `textContent` / `createElement`，不拼接 `innerHTML`
- **数据绑定标记**：渲染自数据表的元素带 `data-sp-bindable="database"` + `data-sp-database-id`，供平台识别数据联动关系
- **兼容性**：ES5 语法（`var` / `function`），无构建步骤，任意现代浏览器可用
- **响应式**：PC / 移动端双适配（`@media` 单列降级）

## 目录结构

```
library-workbench/
├── index.html              # 工作台页面（全部代码，单文件零依赖）
├── database-schema.json    # 数据表「工作台任务清单」建表 schema（部署第 1 步用）
├── package.json            # 项目元数据 + 本地预览脚本
├── LICENSE                 # MIT
└── README.md
```

## License

[MIT](LICENSE)
