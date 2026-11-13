# my-Navigation

一个 DSH（DeepSeek Harness）Web 插件：在左侧导航栏增加一个入口，点开后是一个带
**导航 / 代办 / 便签** 三个标签页的面板，标签行右侧有「添加」按钮，点「添加」即向当前
标签页追加一条内容。

- **导航**：名称 + 链接，点击条目在新标签页打开网址；行内**显示完整 URL**（只在行宽不够时省略）
- **代办**：单行事项，带勾选框，可标记完成；**新增时记录时间并显示在行最右侧**；
  支持**全部 / 未完成 / 已完成**筛选（默认全部）
- **便签**：多行文本，以**卡片**形式呈现，按**瀑布流**（CSS 多列）铺排
- **三个标签都支持拖动排序**：拖行首的 ⠿ 手柄；键盘可用 `Alt + ↑/↓` 移动
- **每行都支持操作**：编辑（原地展开表单，回车保存 / Esc 取消）、复制（导航复制完整链接，
  代办和便签复制正文，复制后行内显示「已复制」）、删除
- **删除只是隐藏**：删掉的条目不会消失，而是移到右侧「历史」抽屉里，随时可以恢复
- **历史抽屉**（标签行右上的「历史」按钮，带未读条数）：
  - 顶部「清空所有历史」是两段式确认（第二次点击才真正清空）
  - 每条历史都可单独**复制 / 编辑 / 删除（彻底删除）**
  - 每条历史都有勾选框，选中后底部出现「恢复」和「彻底删除」，可批量处理
  - 历史跨标签汇总（导航/代办/便签删掉的都在一处），按删除时间倒序，`Esc` 或点遮罩关闭
- 数据存在 **Host 侧**：`%DSH_HOME%\storages\my-navigation.json`（默认
  `C:\Users\<你>\.dsh\storages\my-navigation.json`），跨浏览器、跨端口都在
- 页面底部显示当前数据文件的绝对路径
- 支持浅色/深色主题（只用 `--dsw-alias-*` 语义 token）、中英文文案（跟随 DSH 语言设置）

## 目录结构

仓库根目录就是插件包根（`dsh plugin` 从 GitHub 安装时 pnpm 拉的就是这一层）：

```
dsh-plugin-my-navigation/
├─ package.json        # bundle 声明：dsh.bundle.patch + dsh.client（platform/immediately/inject）+ build 脚本
├─ cordis.patch.yml    # 往 profile 图层里插入一行 id=my-navigation
├─ index.js            # Host 半边：状态文件读写 + /api/my-navigation/state 路由
├─ build.mjs           # 只生成两个"内容 chunk"：i18n 与 styles
├─ client.js           # 入口（约 130 行）：加载 chunk、组装 runtime、注册插槽
├─ client.i18n.js      # 生成：zh/en 文案（源 src/config/*.json）
├─ client.styles.js    # 生成：页面 CSS（源 src/styles.css）
├─ client.utils.js     # 纯工具：createId / normalizeUrl / hostOf / textOf / formatStamp / copyText
├─ client.model.js     # 数据形状与纯操作：隐藏/恢复/彻底删除/校验归一/表单载荷/排序/本地镜像
├─ client.data.js      # Host 数据获取：fetchState / pushState / 能力协商
├─ client.ui.js        # 图标、RowForm、RowActions、三个 List 组件、历史抽屉、标签注册表
├─ client.page.js      # Page：状态机、读写编排、头部与历史抽屉
├─ src/
│  ├─ config/zh.json   # 中文文案（唯一真源）
│  ├─ config/en.json   # 英文文案（唯一真源）
│  └─ styles.css       # 页面样式（唯一真源）
├─ icon.svg            # 插件管理卡片图标
├─ locale/en.json      # 插件显示名与描述（英文）
└─ locale/zh.json      # 插件显示名与描述（中文）
```

## 运行时的真正拆分

DSH 的客户端模块系统每个包只加载**一个**产物（`exports["./client"]`），但支持**包内 chunk**：
`client.<name>.js` 会被 `require.async('./client.<name>.js')` 按需取回并执行一次。所以本插件
是**真拆分**，不是"拼成一个大文件"：

- `client.js` 只做三件事：`require.async` 取 chunk、组装一个共享 `runtime` 对象、注册
  侧边栏入口与页面（约 130 行，4.9 KB）
- 每个 chunk 自己注册：`window.__ModuleLoader__.load({ id: 'my-navigation', chunk: 'client.x.js', factory })`
- chunk 之间不能同步 `require`，因此统一约定 **chunk 导出 `(runtime) => exports`**，由入口按
  依赖顺序把每个 chunk 的导出合并进 `runtime`
- 取到脚本的规则（与 DSH 的实现一致）：key 是 `<包名>/<chunk 文件名>`，URL 由入口 bundle 的
  `/plugins/??<包名>/client.js&rev=…` 推导成 `/plugins/<包名>/client.x.js?rev=…`
- 任何 chunk 取不到时，入口仍会注册一个**半双语失败面板**说明"请重启 dsh web"，不会白屏

## 构建

```powershell
cd dsh-plugin-my-navigation   # 进到 clone 下来的仓库根目录
node build.mjs            # 生成 client.i18n.js 与 client.styles.js
node build.mjs --check    # 只校验这两个文件是否与 src/ 一致（不写文件）
```

只有两个 chunk 是生成的，因为它们的内容来自**真正的配置文件**：

- `src/config/zh.json` + `en.json` → `client.i18n.js`
- `src/styles.css` → `client.styles.js`

其余 chunk 是手写文件，直接就是运行时文件，没有构建步骤。`build.mjs` 顺带做这些检查：

- `zh.json` 与 `en.json` 的键集必须完全一致，值不能为空
- 每个手写 chunk 必须用 `client.<name>.js` 命名、`chunk` 字段与文件名一致、`id` 与包名一致
- 所有脚本都过一遍 `new Function(source)` 解析，语法错在构建期暴露
- `client.js` 必须引用到全部 7 个 chunk（改名的文件不会被静默漏掉）

Host 半边（`index.js`）是普通 ESM，没有构建步骤。

## 数据存在哪

```
%DSH_HOME%\storages\my-navigation.json
```

默认 `C:\Users\<你>\.dsh\storages\my-navigation.json`（`DSH_HOME` 有值时以它为准）。
文件由本插件自己维护，格式简单、可手改：

```json
{
  "version": 1,
  "nav":  [{ "id": "…", "title": "DeepSeek", "url": "https://www.deepseek.com" }],
  "todo": [{ "id": "…", "text": "写测试", "done": false, "createdAt": 1794559210704 }],
  "note": [{ "id": "…", "text": "多行\n便签" }]
}
```

数组顺序就是显示顺序（拖动排序改的就是它）；`createdAt` 让代办在行尾显示创建时间。

删掉的条目**不是被移除**，而是留在原数组里并打上标记，历史抽屉读的就是它们：

```json
{ "id": "…", "title": "旧链接", "url": "https://old.example", "hidden": true, "hiddenAt": 1794559210704 }
```

`hidden: true` 让条目从列表里隐藏并按 `hiddenAt` 倒序进入历史；「恢复」会删掉这两个字段，
「彻底删除」才真正从数组里移除。旧文件没有这两个字段也照常读取（视为未删除）。

- 写入走"临时文件 + fsync + rename"，不会出现半截文件
- 读写都经过同一个服务端校验：丢坏行、补 id、限长（条目 500 条，标题 200 字，链接
  2048 字，正文 20000 字），所以手改出错也不会把页面搞崩
- 文件被改坏（不是合法 JSON）时，会先把它改名成 `my-navigation.json.corrupt-<时间戳>`
  保留，再以空数据继续，不会覆盖你的原内容
- 浏览器里只保留一份 `localStorage` 镜像（`dsh.my-navigation.cache.v1`），用于首帧
  占位和 Host 不可达时兜底显示；它不是主存储
- 当前打开的标签页记在 `dsh.my-navigation.tab.v1`，也属于纯 UI 偏好

## HTTP 接口（Host 半边）

| 方法 | 路径 | 说明 |
|---|---|---|
| `GET` / `HEAD` | `/api/my-navigation/state` | 返回 `{ ok, state, file, capabilities }` |
| `PUT` / `POST` | `/api/my-navigation/state` | 请求体是完整 state，返回校验后的 `{ ok, state, file, capabilities }` |

`capabilities` 是能力协商：当前 Host 声明 `["history", "stamps"]`（前者保证
`hidden`/`hiddenAt` 不被剥掉，后者保证代办 `createdAt` 不被剥掉）。**客户端只采用同时满足
两项、且隐藏条目数没变少的回显**；否则保留本地状态并提示「Host 侧插件还是旧版本，请重启
dsh web」——这样半新不旧的 Host 也会表现为一句明确的话，而不是
"删掉的条目又自己回来了"。旧 Host 会在保存时剥掉 `hidden`/`hiddenAt`，这正是症状来源。

安全边界（路由自己管，因为 DSH 的 web server 本身不做鉴权）：

- 优先复用 Host 的请求网闸：取 `ctx.get('connection').requestRejection(req)`，它同时做
  Host/Origin 信任检查与浏览器登录校验，正是 DSH 自己 `/api` 用的那一套。这样路由会
  继承部署里声明的 `trustedHosts`，挂在反代/隧道后面也不会误判
- 没有 `connection` 服务时（非 Web 组合）退回本机的同源规则：写请求必须带与 `Host`
  一致的 `Origin`，读请求允许无 `Origin`（本机 CLI 调试用）
- 服务器默认只绑 `127.0.0.1`，不对网络暴露
- 请求体上限 256 KiB，超了 `413`
- 非法 JSON `400`、未知方法 `405`、未知路径 `404`
- 服务端不信任任何输入，全部字段重新校验
- 并发写按到达顺序串行发布，最后一个状态胜出

## 安装

前置条件：本机已装好 `dsh` CLI（DeepSeek Harness）与 pnpm，Node.js 版本满足该 DSH 版本的
要求。下面命令里的 `web` 换成你自己的 profile 名；**`--profile` 必须放在所有 pnpm 参数之前**。

### 从 GitHub 安装（推荐）

```powershell
dsh plugin --profile web add github:linzhenjie/dsh-plugin-my-navigation
```

等价的完整 URL 写法：

```powershell
dsh plugin --profile web add https://github.com/linzhenjie/dsh-plugin-my-navigation.git
```

锁版本：`github:linzhenjie/dsh-plugin-my-navigation#v1.6.0`（`#` 后接 tag 或 commit sha）。

仓库根目录就是插件包根，`client.js` / `client.*.js` / `cordis.patch.yml` 等运行时产物都已
提交，包里没有 `prepare` 脚本，安装时无需构建，也不会触发 pnpm 10 的 `allowBuilds` 授权。

### 本地开发

```powershell
git clone https://github.com/linzhenjie/dsh-plugin-my-navigation.git
dsh plugin --profile web add ./dsh-plugin-my-navigation
```

相对路径按当前目录解析，本地以 `link:` 安装。改完 `src/` 跑 `node build.mjs`
（`node build.mjs --check` 只校验，不写文件）；`index.js` 改完要重启 `dsh web`。

### 验证与重启

```powershell
dsh plugin --profile web list
```

`$DSH_HOME\profiles\web\package.json` 的 `dependencies` 与 `dsh.profile.bundles` 里都应有
`my-navigation`。**装完必须重启 `dsh web`**：Host 半边是进程内已加载的模块，路由要重启才
注册。重启后左侧导航出现入口，页面底部显示 `%DSH_HOME%\storages\my-navigation.json`。

### 卸载

```powershell
dsh plugin --profile web remove my-navigation
```

## 实现要点

| 目标 | 机制 |
|---|---|
| 左侧导航入口 | `ctx.slots.register({ name: 'sidebar.panellist', id, order, label }, Icon)` |
| 点入口打开页面 | `ctx.slots.register({ name: 'main', key: id }, Page)`；侧边栏负责调用 `ctx.layout.selectPanel(id)` |
| 页面显示时机 | `dsh.client.immediately: true`，页面启动即注册入口 |
| 文案 | `ctx.locale.register(NS, { zh, en })`，组件通过插槽注入的 `t()` 取词 |
| 样式 | `ctx.effect` 注入 `<style>`，卸载时移除；只引用主题 token |
| 状态路由 | `ctx.webServer.register({ kind: 'exact', path, handler })`，返回的 disposer 交给 `ctx.effect` |
| 状态文件 | `node:fs/promises` 的 temp + `fsync` + `rename`，mtime/size 决定是否重读 |
| 行内编辑 | 同一个 `li` 在 `editingId === item.id` 时渲染 `RowEditor`，保存时只发一次整体 PUT |
| 行内复制 | 优先 `navigator.clipboard.writeText`，被拒则退回 `textarea` + `execCommand('copy')` |
| 软删除 | 行内删除只写 `hidden: true` + `hiddenAt`，不移动、不删除数据 |
| 历史抽屉 | 页面内 `position: absolute` 的右侧面板 + 遮罩，`Esc`/点遮罩关闭，列表由 `historyOf(state)` 汇总排序 |
| 批量操作 | `selected: string[]` 存选中的 id，底部批量条在非空时出现，动作都是整体 PUT |
| 清空所有历史 | 两段式确认（4 秒内二次点击），避免误触销毁历史 |
| 三个标签各自渲染 | `tabs.js` 的 `TAB_DEFS` 把「id + 文案键 + List 组件」放在一处；`page.js` 只渲染 `def.List`，没有按 tab 分支 |
| 拖动排序 | 行首 `⠿` 手柄 `draggable`，行接受 `drop`；`useSortable` 把落点翻译成新的 id 顺序，`applyOrder` 只移动这些 id，其余行（隐藏的、被筛选掉的）保持原位 |
| 键盘排序 | 手柄上 `Alt + ↑/↓`，同样走 `moveStep` → `applyOrder` |
| 代办时间 | `payloadFor` 在新建代办时写 `createdAt`；行尾 `formatStamp` 显示（今天 `HH:mm`，本年 `MM-DD HH:mm`，跨年 `YYYY-MM-DD`） |
| 代办筛选 | `TODO_FILTERS` 纯谓词表，筛选只影响渲染，不写数据；排序在筛选后依然按显示顺序计算 |
| 便签瀑布流 | CSS `column-width: 220px` + 卡片 `break-inside: avoid`，无需 JS 布局 |

## 注意

- `sidebar.panellist` 与 `main` 必须用**同一个 id**（这里是 `my-navigation`），否则点入口
  会因为 `main` 里没有对应 key 而报错。
- 输入框的 Enter 处理带 IME 保护（`isComposing`），中文输入法选词时按回车不会误提交。
- 每次变更立刻发一次 PUT（没有防抖，因为增删改都是离散操作）；客户端按顺序发送，
  保证后一次状态不会被前一次的迟到请求覆盖。
- 历史抽屉是**页面内的侧边抽屉**，没有占用 DSH 原生的右侧栏（`sidebar.right`）。要换成
  原生右栏需要注册 `sidebarRightTabs` + `sidebar.right.pane.tab` 插槽，会多两个客户端依赖。
- **改过 `index.js` 就必须重启 `dsh web`**：Host 半边是进程内已加载的模块，浏览器半边
  能热更新、Host 不能。重启前旧 Host 会把 `hidden` 剥掉，客户端现在会检测到并提示，
  但数据要等新 Host 才会真正落盘成隐藏状态（看 `my-navigation.json` 里有没有
  `"hidden": true` 即可判断）。
- 恢复的条目会追加到所属标签的末尾（不还原原来的相对顺序）。
- 拖动排序改的是**数组顺序**，`applyOrder` 只挪被拖动的可见项，隐藏行和被筛选掉的行保持
  原来的槽位，所以"筛选状态下排序"不会打乱看不见的条目。
- 拖动用的是原生 HTML5 拖放（鼠标/触屏板有效）；触屏设备拖不动时可用键盘 `Alt + ↑/↓`
  （手柄可获得焦点）。
- 便签瀑布流用的是 CSS 多列，列内是自上而下填充，所以"拖动排序"改的是列间流动顺序，
  不是精确的二维网格位置。
- 导航行的 URL 用 `text-overflow: ellipsis` 单行显示：能放下就完整显示，放不下才出现
  `…`；完整地址仍然在 `title`（悬停提示）、编辑器输入框和"复制链接"里。
- 鼠标设备上行的操作按钮悬停/聚焦才显现（`@media (hover: hover)`），触屏设备常显，
  避免点不到。
- 一个进程改、另一个进程（比如另开一个 profile）也在改同一文件时，最后一次写入胜出；
  这是单机单人场景下的取舍，没有做跨进程锁。
