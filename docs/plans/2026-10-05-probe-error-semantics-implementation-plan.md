# quota-watch 探测错误语义化与轮询韧性实施计划（v0.1.22）

## 1. 目标

本轮解决 2026-10-05「Copilot 数据可能已过期: HTTP 401」误诊事故暴露的三个产品缺陷：

1. **A1 错误分类**：401/403 凭证类失败与普通失败共用「数据可能已过期」文案，直接导致当天的误诊排查；
2. **A3 无观测性**：探测失败不落任何日志，journal 排查为零线索；
3. **B1 冷启动重试无上限**：特定状态组合下以 5s 间隔无限快打上游 API。

> 评审修订（R1-1）：原第 4 项「A2 页脚更新时间失真」前提失实——仓库从未存在可见的「更新于」展示；`[data-role="updated"]` 行在数据渲染后整体隐藏是**既有设计**（src/client.mjs `renderAll`：有快照时间即 `hidden=true`，无时间显 Loading，传输失败显 requestFailed/failed；tests/client.test.mjs 既有断言「数据渲染后该行隐藏」）。「数据年龄可视化」改判顺延，与 B2 合并，见 §2。

设计来源：2026-10-05 会话诊断（know-how 017 §8）与本仓库优化提案讨论；本计划即评审物，获批后由普通编码 agent 或人工执行。

## 2. 范围

**本轮做**：A1 + A3 + B1（含并入 A1 的 cause.code 细化——与 A1 触碰同一函数，拆开发版无意义）。

**本轮明确不做**（评审者可改判，但默认顺延到下一版）：

- **A2 数据年龄可视化**（评审 R1-1 改判：原「页脚更新时间失真」缺陷前提不成立；「新增数据时间展示」属新 UI 特性而非缺陷修复，与 B2 合并为「last-good 时限 + 数据年龄可视化」一并设计）；
- B2 last-good 时限渲染（stale 数据超龄显示「--」，与 A2 合并处理）；
- C2 `parseCopilotUser` 结构变更细化文案；
- C3 打开面板时按数据年龄自动 refresh；
- glm usage 子状态的 auth 分型文案（provider 级已覆盖 GLM 密钥被拒场景；`usage.errorKind` 本轮仅作 wire 预埋，见 Task 1）；
- status='error' 行「查询失败: …」与 Copilot 详情 quota 缺失裸 `copy.stale` 的 auth 分型（评审 R1-7 显式裁决：本轮排除，随上述文案轮一并处理）。

## 3. 全局约束

- **版本锚点不可破坏**：know-how 017 §5/§7 以 `coldRetryMs`（src/host.mjs）与 `retryNoDataMs`（lib/client.js 构建产物，源在 src/client.mjs）作为版本判据 grep 锚点，两个标识符必须原样保留。
- **新增零依赖**：只用 node:test / 现有 esbuild 管线；`engines.node >=20`、`dsh >=0.1.7-rc.2` 不变。
- **双语文案同步**：zh/en 两个 copy 对象必须同时新增键（仓库既有模式）。
- **客户端测试断言用英文文案**（tests/client.test.mjs 的 jsdom 环境为 en locale，既有断言先例 `Data may be stale: HTTP 503`）。
- **发版纪律**：tag 推送（触发 OIDC publish）前必须经主人确认；npm staged 延迟属正常（know-how 018 §8）。
- **无额外全局约束**（版本下限、平台要求如上，已逐字继承）。

## 4. 架构快照（仅本次方案）

数据流不变：宿主每轮 `refresh()` 并发探测 GLM+Copilot → `view()` 快照 → 客户端 30s 轮询渲染。本轮改四个点：

1. `classifyProbeError(error)` 新函数：把探测异常归类为 `{ message, kind }`，`kind ∈ 'auth' | 'server' | 'network' | 'parse'`。`errorKind` 随 `error` 一起进入 view model（`providers[].errorKind`；glm `usage.errorKind` 为本轮 wire 预埋，消费方在下一版 usage 文案分型）。
2. 客户端按 `errorKind === 'auth'` 切换横幅与胶囊/概览行 ⚠ tooltip 的前缀文案（`copy.authRejected`），原始错误串保留在后；分型覆盖 `glmRowModel` 与 `copilotRowModel` 两条行模型链。
3. `schedule()` 冷启动重试改指数退避：`min(coldRetryMs × 2^(n-1), pollIntervalSec×1000)`，非 retryable 轮复位计数。

## 5. 文件结构与职责

| 文件 | 动作 | 职责 |
|---|---|---|
| `src/host.mjs` | 修改 | A1 `classifyProbeError` + `errorKind` 贯通；A3 转移日志；B1 退避 |
| `src/client.mjs` | 修改 | A1 auth 文案分型（横幅 + 胶囊/概览行 tooltip；glm 与 copilot 两条行模型） |
| `tests/host.test.mjs` | 修改 | Task 1/2/3 的新断言 |
| `tests/client.test.mjs` | 修改 | Task 4 的新断言 |
| `package.json` | 修改 | `version: 0.1.22` |
| `lib/client.js` | 构建产物 | `npm run build` 生成，不手改 |

边界：`src/core/adapters.mjs`（解析/构造请求）本轮**不动**；`cordis.patch.yml` 不动。

## 6. 任务清单

### Task 1 — host：`classifyProbeError` + `errorKind` 贯通（A1+C1）

**文件**：`src/host.mjs`、`tests/host.test.mjs`

**接口契约（Produces，后续任务引用）**：

```js
// src/host.mjs 导出（供测试直测；宿主内部经 markFailure/markGlmUsageFailure 使用）
export function classifyProbeError(error) {
  // 返回 { message: string, kind: 'auth' | 'server' | 'network' | 'parse' }
  // 判定顺序：
  //   1. error.name === 'TimeoutError' | 'AbortError'   → { message: '请求超时', kind: 'network' }
  //   2. Number.isInteger(error.status) && (401|403)    → { message: `HTTP ${status}`, kind: 'auth' }
  //   3. Number.isInteger(error.status)                 → { message: `HTTP ${status}`, kind: 'server' }
  //   4. error.code === 'UNRECOGNIZED_RESPONSE'         → { message: '服务商响应格式无法识别', kind: 'parse' }
  //   5. error.cause.code 匹配 /^[A-Z0-9_]{3,60}$/      → { message: `探测失败 (${code})`, kind: 'network' }
  //   6. 兜底                                           → { message: '探测失败', kind: 'network' }
}
```

- view model：`providers[]` 增加 `...(provider.errorKind !== undefined ? { errorKind: provider.errorKind } : {})`；glm 的 `usage` 对象同理可带 `errorKind`（**wire 预埋**：客户端本轮不消费，下一版 usage 文案分型时启用）。
- `markFailure` / `markGlmUsageFailure` 改用 `classifyProbeError(error)` 取 `{ message, kind }`，存 `error` 与 `errorKind`。
- 原 `safeProbeError` 删除（唯一调用方就是上述两处；`请求超时`/`HTTP ${status}` 语义由 classify 承接）。

**步骤**：

1. 在 `tests/host.test.mjs` 追加直测：401→`{message:'HTTP 401',kind:'auth'}`；503→server；`Object.assign(new Error(),{name:'TimeoutError'})`→network/请求超时；`{code:'UNRECOGNIZED_RESPONSE'}`→parse；`Error('fetch failed',{cause:{code:'ECONNRESET'}})`→`探测失败 (ECONNRESET)`/network；裸 Error→network/探测失败。
2. 追加行为断言：mock `fetchImpl` 恒 401 → `service.overview()` 中 copilot provider `error:'HTTP 401'` 且 `errorKind:'auth'`。
3. 运行确认新断言失败（红）。
4. 按**接口契约**实现 `classifyProbeError`，改写 `markFailure`/`markGlmUsageFailure`/view；删除 `safeProbeError`。
5. 运行确认全绿。

**验证**：`node --test tests/host.test.mjs` → 全部通过（含既有用例零回归）。

### Task 2 — host：状态转移日志（A3）

**文件**：`src/host.mjs`、`tests/host.test.mjs`

**Consumes**：Task 1 的 `classifyProbeError`（日志 detail 用其 `message`）。
**Produces**：日志格式契约 `quota-watch: <key> <from>→<to> (<detail>)`，其中 `<key>` ∈ `glm | copilot | glm-usage`；`<from>`/`<to>` 一律取状态机真实值——provider 初始状态是 `loading`（src/host.mjs `initialProvider`），**不做 init 特判**；`glm-usage` 的 from 取旧 `usage.status`，无旧 usage 时记 `loading`。级别：→`error`/→`stale` 走 `ctx.logger?.warn?.`；→`ready`/→`missing` 走 `ctx.logger?.info?.`（缺凭据非劣化）；`from === to` 不打；`ctx.logger` 缺失或方法缺失时静默（可选依赖，与 copilot-auth 同模式）。

**步骤**：

1. 扩展 `makeContext()` 工厂（tests/host.test.mjs:19）接受可选 `logger` stub（收集 warn/info 调用参数的数组对象）。追加断言：首轮 200 → info 含 `copilot loading→ready`（初始状态为 loading，非 init）；下一轮 401 → warn 含 `copilot ready→stale (HTTP 401)`；再恢复 200 → info 含 `copilot stale→ready`；同状态连续轮不打日志。
2. 运行确认失败（红）。
3. 实现：`createQuotaService` 内加 `logTransition(key, from, to, detail)` 辅助；在 `markMissing`/`markSuccess`/`markFailure`/`markGlmPlanSuccess`/`markGlmUsageSuccess`/`markGlmUsageFailure` 六处 set 前捕获旧状态、set 后调用。detail 取 classify 的 message（usage 级用 usage 错误消息）。
4. 运行确认全绿。

**验证**：`node --test tests/host.test.mjs` → 全部通过。

### Task 3 — host：冷启动重试指数退避（B1）

**文件**：`src/host.mjs`、`tests/host.test.mjs`

**Consumes**：无（独立于 Task 1/2 的逻辑，但同文件，按序执行避免冲突）。
**Produces**：调度契约——retryable 轮的延迟序列 `coldRetryMs × 2^(n-1)` 封顶 `pollIntervalSec×1000`（默认 5s,10s,20s,40s,60s,60s…）；任一非 retryable 轮（含成功、全 missing）计数归零，其后 retryable 从 `coldRetryMs` 重新起步。`coldRetryMs` 标识符与语义（首轮延迟）**不变**（know-how 版本锚点）。

**步骤**：

1. `tests/host.test.mjs` 用既有 mock timers 模式（`t.mock.timers.enable({ apis: ['setTimeout'] })`）做三件事：
   - ① 新用例：mock fetch 恒失败 + 双 provider 均非 ready/stale → tick 触发延迟序列 5s/10s/20s/40s/60s/60s（封顶保持）；
   - ② **适配既有用例** `cold start probes immediately, retries fast while failing…`（tests/host.test.mjs:233-271）：其 tick 编排按恒 5s 设计（第二个 `tick(5_000)` 期望恢复轮 `attempts===7`），退避后第二轮失败改排 10s——把该用例恢复轮前的 tick 由 5s 改为 10s，其余断言不动；此适配是本任务的**预期改动**，不算回归；
   - ③ 复位用例（可达路径，**须有判别力**）：前置 1-2 个全失败轮把 coldAttempts 抬到 ≥2（此时 retryable 延迟已 ≥20s），随后一轮成功（glm ready，计数应复位），再令 glm 凭据解析失败转 `missing`（可变凭据变量 + 覆写 resolve，参照 tests/host.test.mjs:66-72 rotated-credentials 模式；**不得**在构造期传 `glmKey: ''`——那会让 glm 第 1 轮即 missing），copilot 持续 401（error、无 last-good）→ 状态 `[missing, error]` 重新满足 retryable → 断言延迟从 5s 重新起步（若「非 retryable 轮归零」实现缺失，此处将是 ≥20s，用例即红）。不可达场景不测：任一 provider 成功过一次后，其失败必走 last-good→`stale`（src/host.mjs `markFailure`），`every(!ready && !stale)` 从此永假，退避序列不会再启动。
2. 运行确认失败（红，当前恒 5s）。
3. 实现：`createQuotaService` 加 `let coldAttempts = 0`；`schedule()` 回调内 `retryable ? coldAttempts += 1 : coldAttempts = 0`；延迟 `retryable ? Math.min((options.coldRetryMs ?? 5_000) * 2 ** (coldAttempts - 1), pollIntervalSec * 1000) : pollIntervalSec * 1000`。
4. 运行确认全绿（首轮 5s 语义不变；既有冷启动用例的 tick 编排已按步骤 1② 适配为退避序列——属**预期改动**而非回归）。

**验证**：`node --test tests/host.test.mjs` → 全部通过。

### Task 4 — client：auth 类错误文案分型（A1 客户端半区）

**文件**：`src/client.mjs`、`tests/client.test.mjs`

**Consumes**：Task 1 的 wire 契约——`providers[].errorKind`，值域 `'auth' | 'server' | 'network' | 'parse'`，仅伴随 `error` 出现。fixture 工厂（`tests/client.test.mjs` 的 `copilotProvider(overrides)` / `glmProvider(overrides)`）直接传 `errorKind: 'auth'` 即可。
**Produces**：copy 新键 `authRejected`——zh `凭证已被服务商拒绝，请重新授权`、en `Credential rejected by provider — re-authorize`。

渲染规则（替换现两处）：

- 详情横幅（src/client.mjs `dqw-error` 行，现 `` `${copy.stale}${provider.error ? `: ${provider.error}` : ''}` ``）：`errorKind === 'auth'` 时 `` `${copy.authRejected}${provider.error ? `: ${provider.error}` : ''}` ``，否则维持原文案。
- 胶囊/概览行 ⚠ tooltip（`renderProvider` 内现 `mark.title = model.error ?? copy.stale`；tooltip 消费的是行模型字段）：`glmRowModel` 与 `copilotRowModel` 的返回值都透传 `errorKind`（data 与 error 两个分支都要）；auth 时 `` `${copy.authRejected}: ${model.error}` ``，否则维持原文。

**步骤**：

1. `tests/client.test.mjs` 追加：① stale copilot 带 `error:'HTTP 401', errorKind:'auth'` → 面板 `.dqw-error` 文本 = `Credential rejected by provider — re-authorize: HTTP 401`；② stale copilot 带 `error:'HTTP 503'`（无 errorKind）→ 维持 `Data may be stale: HTTP 503`；③ 胶囊 ⚠ title 同步断言（auth 与非 auth 各一，copilot）；④ glm stale 带 `error:'HTTP 401', errorKind:'auth'` → 概览行 ⚠ title 同为 auth 文案（覆盖 glmRowModel 链）。
2. 运行确认失败（红）。
3. 实现：zh/en copy 各加 `authRejected`；两处渲染点按上述规则分支；`glmRowModel` 与 `copilotRowModel` 的返回值都透传 `errorKind`。
4. 运行确认全绿。

**验证**：`node --test tests/client.test.mjs` → 全部通过。

### Task 5 — 构建、全量验证、版本 0.1.22、提交（停点：不 tag）

**文件**：`package.json`、`lib/client.js`（构建产物）

**步骤**：

1. `npm test`（= `node --test` 全量）→ 全绿，记录用例数（基线 121，评审 Agent 2026-10-05 实测；本轮净增 ≥8）。
2. `npm run build` → `lib/client.js` 再生；`grep -c retryNoDataMs lib/client.js` ≥1 且 `grep -c coldRetryMs src/host.mjs` ≥1（know-how 版本锚点完整性）。
3. `npm run security:check` → 通过。
4. `package.json` `version` → `0.1.22`。
5. 单条 commit（仓库惯例 `type(scope): 摘要 (v0.1.22)`），建议：`feat(host,client): classify probe errors (auth/server/network/parse), auth-aware copy, transition logs, capped cold-retry backoff (v0.1.22)`；本计划文档（docs/plans/2026-10-05-probe-error-semantics-implementation-plan.md）随本 commit 一并收编——docs/plans 既有计划均入库，npm `files` 不含 docs，不影响发布物。commit 前 `git status` 确认除本计划文档外无其他未跟踪文件混入。

**验证**：`npm test && npm run build && npm run security:check` 三连全绿；`git log --oneline -1` 显示 v0.1.22 commit；`git status` 干净。

**🔴 停点**：到此为止，不 push、不 tag。发版由主人确认后进入 Task 6。

### Task 6 — 发版、装机与升级后核对（主人确认后执行）

**前置**：主人明确同意发版。

**步骤**：

1. `git push` + `git tag v0.1.22 && git push origin v0.1.22` → GitHub Actions OIDC 发布。**注意 know-how 018：workflow 绿 ≠ 已上架**，用 `npm view @iasiv5/dsh-quota-watch version` 轮询至 `0.1.22`（staged 延迟可达 ~1h，勿重推 tag）。
2. 装机：产品内（dsh-m）升级 web profile；desktop profile 走产品内升级（know-how 020 §8.4，窗口内自研包直接升级）。
3. 升级后核对（know-how 017 §5 判据）：`grep -c attachShadow ~/.dsh/profiles/web/node_modules/@iasiv5/dsh-quota-watch/lib/client.js` ≥1；`grep -c retryNoDataMs ~/.dsh/profiles/web/node_modules/@iasiv5/dsh-quota-watch/lib/client.js` ≥1；`grep -c errorKind ~/.dsh/profiles/web/node_modules/@iasiv5/dsh-quota-watch/lib/client.js` ≥1（新版本标记）。
4. 行为抽查：`curl -s http://127.0.0.1:3080/api/dsh-quota-watch/overview`（本机 loopback 走 access.mjs 放行）确认 providers[] 可含 `errorKind`；若宿主会话门禁拦截则改浏览器 console `fetch('/api/dsh-quota-watch/overview').then(r=>r.json())`。auth 新文案的 UI 抽查需真实 401 才会出现——不强求当场验证（单元测试已覆盖渲染分支）；转移日志核对见下一步。
5. journal 核对：`sudo journalctl -u deepseek-harness.service -g 'quota-watch:' -n 20` 能看到转移日志（或至少无异常刷屏）。
6. 文档收尾：更新 know-how `017-quota-watch-style-strip-selfheal.md`（§8 补 0.1.22 复发处置指向新文案；版本判据补 `errorKind` 标记；**§7 冷启动叙事同步为退避序列**——原文「以 5s 短间隔重试直至拿到数据」在 B1 落地后过时）+ 索引表 017 行同步；desktop 机核对延后到下次使用时顺带。

**验证**：npm 查得 0.1.22；profile grep 三锚点全过；胶囊/面板正常在装、journal 无异常刷屏；know-how 已更新。

## 7. 执行纪律

- 开始实现前先批判性复查本计划；发现缺项/矛盾/命令无效，先修计划再动手。
- 本仓库惯例直接在 `main` 提交（历史核实：单线 release commit）；计划获批即为在 main 实施的同意。
- 按任务顺序执行（1→6），不无声跳步、不合并任务目标；每个任务完成即跑该任务验证。
- 测试失败先判测试错还是实现错，不许为过测试改断言语义。
- 遇阻塞或计划与仓库现实不符（锚点找不到、命令不存在），立即停下说明，不猜。
- 全部完成后运行 Task 5 的三连验证并输出修改摘要。

## 8. 最终验证

```
npm test            # 全量用例全绿
npm run build       # lib/client.js 与 src 同步
npm run security:check
grep -c coldRetryMs src/host.mjs        # ≥1（版本锚点）
grep -c retryNoDataMs src/client.mjs    # ≥1（版本锚点）
grep -c errorKind src/host.mjs src/client.mjs   # 各 ≥1（新能力落位）
git status              # 干净，仅 v0.1.22 commit 待推/已推（视 Task 6 进度）
```

装机侧验证见 Task 6 第 3–5 步。
