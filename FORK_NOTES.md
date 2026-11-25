# Fork 备忘（felix5572/hypersdk）

记录本 fork 与上游 `infinitefield/hypersdk` 的差异、已知 SDK 缺陷和待办。
消费方：`hyperliquid-trade` 根 `Cargo.toml` 的 `[patch.crates-io]`，使用本 fork 的 `main`，具体 commit 由 `Cargo.lock` 锁定。

## 维护策略（2026-09-30）

- 出于供应链安全考虑，**维护自己的 fork 并持续演进**，不直接依赖上游。上游的改动先审阅，再同步进来。
- 上游大概率不会接受的改动（例如给 `OrderStatus` 加 `#[serde(other)] Unknown` 兜底）不往 SDK 里加。

## 现状快照（2026-10-13）

| 位置 | commit | 说明 |
|---|---|---|
| fork `origin/main` = 本地 `main` | `791ce28` | 合并上游 v0.3.0（`db34e8d`）+ 保留 ScheduleCancel 补丁；`hyperliquid-trade` 的 `Cargo.lock` 锁在这里 |
| 上游 `main` | `db34e8d` | v0.3.0（2026-10-12） |

- fork 与上游的唯一代码差异：ScheduleCancel 清除时省略 `time`（见下节）。上游 0.3.0 仍未修。
- 2026-10-13 用户授权：本仓库的读写与 git 拉取、合并、提交、推送由助手自行处理；这次同步按用户要求不审供应链。
- 同步动机：HL 主网只优先带 `f`（fast）的撤单（文档仍写“未来升级”，区块数据已证实生效），上游 0.2.15 起的 `BatchCancel` / `BatchCancelCloid.fast` 直接用上游实现。
- `hyperliquid-trade` 侧签名对拍：`connector::tests::actions_sign_like_the_official_python_sdk`（fast 撤单、scheduleCancel 清除/设置、batchModify），参考值由 `trading-system-rust/scripts/offline/hl_signing_vectors.py` 用官方 Python SDK 生成。

## 供应链轻量检查（2026-10-13，同步到 v0.3.0 后，用户要求“不用太细致”）

结论：编进 `hyperliquid-trade` 的部分没发现可疑点。

- **依赖**：库 `Cargo.toml` 无新增依赖，只升版本 `base64 0.22 → 0.23`、`hex-literal 0.4 → 1`；无 `build.rs`。`hyperliquid-trade` 的 `Cargo.lock` 仅新增 `base64 0.23.1`、`hex-literal 0.4.1 → 1.1.0`，其余不变。
  - `base64 0.23.1`：2026-08-04 由维护者 marshallpierce 发布，下载约 4,500 万，未撤回。
  - `hex-literal 1.1.0`：2025-10-29 由 RustCrypto 的 newpavlov 发布，下载约 1,460 万，未撤回。
- **代码**：`21ed588..791ce28` 的 `src/` 新增约 6,800 行，扫描未见读环境变量、读写文件、执行外部命令、连接陌生主机；新增网址均为注释中的 HL 官方文档。签名由 `hyperliquid-trade` 的 Python SDK 逐字节对拍测试覆盖。
- **提交者**：维护者 Dario 45 个、vignesh-chaturvedi 11 个；8 位一次性贡献者的改动都很小且与说明一致（价格精度返回 None、市价单改 FrontendMarket、逐仓保证金可减、资金费预测接受 null、PerpMarket.delisted、WS 关闭帧日志、deployerFeeScale、AI SKILL 文档）。
- **不编译但需留意**：`skills/` 下给 AI 助手的文档，`hyperliquid-trading` 推荐 `curl … | sh` 安装 hypecli；`hyperliquid-payments` 引导用户去第三方网站 `https://cctp.to` 跨链充值 USDC（2026-01-30 旧提交 `140b7a4`，非本次引入）。在本仓库让 AI 工作时，不要照这些文档推荐资金操作。
- **未覆盖**：未逐行审完全部新增代码；未审 alloy 等本次未变的间接依赖；hypecli（Trezor、iroh 等）未看。

## SDK 缺陷：`ScheduleCancel { time: None }` 签名不对（fork 已修，上游未修）

- **API 语义**：官方 Exchange endpoint 文档 "Schedule cancel (dead man's switch)" 一节写的是 `"time": number (optional)`，原文 "Not including time will remove the scheduled cancel operation."，也就是说清除的方式是**不带** `time` 这个字段。
- **SDK 行为**：结构体写作 `ScheduleCancel { time: Option<u64> }`。`None` 会被序列化成 `"time": null` 参与 msgpack 签名，而交易所计算哈希时这个字段不存在，两边字节不一致，签名还原出另一个地址。报错是 `User or API Wallet ... does not exist`。
- **为什么别人很少遇到**：上游公开接口 `HttpClient::schedule_cancel(when)`（`http.rs`）只接受一个具体时间，内部永远写 `Some(time)`，SDK 没有提供清除接口。只有自己构造 `ScheduleCancel { time: None }` 去清除的调用方才会遇到。
- **我们的用法**：`hyperliquid-trade` 的 `connector.rs::schedule_cancel_at(Option<u64>)` 自己构造 action 并签名，设置和清除共用这一个函数。清除调用发生在停机解除 DMS 时（`hyperliquid_account.rs::shutdown` → `clear_schedule_cancel`）。
- **不修的后果**：DMS 清除失败，deadline 仍然有效，到点会撤掉账户所有挂单（这一侧是保守的）；停机报告会带 `HL DMS: ...` 错误。
- **fork 修法**（`20dac1a`）：给 `time` 加上 `#[serde(default, skip_serializing_if = "Option::is_none")]`，`None` 时省略这个字段。

### 待办（另案处理）

- [ ] 给上游提 PR：上面这个 `skip_serializing_if`，外加一个公开的 `clear_schedule_cancel()` 接口；理由引用上面的 API 文档原文，附一条签名字节测试（`None` 时 msgpack 中没有 `time` 键）。
- [ ] 上游合并后，本 fork 的这个差异可以撤掉。

## 同步上游：2026-09-30 的评估（已于 2026-10-13 完成同步，保留作记录）

上游比 fork 多 50 个提交（约 +12k 行）。其中和 `hyperliquid-trade` 相关、需要逐一审阅的有：

- **Breaking（编译期就会暴露）**：
  - `4de4f17`：`BatchCancel` / `BatchCancelCloid` 新增 `fast` 字段；`reserve_request_weight` 新增 destination 参数。
  - `028e4a3`：`BatchModify` 新增 `always_place`；`Subscription::WebData2` / `Incoming::WebData2` 被删除；`SpotDeployAction::Outcome` 改为 `Action::OutcomeDeploy`。
- **签名相关**：
  - `68a0860`：带地址的 L1 action 改为按小写 hex 签名。原来按 20 字节签名，会被拒并报 "User or API Wallet does not exist"。
  - `94644af`：撤单签名字节在 `fast` 标志下保持不变，有测试锁定。
- **可能有用**：`028e4a3` 新增 `Incoming::Error`，订阅失败的错误帧从此能被收到。这正好对应 `hyperliquid-trade` 待议的"HL 用户流订阅确认"一项。
- **其他**：`034d4b9` 市价单改为 FrontendMarket；`0ad6978` 依赖整体升级（传递依赖变动需要按供应链风险审阅 `Cargo.lock`）。
- **硬前置**：上游 MSRV 是 **1.94.1**（edition 2024），本机 rustc 是 1.91.1。凡是要编译的机器（本地、trade、bm）都得先升级工具链，部署顺序不能反。
- 其余大多是 HIP-3/HIP-4 部署动作、hypecli、Trezor 签名，与我们无关。

同步时的建议步骤：先 fetch 并合并上游，保留 ScheduleCancel 补丁 → 审阅 `http.rs` / `signing.rs` / `types` 以及 `Cargo.lock` 的变动 → 推送 fork → 在 `hyperliquid-trade` 里执行 `cargo update -p hypersdk`，修编译问题、跑全量测试、送审。
