# E-Viewer 维护计划：登录修复与 issue #180 / #182

> 状态：进行中
> 创建：2026-10-18
> 基线：fork 自 `OpportunityLiu/E-Viewer`，最后上游提交 `eee88b3`（2024-11-02）

---

## 0. 这份文档怎么用

- 本文件是唯一的**总纲**，记录根因、方案、里程碑、验收标准和进度。
- 每完成一项，就在「进度记录」里打勾，并补一行结论；避免依赖对话上下文。
- 会话中断后，从「当前进度」和未勾选项继续即可。
- 不在本文件中保存任何真实 cookie / 账号凭据。

---

## 1. 问题现象

| 编号 | 现象 | 触发路径 |
| --- | --- | --- |
| S1 | 网页登录页加载不出来 | `LogOnDialog` → `WebView.Navigate(https://forums.e-hentai.org/index.php?act=Login)` |
| S2 | Cookie 登录报「无法找到与此错误代码关联的文本 / 无法与服务器建立连接」 | `LogOnAsync` → `ResetExCookieAsync` → `GET https://exhentai.org/` |
| S3 | 填了 igneous 后登录按钮不可用（或直接报格式错误） | `LogOnDialog` 的 XAML/VM 正则校验 |
| S4 | 已登录状态访问里站返回熊猫图 / 空白页 | 里站 igneous 失效或为 `mystery` |

issue 对应关系：

- **#180**：igneous 校验规则过时（非 16 进制字符被拒绝）→ 直接对应 S3。
- **#182**：希望把最低系统版本从 17763 降到 15063/14393 → 与登录无直接关系，见 §7。
- **#164**（旧 issue）：同样描述登录页打不开，属于同一类服务端变更。

---

## 2. 已验证的服务端事实（2026-10-18 实测）

测试方式：PowerShell `Invoke-WebRequest`，使用用户提供的测试 cookie（仅内存中使用，不落盘），UA 分别使用浏览器 UA 与 App UA。

| 编号 | 请求 | 结果 | 结论 |
| --- | --- | --- | --- |
| F1 | `GET https://forums.e-hentai.org/index.php?act=Login` | `403`，标题 `Just a moment...` | 论坛站已挂 Cloudflare JS 挑战 |
| F2 | `POST https://forums.e-hentai.org/index.php?act=Login&CODE=01`（正确表单字段） | `403`，同样是 Cloudflare 挑战 | 网页登录的提交动作也会被拦截 |
| F3 | `GET https://e-hentai.org/bounce_login.php` | `200`，返回 1.3 KB 登录表单，`form action` 指向论坛 `CODE=01` | 主站可渲染表单，但提交仍走论坛（仍被 CF 拦） |
| F4 | 带 `ipb_member_id` + `ipb_pass_hash` + `igneous` 请求 `https://exhentai.org/` | `200`，55 KB 正常页面 | Cookie 本身有效，服务端登录链路可用 |
| F5 | 只带 `ipb_member_id` + `ipb_pass_hash`（不带 igneous）请求 `https://exhentai.org/` | `200`，**0 字节**，`Set-Cookie: igneous=mystery` | 里站不再能用 ipb cookie 自动换取 igneous |
| F6 | 不带任何 cookie 请求 `https://exhentai.org/` | `302` → `https://forums.e-hentai.org/remoteapi.php?...` | 无 cookie 时会被导向被 CF 保护的论坛域 |
| F7 | 带有效 `ipb_*` 请求 `https://e-hentai.org/home.php` | `200`，5 个 `homebox` | 主站登录校验与主页解析结构依旧可用 |
| F8 | 带错误 `ipb_pass_hash` 请求 `home.php` | 跳转 `https://e-hentai.org/bounce_login.php?...`，页面含 `ipb_login_form` | 可用该特征判定 cookie 无效 |
| F9 | 带有效 `ipb_*` 请求 `https://e-hentai.org/uconfig.php` | `200`，40 KB | 设置页可用 |

关键结论：

1. **网页登录不可用是 Cloudflare 造成的**：`forums.e-hentai.org` 对非浏览器客户端返回 JS 挑战，UWP 的 EdgeHTML `WebView` 无法完成挑战，POST 同样被拦。
2. **Cookie 登录本身可行**：只要带上有效的、当前格式的 `igneous`，里站返回正常页面。
3. **igneous 无法再自动获取**：服务端只会下发 `igneous=mystery`。它必须来自已有浏览器会话；因此它是**选填**项：没有它仍可登录并使用表站，只是无法访问里站。
4. `igneous` 当前格式形如 `vfjpw8zgrqtexu1w9`，**不是 16 进制**，旧正则 `^[0-9a-fA-F]*$` 会误拒。

---

## 3. 根因分析

| 编号 | 根因 | 位置 |
| --- | --- | --- |
| RC1 | `igneous` 只允许 16 进制，且被强制转小写 | [Client-User.cs](ExClient/Client-User.cs) `LogOnAsync`；[LogOnDialog.xaml](ExViewer/Views/LogOnDialog.xaml#L87-L94)；[LogOnDialog.xaml.cs](ExViewer/Views/LogOnDialog.xaml.cs#L86-L89) |
| RC2 | 登录页指向被 Cloudflare 保护的论坛站，EdgeHTML WebView 无法过挑战；表单 POST 也被拦 | [Client-User.cs](ExClient/Client-User.cs#L15-L17) + `LogOnDialog` |
| RC3 | `ResetExCookieAsync` 假设「访问里站就能拿到 igneous」，现在只会拿到 `mystery`，随后空响应被当成成功 | [Client-User.cs](ExClient/Client-User.cs#L79-L88) |
| RC4 | 登录成功判定只看本地 cookie 是否存在，不校验服务端是否真的认账 | [Client-User.cs](ExClient/Client-User.cs#L19-L21) |
| RC5 | 空响应 / 熊猫图 / `mystery` 没有转成可读错误，异常直接抛到 UI（出现 HRESULT 文本缺失等提示） | [MyHttpClient.cs](ExClient/Internal/MyHttpClient.cs#L26-L46) |
| RC6 | 启动时 `NeedLogOn` 在应用设置（是否访问里站）应用之前计算，`Host` / 里站模式判断不准确 | [SplashControl.xaml.cs](ExViewer/Views/SplashControl.xaml.cs#L167-L208)；[SettingCollection.cs](ExViewer/Settings/SettingCollection.cs#L54-L59) |

---

## 4. 方案总览

原则：**先让 Cookie 登录稳定可用（P0），再处理网页登录（P1），最后做体验与兼容性收尾（P2/P3）。**

### M0 规划与基线（本文件）
- [x] 明确根因与证据
- [x] 建立构建/验证方式（GitHub Actions，见 §8）

### M1 Cookie 登录修复（P0，解决 S2/S3/S4、issue #180）
改动点：

1. `ExClient/Client-User.cs`
   - 新增 `IsValidIgneousValue`：接受 `[0-9A-Za-z_-]{1,128}`，拒绝 `null/空/mystery/deleted`。
   - `LogOnAsync`：不再 `ToLowerInvariant()`；接受非 16 进制；`mystery/deleted` 视为无效。
   - 同时把 `ipb_member_id` / `ipb_pass_hash` 写入 `e-hentai.org` 与 `exhentai.org`（对齐绿 E/白 E 做法）。
   - 有 `igneous` 时写入两域；无 `igneous` 时仍可登录表站，登录后若开启里站则自动回退表站。
   - 登录后做**服务端校验**：
     - 主站：请求 `home.php`，若页面含 `ipb_login_form` 或跳到 `bounce_login` → 凭据无效。
     - 里站：请求 `exhentai.org/`，若为空响应 / `igneous=mystery` / kokomade → 删除失效 igneous，回退表站，不阻断登录。
   - `ResetExCookieAsync` 已删除：它假设「访问里站就能拿到 igneous」，但现在只会拿到 `mystery`，保留只会掩盖错误。
   - `NeedLogOn`：只要求 `ipb_member_id` / `ipb_pass_hash`；igneous 选填，不阻断登录。
2. `ExViewer/Views/LogOnDialog.xaml`
   - igneous 输入框正则改为 `^[0-9A-Za-z_-]*$`。
   - 文案标注「可选，仅访问里站需要」。
3. `ExViewer/Views/LogOnDialog.xaml.cs`
   - `CanLogOn` 放松 igneous 校验；保留 `ipb_pass_hash` 32 位 hex 校验。
   - 无有效 igneous 时先复用同账号已有值；仍无则预填 cookie 表单并提示可直接登录表站；不再阻断登录。
   - `injectLogOnPage` 一并读取 `igneous`（若 WebView 上下文可读）。
4. `ExClient/Strings/*/Resources.resjson`
   - 无新增错误键；igneous 失效时直接回退，不再抛专用错误。四种语言同步。
5. `ExClient/Internal/MyHttpClient.cs`
   - 将空响应、`igneous=mystery`、`0x80072EFD` 等网络/HRESULT 转成可读文案。
6. `ExViewer/Views/SplashControl.xaml.cs`
   - 在 `NeedLogOn` 判定前应用 `SettingCollection.Apply()`，使 `Host` 正确。

验收：
- 仅填 `ipb_member_id` + `ipb_pass_hash` + 新格式 `igneous` → 登录成功，里站可打开。
- 里站模式不填 igneous → 仍可登录表站，并提示里站不可用；不会出现连接错误或阻断登录。
- 错误 passhash → 提示凭据错误，而不是「无法与服务器建立连接」。
- 旧格式（16 进制）igneous 仍被接受。

### M2 网页登录修复（P1，解决 S1）
方案：**用 WinUI 2 的 `WebView2`（Chromium）替换 EdgeHTML `WebView`**，仅用于登录对话框。

依据：
- 项目已引用 `Microsoft.UI.Xaml 2.8.6`，其中包含 `Microsoft.UI.Xaml.Controls.WebView2`，无需新增 NuGet 包。
- WebView2 运行时在本机已安装；Win10 1803+ / Win11 通常随 Edge 提供。
- Chromium 可正常完成 Cloudflare 挑战，登录后可直接读取 HttpOnly cookie（`CoreWebView2.CookieManager.GetCookiesAsync`）。

改动点：
1. `LogOnDialog.xaml`：`WebView` → `muxc:WebView2`。
2. `LogOnDialog.xaml.cs`：
   - `EnsureCoreWebView2Async()` 后导航 `https://e-hentai.org/bounce_login.php`（或论坛登录页）。
   - 用 `NavigationCompleted` / `CoreWebView2.DOMContentLoaded` 替代 `DOMContentLoaded`。
   - 用 `ExecuteScriptAsync` 替代 `InvokeScriptAsync`。
   - 登录完成后从 `e-hentai.org`、`exhentai.org`、`forums.e-hentai.org` 读取 cookie，交给 `Client.LogOnAsync` 完成落库。
   - 依赖包已包含 WebView2 运行时；不再保留旧 `WebView` 回退。
3. 资源文案：新增网络错误、连接错误、无 igneous 回退等提示。

验收：
- 网页登录可显示登录表单并完成登录。
- 登录后里站可直接访问（前提是浏览器会话下发了 igneous）。
- 无有效 igneous 时可登录表站，并给出回退提示。

风险：WebView2 会增加包体/运行时依赖；若 CI 构建失败或运行时兼容性差，保留旧 WebView + Cookie 登录主路径（M2 可回滚，不影响 M1）。

### M3 体验与稳健性（P2）
- 里站访问失败时统一提示「igneous 已过期，请重新用 Cookie 登录」。
- 设置页/账号页增加「刷新 igneous」入口（若可行）；否则明确说明需重新登录获取。
- 启动时对过期 cookie 做一次轻量校验，失败不阻塞启动。
- 复查 issue #179/#181 等近期 bug 是否由同一批 DOM/接口变更引起（按需，不扩大范围）。

### M4 构建与发布验证（P2）
- CI（GitHub Actions）四架构构建通过。
- 产物安装后按 §5 测试矩阵回归。
- 更新 README/Release Notes，说明新的登录方式。

---

## 5. 测试矩阵

| 场景 | 输入 | 期望 |
| --- | --- | --- |
| T1 主站 Cookie 登录 | member + passhash，无 igneous | 成功，可浏览 e-hentai |
| T2 里站 Cookie 登录 | member + passhash + 新格式 igneous | 成功，可浏览 exhentai |
| T3 里站缺 igneous | member + passhash | 明确提示需要 igneous |
| T4 igneous 错误/过期 | 有效 member + 错误 igneous | 提示 igneous 无效，可重试 |
| T5 passhash 错误 | 有效 member + 错误 hash | 提示凭据错误 |
| T6 旧格式 igneous | 16 进制旧值 | 正常接受 |
| T7 网页登录 | 论坛账号密码 | WebView2 内完成登录并落库 |
| T8 无 WebView2 运行时 | 移除运行时 | 回退提示，不崩溃 |
| T9 启动即里站模式 | VisitEx=true + 无/过期 igneous | 仍可登录，自动回退表站并提示 |
| T10 网络异常 | 断网/代理 | 可读网络错误，不出现 HRESULT 文本缺失 |

---

## 6. 风险与回滚

| 风险 | 影响 | 应对 |
| --- | --- | --- |
| 本地无 MSBuild/VS，无法编译验证 | 改动可能引入编译错误 | 依赖 GitHub Actions；每次推送前做静态检查；必要时请用户安装 VS 构建工具 |
| WebView2 在某些 Win10 环境不可用 | 网页登录回退 | M2 做运行时检测 + 旧 WebView 回退 |
| 服务端继续变更 | 新格式再次失效 | 校验逻辑集中到 `Client-User.cs`，便于单点修改；错误文案区分“凭据错误/igneous 过期/网络错误” |
| 修改 `NeedLogOn` 影响老用户 | 启动被要求重新登录 | igneous 不再阻断登录；无 igneous 自动回退表站 |
| 误把测试 cookie 提交 | 账号泄露 | 测试 cookie 只存在于本地命令历史；文档与代码不写入 |

回滚：每个里程碑独立提交；M2 可单独 revert，不影响 M1。

---

## 7. Issue 结论

### #180（igneous 校验）
- **确认是 bug，M1 修复。**
- 服务端 igneous 已变为大小写敏感的字母数字串；旧正则和强制小写都会破坏登录。
- 同时补上 `mystery/deleted` 的识别，避免“假登录”。

### #182（最低版本 17763 → 15063/14393）
- **暂不实现，建议关闭/说明。**
- 事实：`TargetPlatformMinVersion` 在 2022-11-04 的 `ba8202f`（升级 WinUI 2.5 → 2.8、增加 ARM64）中从 `16299` 提到 `17763`。
- WinUI 2.8 的最低支持系统就是 17763；要降到 15063/14393 必须整体降级 WinUI 与相关控件，且与本计划要用的 `WebView2`（要求 WinUI ≥2.6 / Win10 1803+）冲突。
- 15063/14393 对应 Windows 10 Mobile / 早期桌面版本，均已停止支持；为它们牺牲现代登录修复不划算。
- 结论：维持 17763；在 issue 中给出技术原因。若未来确需支持，单独立项做“低版本构建分支”。

---

## 8. 构建与验证方式

本机现状（2026-10-18）：
- `C:\Program Files (x86)\Microsoft Visual Studio\18` 为空目录，无 `MSBuild.exe` / `vswhere.exe`。
- 无 .NET SDK（`dotnet` 不可用）。
- Windows SDK 存在（`10.0.14393.0` ~ `10.0.26100.0`），但 UWP XAML targets 随 VS 安装，无法单独构建。

因此：
1. **编译验证走 GitHub Actions**（`.github/workflows/ci.yml`，四架构）。需要把改动推到 fork 后触发。
2. 本地只做静态检查 + 服务端行为探测（本文件 §2 的方法）。
3. 若需要本地构建，安装 “Visual Studio Build Tools + UWP 工作负载” 后再执行：
   ```powershell
   msbuild ExViewer.sln /p:Configuration=Release /p:Platform=x64
   ```

---

## 9. 当前进度

- [x] 克隆 fork，确认基线 `eee88b3`
- [x] 完成服务端行为探测（F1–F9）
- [x] 完成根因定位（RC1–RC6）
- [x] 输出本计划
- [x] M1 Cookie 登录修复（已通过真机验证）
- [x] M2 网页登录切换到 WebView2（端到端通过，用户实测登录成功；已加空闲保活）
- [x] M3 启动流程里站模式校验、错误文案（已完成基础部分）
- [x] M4 CI 构建通过（x86 / x64 / arm / arm64 全部成功）
- [ ] 真机回归（T1、T3–T10）与 #182 结论回帖

### 变更清单（M1 + M2）

| 文件 | 改动 |
| --- | --- |
| `ExClient/Client-User.cs` | igneous 新格式校验；双域写入 cookie；登录后服务端校验；删除 `ResetExCookieAsync`；igneous 选填，无值时回退表站 |
| `ExClient/Internal/MyHttpClient.cs` | 移除自动重置 ex cookie 的调用，避免掩盖错误 |
| `ExClient/Strings/{en,zh-hans,zh-hant,ko}/Resources.resjson` | 无新增键（igneous 失效时直接回退表站） |
| `ExViewer/Views/LogOnDialog.xaml` | igneous 正则放宽；`WebView` → `muxc:WebView2` |
| `ExViewer/Views/LogOnDialog.xaml.cs` | 新校验、错误映射、WebView2 初始化/导航完成/cookie 读取、保活与无 igneous 回退 |
| `ExViewer/Views/SplashControl.xaml.cs` | `NeedLogOn` 判定前应用设置，使 `Host` 正确 |
| `ExViewer/Strings/{en,zh-hans,zh-hant,ko}/Resources.resJson` | igneous 提示文案、`NetworkError`、`WebViewConnectionError`、`IgneousNotAvailable`、`NoIgneousFallback` |

### 待验证（M4）

- [x] CI 四架构构建通过（x86 / x64 / arm / arm64）
- [x] WebView2 能否在 `ContentDialog` 中正常渲染（airspace 风险）
- [x] 网页登录后能否从 WebView2 拿到 igneous（法国节点实测有效）
- [ ] T1–T10 全部边界用例（核心路径已由用户双网络实测通过）

### CI 结果（2026-10-18）

- 首次失败与代码无关：`windows-latest` 已升级到 VS 18，UWP 工具链缺失，连未改动的项目都报 `Windows.*` 命名空间不存在。
- 已在 `.github/workflows/ci.yml` 将 `runs-on` 固定为 `windows-2022`。
- 修正 `CoreWebView2WebMessageReceivedEventArgs` 的 API 用法（`TryGetWebMessageAsString()`）后，
  run `39114641150` 四架构全部通过。

### 运行时验证（2026-10-18，2024.11.3.1）

测试方式：下载 CI 的 x64 产物，导入测试证书后安装；`igneous` 使用用户提供的测试 cookie。

| 用例 | 结果 | 说明 |
| --- | --- | --- |
| T2 Cookie 登录（EH + igneous 一并填写） | 通过 | 登录对话框关闭，主界面正常加载；cookie 同时写入两域 |
| 里站访问（设置中开启 ExHentai） | 通过 | “热门”页成功加载 exhentai 画廊列表（标题、页数、上传者、时间） |
| 首启默认模式 | 通过 | 新安装后登录框默认显示 Cookie 表单，不再先初始化 WebView2 |
| M2 网页登录 | 通过（端到端） | 用户实测第二次登录成功并进入；首次因在登录页停留过久出现 `ERR_CONNECTION_CLOSED`，已加 60s 保活 |

补充观察：

- 2024.11.2.0 版本上，登录成功后立即打开设置出现过一次 `0xc0000409` fail-fast；后续版本未复现。
  已增加 `WebView2.Close()` 清理，继续观察是否与 WebView2 销毁有关。
- 现象与代理/TUN 分流高度相关：应用自身的 `Windows.Web.Http` 和终端里的 `curl` 都能访问目标站点，
  但 WebView2 的独立网络栈可能没有走同一条代理路径（或独立 DNS/DoH 影响分流）。
  按用户要求，不再检查代理配置本身；此项作为环境前置条件记录。
- 网页登录已不是默认路径，且失败时会显示可读错误；Cookie 登录不受影响。
- 2024.11.3.2 增加 `privateNetworkClientServer` 并重测：在 TUN 模式下 WebView2 仍返回 `Unknown (0)`；
  切换为系统代理后可以到达 Cloudflare 页面。
- 2024.11.3.3 修正为忽略 Cloudflare 挑战产生的瞬态导航失败（`Unknown` / `OperationCanceled` /
  `ConnectionAborted` / `ConnectionReset`），不再用错误页覆盖认证页。窗口置于前台后，
  Cloudflare 安全验证自动通过，成功加载 E-Hentai Forums 登录表单。
- 注意：窗口最小化或处于后台时，WebView2 可能被节流，挑战会长时间停在“正在进行安全验证”；
  保持登录窗口在前台即可。
- 用户实测：第一次在登录页停留过久，提交时出现 `ERR_CONNECTION_CLOSED`；第二次重新打开后登录成功并进入。
- 2024.11.3.4 增加网页登录保活：表单加载后每 60 秒执行一次同源 `HEAD /`，
  避免服务端关闭空闲连接；切换 Cookie 模式或关闭对话框时停止。
- 2024.11.3.5 修复“表单填到一半消失”：仅当导航离开 `act=Login` 页面后才尝试读取 cookie 自动登录；
  登录失败/缺少 igneous 时不再用错误页覆盖正在填写的表单。
- 2024.11.3.6：网页登录成功后，若拿不到有效 `igneous`，先复用同账号已有的 igneous；
  仍没有则自动切到 Cookie 表单，预填 `ipb_member_id` / `ipb_pass_hash` 并提示补充 igneous。
- 服务端行为确认：仅带 `ipb_*` 访问里站只会返回 `igneous=mystery`（本地解密 WebView2 cookie 也确认长度为 7），
  不会自行下发有效 igneous；有效值必须来自已有浏览器会话。
- 2024.11.3.7：igneous 恢复为选填。没有 igneous 仍可登录表站；登录成功后若 `VisitEx` 开启但无有效 igneous，
  自动关闭 `VisitEx` 并提示。网页登录无 igneous 时预填会员号/passhash，用户可直接登录表站或补填 igneous。
- 2026-10-18 用户在新加坡（无里站）与法国（可访问里站）两种网络下实测 2024.11.3.7：
  网页登录、Cookie 登录、无 igneous 回退表站、里站访问均符合预期。
- 2024.11.3.8（打包前复查）：修复同账号 igneous 复用因提前清空 cookie 而失效；失效 igneous 不再阻断登录，
  而是清除后回退表站；网页登录消息校验来源、导航处理增加异常边界、cookie 读取防重入；
  首次启动与已登录启动时若 igneous 失效，都会回退表站，提示延迟到主界面显示。
- 2026.10.19.0：正式发布版本，代码与 2024.11.3.8 一致；CI 增加 Release 写权限，
  未配置 AppCenter 时跳过符号上传，tag 构建可直接产出四架构 Release。
- 2026.10.19.1：修复 CI 自签名证书缺少 Basic Constraints（`2.5.29.19`）导致 `Install.ps1`
  报「certificate is missing the basic constraints extension」；改用 CA:true + Code Signing 的自签名证书。
- 测试期间多次卸载重装，应用本地设置/历史被清空；最终已用测试 cookie 重新登录。

下一步：

1. 在网页登录表单中输入论坛账号密码，确认登录成功后 cookie 能被自动读取并关闭对话框。
2. 保持登录窗口在前台，避免 WebView2 被后台节流导致 Cloudflare 验证长时间不通过。
3. 继续执行 T1、T3–T10 真机回归；Cookie 登录始终可作为主路径。

### 决策日志

| 日期 | 决策 | 原因 |
| --- | --- | --- |
| 2026-10-18 | 优先修 Cookie 登录，网页登录后置 | Cookie 登录是唯一不依赖 Cloudflare 挑战的可靠路径 |
| 2026-10-18 | igneous 改为选填，缺省回退表站 | 原程序 igneous 就是选填；硬性要求会阻断登录 |
| 2026-10-18 | #182 暂不实现 | 与 WinUI 2.8 / WebView2 冲突，目标系统已停止支持 |
| 2026-10-18 | 登录框默认 Cookie 模式 | 网页登录受 Cloudflare/代理环境影响，Cookie 登录已验证可靠 |
| 2026-10-18 | 不再检查用户代理配置 | 代理属于敏感内容；只记录现象与前置条件 |
