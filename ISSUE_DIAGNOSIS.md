# QR 扫码登录无法完成 — 代码层面根因定位与修复

## 结论（先说重点）

**根因找到了，是后端代码 bug，不是网络/配置/风控问题。**

`backend/services/telegram/login_qr.py` 的 `get_qr_login_status` 方法在 `scanned_wait_confirm`（已扫码待确认）状态下，有一个**错误的 fallback 逻辑**：会再次调用 `ExportLoginToken`。这会生成一个**新的登录 token**，直接导致用户手机端已扫码的确认流程失效（旧 token 作废），并且每次前端轮询（3 秒一次）都会触发 DC 迁移、反复创建新 auth key——这完美对应了日志里反复出现的 `Start creating a new auth key on DC5`。

**修复已完成**，详见下方。

---

## 根因详解

### Telegram QR 登录的正确协议流程

```
1. ExportLoginToken          → 拿到初始 token，生成二维码
2. 用户扫码 + 手机点确认       → 服务端推 UpdateLoginToken
3. ImportLoginToken(token)    → LoginTokenSuccess（确认完成）/ LoginToken（未确认，刷新 token）
```

`ExportLoginToken` **只在开始时调用一次**。后续轮询只应使用 `ImportLoginToken`。

### Bug 代码位置

`backend/services/telegram/login_qr.py`，`get_qr_login_status` 方法的 `scanned_wait_confirm` 分支（修复前约 780-796 行）：

```python
# 先 ImportLoginToken 轮询扫码确认（正确）
if data.get("token"):
    result, _ = await self._import_login_token(...)

if isinstance(result, raw.types.auth.LoginTokenSuccess):
    return await self._finalize_qr_login(...)   # ← 正常完成路径
if isinstance(result, raw.types.auth.LoginToken):
    self._apply_login_token_update(data, result)
    data["status"] = "scanned_wait_confirm"

# ❌ BUG：fallback 又调用了 ExportLoginToken
last_export_ts = data.get("last_export_ts", 0)
if (result is None or isinstance(result, raw.types.auth.LoginToken)) and now - last_export_ts >= 3:
    data["last_export_ts"] = now
    try:
        outcome = await self._export_login_token(client, data)  # ← 生成新 token！
        ...
```

### 为什么这会导致登录永远无法完成

1. 用户扫码 → 后端收到 `UpdateLoginToken` → 状态变 `scanned_wait_confirm`
2. 前端每 3 秒轮询 `GET /qr/status`
3. 后端调 `ImportLoginToken` → 返回 `LoginToken`（用户还没点确认，正常）
4. **然后 fallback 又调 `ExportLoginToken`** → 服务端生成新 token，旧 token 作废
5. 如果新 token 触发 `LoginTokenMigrateTo`（中国区 home DC 是 DC5），又调 `get_session(client, DC5)` 创建新 auth key → **日志出现 `Start creating a new auth key on DC5`**
6. 用户在手机上点"确认" → 但 token 已被刷新，确认无效
7. 下一轮轮询重复步骤 3-6 → 死循环

### 为什么日志里没有报错

`_import_login_token` 和 `_export_login_token` 内部用 `except Exception: error = exc` / `return None` **静默吞掉了所有异常**，没有任何日志输出。这就是为什么日志只看到 pyrogram 的 INFO，看不到任何账号保存相关的成功/失败日志。

---

## 修复内容（4 个文件）

### 1. `backend/services/telegram/login_qr.py`（核心修复）

- **移除** `scanned_wait_confirm` 状态下的 `ExportLoginToken` fallback。已扫码确认阶段只应使用 `ImportLoginToken` 轮询，等待用户在手机端点确认后服务端返回 `LoginTokenSuccess`。
- 给 `_import_login_token` / `_export_login_token` 中被静默吞掉的异常增加 `logger.warning`，便于排查真实失败原因。

### 2. `frontend/src/components/accounts/AddAccountModal.vue`（前端体验修复）

- 新增 `scanned_wait_confirm` 状态的 UI 反馈：二维码变半透明 + 显示"已扫码，请在手机上点击确认登录"提示（带脉冲动画），避免用户误以为卡住。
- 修复 `handleSave`：在等待手机确认时（轮询运行中），点"确认保存"不再错误停止轮询，只提示用户去手机确认。此前停止轮询会直接中断登录流程。
- 获取新二维码时重置扫码状态。

### 3. `frontend/src/locales/{zh-CN,en-US}.json`

- 新增 `scannedWaitConfirm` 文案。

---

## 关于"共享公共 API 凭据"的排查结论

之前怀疑的"使用上游 tg-signer 公开默认 api_id/api_hash 导致 Telegram 风控"——**从代码层面看这不是根因**。核心 bug 是上面的 `ExportLoginToken` fallback。但仍然**建议配置自己的 TG_API_ID/TG_API_HASH**（从 https://my.telegram.org 申请），因为：

- 公开 api_id 确实可能被 Telegram 限流/风控，影响其他功能
- 修复后如果登录仍然异常，可排除凭据因素

---

## 关于 `AUTH_KEY_UNREGISTERED`（已解决）

之前清理死会话后 `AUTH_KEY_UNREGISTERED` 消失——这是对的。那个问题是**死会话残留**导致的，与本次 QR 登录 bug 无关。清理 session 文件 + 重置 accounts.json 的做法正确。

---

## 测试建议

1. 重新构建镜像（或本地 `docker compose up --build`）
2. 扫码登录，观察日志：
   - 应看到 `state=waiting_scan` → `state=scanned_wait_confirm`（只一次）
   - 不应再看到反复 `Start creating a new auth key on DC5`
   - 手机确认后应看到 `state=success` + 账号保存
3. 前端应显示"已扫码，请在手机上点击确认登录"提示
4. 如果 ImportLoginToken 有异常，现在会输出 `QR ImportLoginToken 异常` warning 日志，可据此进一步排查

---

## 待定位的遗留疑点（修复后观察）

1. `start_qr_login` 中第一次 `ExportLoginToken` 如果直接返回 `LoginTokenMigrateTo`（中国区用户常见），代码直接用 MigrateTo 的 token 生成二维码，没有先在目标 DC import 拿到正确的 LoginToken。当前流程能工作（扫码后在 ImportLoginToken 时完成迁移），但不是最优实现。如果修复后仍有问题，可考虑在 `start_qr_login` 里先处理 MigrateTo。
2. 修复后若 ImportLoginToken 持续返回 `LoginToken`（非 Success），且用户确实在手机上点了确认——则需要排查是否 api_id 风控。
