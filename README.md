# 极核 ZEEHO 自动签到 · 安卓 AutoX / iOS Scriptable

每天自动给极核 App 签到，**不抓包、不装 APK、不依赖无障碍服务、不root**。
两套方案任选其一：

| 方案 | 适用 | 特点 |
|---|---|---|
| **安卓 AutoX.js** | 有闲置安卓机 | 常驻循环，跨天自动重签；需自己解决后台保活 |
| **iOS Scriptable** | 日常在用的 iPhone | 快捷指令定时，**锁屏也能跑**；无需保活 |

签名算法与接口实现来自 [cluck798/ZEEHO](https://github.com/cluck798/ZEEHO)，
这里做的是两个更轻量的部署形态。

## 它怎么工作

```
AutoX.js / Scriptable  ──HTTP + H5签名──▶  h5.zeehoev.com
                                            /cfmotoservermine/H5/signin
```

两边共用同一套签名：`md5(sha1(query + body + appId&nonce&timestamp + secret), 32)`。

## 方案一：iOS Scriptable（推荐）

不需要把手机一直插着电，也不需要保活。

1. App Store 装 **Scriptable**（免费，开发者 Simon Støvring）
2. 把 `ios/zeeho_checkin.js` 放进 iCloud 云盘 → `Scriptable` 目录
3. 在 Scriptable 里运行一次，输入手机号 + 短信验证码完成登录
4. 快捷指令 → 新建 → 添加 `Run Script`（Scriptable）→ 选 `zeeho_checkin`
5. 给它加「特定时间」触发器（如 00:05 每天）
6. **关掉「运行前确认」，打开「锁定时允许运行」**（iOS 27 在页面底部「ⓘ」→ 隐私）

完整步骤（含 iOS 27 与旧版差异、锁屏设置、常见问题）见 [ios/README.md](ios/README.md)。

## 方案二：安卓 AutoX.js

1. 装 [AutoX.js](https://github.com/kkevsekk1/AutoX) v7，授予存储权限
2. 把 `autojs/zeeho_autojs.js` 放到 `/sdcard/脚本/`
3. 拿 token 写进 `/sdcard/脚本/zeeho_token.txt`（一行，`token,user_id`）：

   ```bash
   node autojs/get_token.js --phone 13800000000                  # 发验证码
   node autojs/get_token.js --phone 13800000000 --code 123456    # 登录，输出 token,user_id
   adb push zeeho_token.txt /sdcard/脚本/zeeho_token.txt
   ```

4. AutoX 里运行脚本，并在系统设置里给 AutoX 开**自启动**、省电策略**无限制**、最近任务里**锁定**

> 实测提醒：MIUI 等系统仍可能半夜清理后台进程（表现为进程还在、但脚本不再执行）。
> 如果连续几天漏签，建议换 iOS 方案。

详细说明、保活设置、接口备查见 [autojs/README.md](autojs/README.md)。

## 文件

| 文件 | 作用 |
|---|---|
| `ios/zeeho_checkin.js` | Scriptable 签到脚本，内嵌 md5/sha1，iCloud 存 token 与日志 |
| `ios/zeeho_set_token.js` | 导入 token 小工具（从剪贴板或弹窗读 `token,userId`） |
| `ios/README.md` | iOS 安装、定时自动化、常见问题 |
| `autojs/zeeho_autojs.js` | AutoX.js 单文件签到脚本，可直接改 |
| `autojs/get_token.js` | Node 零依赖，手机号 + 短信验证码换 token（免抓包） |
| `autojs/README.md` | 安卓版使用说明 + 实测结论 |

## 实测踩到的坑（写下来省别人的时间）

1. **token 是单点的**：同账号在 iPhone 上登录后，安卓端原 token 立即 `401 invalid_token`，
   几小时前刚拿的也一样。想两端同时在线，只能共用同一条 token（iOS 版支持导入 App 的 token）。
2. **发码接口后发顶前发**：重发会让上一条验证码立即作废，所以「发码 → 读短信 → 提交登录」
   中间绝不能再发，否则一直报 `验证码有误或已过期`，很容易误判成签名问题。
3. **登录接口不校验 body 入签**：`loginByPhone` 是 POST，但 GET 风格入签和 query+body 入签服务端都接受。
4. **token 约 10 天有效且无可用 refresh**：`/oauth/token` 需 App 内置 client 凭证，
   `/user/refreshToken` 不存在，`/user/tokenRefresh` 是心跳接口。到期只能重新登录。
5. **js md5 默认可能是 16 位短哈希**：必须显式取 32 位，否则服务端返回 `30121 permit error`。
6. **锁屏/后台不能弹窗**：iOS 快捷指令调用时弹窗显示不出来会让脚本一直等待，
   要用通知代替（脚本已按 `config.runsInShortcut` 自动切换）。
7. **iOS 27 的自动化入口变了**：底部没有「自动化」标签，触发器挂在快捷指令第一步；
   「锁定时允许运行」在页面**底部**「ⓘ」→ 隐私里，不在右上角。

## 免责

仅供学习研究与个人自用，请勿商用。签名常量、接口信息均来自 cluck798/ZEEHO 的既有实现，
使用遵循原项目的《非商业许可协议》，风险自负。
