# 轻播 vela 授权服务端

PHP 8 + MySQL 实现的收费闭环：**用户在 App 里扫码付款并填邮箱 → 服务端落库 → 人工核对后签发 → 邮件发激活码 → 客户端激活**。

不依赖 Composer 与任何第三方库，放到任意支持 PHP 8 的虚拟主机即可。

## 1. 目录结构

```
server/
├─ api/
│  ├─ payment-notify.php    # 用户付款申报（App 调用）
│  └─ verify.php            # 激活 / 联网校验（App 调用）
├─ android/                 # 后台（伪装目录名，登录 + CSRF + 密码哈希）
│  ├─ _bootstrap.php        # 会话、鉴权、CSRF、布局
│  ├─ login.php  logout.php
│  ├─ index.php             # 仪表盘
│  ├─ pending.php           # 待处理订单（未处理 / 已签发 / 已拒绝）
│  ├─ issue.php             # 签发 / 续期 + 发邮件
│  └─ licenses.php          # 已签发（搜索、重发、作废）
├─ _lib/
│  ├─ config.php            # 配置读取（env > config.local.php > 默认值）
│  ├─ db.php  helpers.php
│  ├─ license.php           # 激活码 / 授权文件算法（必须与客户端一致）
│  └─ mail.php              # fsockopen SMTP + 邮件模板
├─ tools/
│  ├─ vector.php            # 与 Dart 的算法自检
│  └─ reset_admin.php       # 重置后台密码（CLI）
├─ install.php              # 建表 + 初始化管理员
└─ config.local.example.php # 配置模板
```

## 2. 部署步骤

1. **上传**：把 `server/` 整目录传到 web 根目录（例如 `/www/vela/server/`），
   最终对外地址是 `https://你的域名/server/api/...`；
   如果想让接口落在 `/api/...`，就把 `server/` 的内容直接放到 web 根。
2. **配置**：复制 `config.local.example.php` 为 `config.local.php`，至少填写：
   - `db_*`：MySQL 连接信息（先建好库，字符集 `utf8mb4`）
   - `license_secret`：32 位以上随机串（`openssl rand -hex 32`），**必须与打 APK 时的
     `--dart-define=VELA_LICENSE_SECRET` 完全一致**
   - `smtp_pass`：163 邮箱的「客户端授权码」（不是登录密码）
   - `site_url`：`https://你的域名`
3. **建表**：命令行 `php install.php`（推荐），或设置 `install_token` 后访问
   `install.php?token=你的口令`。安装会建表并创建管理员，**只显示一次初始密码**。
4. **登录后台**：`/android/login.php`（刻意用了 `android` 这种不起眼的目录名，
   扫描器一般不会去试探；想更隐蔽可以把目录再改个名）。
   忘记密码用 `php tools/reset_admin.php <用户名>`。
5. **删掉 `install.php`**（锁文件已生效，但删掉更稳妥）。
6. **自检**：
   ```bash
   php tools/vector.php                     # 与客户端算法对齐
   curl -s https://你的域名/api/verify.php  # 应返回 405 JSON，说明 PHP 正常
   ```

## 3. 配置项（环境变量 / config.local.php 二选一）

| 环境变量 | 说明 | 默认 |
| --- | --- | --- |
| `VELA_DB_HOST/PORT/NAME/USER/PASS` | 数据库 | localhost / 3306 / 空 |
| `VELA_LICENSE_SECRET` | 授权密钥，必须与客户端一致 | 空（不配置无法签发） |
| `VELA_PRICE_CENTS` | 价格表 JSON，如 `{"H6":1800,"Y1":2800,"LIFE":6800}` | ¥18 / ¥28 / ¥68 |
| `VELA_SMTP_HOST/PORT/SECURE/USER/PASS` | 邮件 | smtp.163.com / 465 / ssl |
| `VELA_SMTP_FROM_NAME` | 发件人显示名 | 轻播 vela |
| `VELA_ADMIN_EMAIL` | 新订单通知收件人 | xtc_cn@163.com |
| `VELA_SUPPORT_EMAIL` | 邮件 Reply-To | xtc_cn@163.com |
| `VELA_SITE_URL` | 站点地址 | 空 |
| `VELA_INSTALL_TOKEN` | 网页安装口令 | 空（只能用 CLI 安装） |
| `VELA_ADMIN_USER` / `VELA_ADMIN_PASSWORD` | 安装时创建的管理员 | admin / 随机 |
| `VELA_ORDER_RATE_LIMIT` | 每 IP 每小时可提交订单数 | 10 |
| `VELA_VERIFY_RATE_LIMIT` | 每 IP 每小时可校验次数 | 120 |
| `VELA_NOTIFY_ADMIN` | 是否给管理员发新订单邮件 | true |

> 163 邮箱的授权码获取：登录网页版 → 设置 → POP3/SMTP/IMAP → 开启 SMTP 服务 → 生成授权码。
> 注意发件人必须与登录账号一致；出现 `535` 基本都是把登录密码当授权码用了。

## 4. 接口约定

### POST /api/payment-notify.php

```json
{ "email": "buyer@example.com", "plan": "Y1",
  "deviceId": "ABCD1234EF567890", "amount": 2800, "channel": "wechat" }
```

- `plan`：`H6` / `Y1` / `LIFE`；`channel`：`wechat` / `alipay` / `paypal`
- `amount` 是**分**，服务端会与价格表比对，不一致直接拒绝（防止改价）
- 同一 `邮箱 + 设备码 + 档位` 已有待处理订单时返回「请勿重复提交」
- 成功返回 `{ "ok": true, "message": "已收到（单号 #12）..." }`

### POST /api/verify.php

```json
{ "licenseKey": "VELA-Y1-ABCD1234EF567890-4615ED3E8240C791",
  "deviceId": "ABCD1234EF567890" }
```

- 成功：`{ "ok": true, "licenseFile": "...", "plan": "Y1", "expiresAt": "...", ... }`
- 拒绝（未签发 / 已到期 / 已作废）：`{ "ok": false, "message": "..." }`
  → 客户端会**清除本地授权**
- 服务端自身故障：`503/500` **纯文本**，客户端会当作「暂时不可用」继续用离线授权。
  这一条很重要：配置错误或数据库故障时，**不要**返回 `ok:false`，否则等于批量吊销用户授权。

## 5. 日常运营

1. 打开后台 → 仪表盘看「待处理订单」数量。
2. 进「待处理订单」，核对到账金额与渠道。
3. 点「签发」→ 表单自动带入邮箱 / 设备码 / 档位 → 提交。
   系统会：写 `licenses` → 发激活码邮件 → 把订单标记为已签发。
4. 用户没收到邮件：去「已签发」搜邮箱 → 点「重发邮件」。
5. 退款：在「已签发」点「作废」。客户端下次联网确认即失效；完全离线最多再用 30 天。

**续费**：同一设备 + 同一档位再次签发，激活码不变，`expires_at` 从当前到期日往后累加。
用户不需要重新输入激活码，客户端联网确认后会自动变成新的到期日。

## 6. 故障排查

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 用户提交订单报「价格已调整」 | 客户端内置价格与服务端不一致 | 统一改价格后重新发版 |
| 用户收到邮件但激活提示「未签发」 | 后台只标了状态没点签发，或数据库未写入 | 检查「已签发」列表 |
| 用户激活提示「未签发或已撤销」 | 激活码是别的密钥签的（测试残留） | 核对 `license_secret` |
| 邮件发不出去，日志 `535` | 163 授权码错误 / 发件人与登录账号不一致 | 重新生成授权码 |
| 邮件进垃圾箱 | 163 个人邮箱发信在 Gmail/QQ 容易进垃圾箱 | 让用户加白名单；有条件改用域名邮箱 + SPF/DKIM |
| 后台登录提示尝试过多 | 同 IP 15 分钟内失败 8 次 | 等待或清 `login_failures` 表 |
| 接口 500 | 数据库不可用 / 表未建 | 跑 `install.php`，看 PHP error_log |

## 7. 安全清单

- [ ] `license_secret` 是随机串且与客户端一致；泄露后必须同时换服务端与 App
- [ ] `config.local.php` 不入库、不可通过 URL 读到（Apache 已由 `.htaccess` 拦截；
      Nginx 需自行加 `location ~ ^/server/(_lib|tools)/ { deny all; }` 与
      `location = /server/config.local.php { deny all; }`）
- [ ] `install.php` 安装后删除
- [ ] 后台密码 ≥ 16 位随机；不要与其它系统共用
- [ ] 数据库账号只授权本库，不要用 root
- [ ] 建议给 `api/verify.php` 与 `api/payment-notify.php` 配置 HTTPS
- [ ] 定期看 `audit_log`（登录、签发、作废、重发都有记录）

## 8. 数据结构

| 表 | 作用 |
| --- | --- |
| `pending_orders` | 用户付款申报；`status` = `pending` / `issued` / `rejected` |
| `licenses` | 签发结果；`UNIQUE(device_id, plan)`，续期通过更新 `expires_at` 实现 |
| `admins` | 后台账号（`password_hash` 存 bcrypt） |
| `rate_limits` | 接口限流计数 |
| `login_failures` | 后台登录失败计数（15 分钟窗口） |
| `audit_log` | 后台操作审计 |
