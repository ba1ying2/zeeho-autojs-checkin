# 极核 ZEEHO 自动签到 · AutoX.js 版

用一台闲置的安卓机每天自动给极核 App 签到，**不用抓包、不用配代理证书、不装 APK、不依赖无障碍服务**。
装个 [AutoX.js](https://github.com/kkevsekk1/AutoX)，放两个文件，跑起来就不用管了。

签名算法与接口实现来自 [cluck798/ZEEHO](https://github.com/cluck798/ZEEHO)，这里做的是一个更轻量的部署形态。

## 它怎么工作

```
AutoX.js  ──HTTP+签名──▶  h5.zeehoev.com /cfmotoservermine/H5/signin
    │
    └── 读 /sdcard/脚本/zeeho_token.txt（token,user_id）
        文件一变就热加载，不用重启脚本
```

- 启动即查一次签到状态，之后每 10 分钟查一次
- 已签到 → 休眠，每 5 分钟比对日期，**跨天自动重签**（0 点后自己补上，不用定闹钟）
- 未签到 → 连续重试 3 次，成功后回查确认
- token 失效（401 / `invalid_token`）→ 震动提醒 + 每 2 分钟快速重试，等新 token 送进来

## 快速开始

1. 装 AutoX.js v7，授予存储权限
2. 把 `autojs/zeeho_autojs.js` 放到 `/sdcard/脚本/`
3. 拿 token 写进 `/sdcard/脚本/zeeho_token.txt`（一行，`token,user_id`）：

   ```bash
   node autojs/get_token.js --phone 13800000000            # 发验证码
   node autojs/get_token.js --phone 13800000000 --code 123456   # 登录，输出 token,user_id
   adb push zeeho_token.txt /sdcard/脚本/zeeho_token.txt
   ```

4. AutoX 里运行脚本，保持后台

详细说明、保活设置、接口备查都在 [autojs/README.md](autojs/README.md)。

## 文件

| 文件 | 作用 |
|---|---|
| `autojs/zeeho_autojs.js` | AutoX.js 单文件签到脚本，内嵌 md5/sha1，可直接改 |
| `autojs/get_token.js` | Node 零依赖，手机号 + 短信验证码换 token（免抓包） |
| `autojs/README.md` | 使用说明 + 实测结论（含几个容易踩的坑） |

## 实测踩到的坑（写下来省别人的时间）

1. **token 是单点的**：同账号在 iPhone 上登录后，安卓端原 token 立即 `401 invalid_token`，几小时前刚拿的也一样；
2. **发码接口后发顶前发**：重发会让上一条验证码立即作废，所以「发码 → 读短信 → 提交登录」中间绝不能再发，
   否则会一直报 `验证码有误或已过期`，很容易误判成签名问题；
3. **登录接口不校验 body 入签**：`loginByPhone` 是 POST，但 GET 风格入签和 query+body 入签服务端都接受；
4. **token 约 10 天有效且无可用 refresh**：`/oauth/token` 需 App 内置 client 凭证，`/user/refreshToken` 不存在，
   `/user/tokenRefresh` 是心跳接口。到期只能重新登录 —— 所以脚本做了 token 热加载。

## 免责

仅供学习研究与个人自用，请勿商用。签名常量、接口信息均来自 cluck798/ZEEHO 的既有实现，
使用遵循原项目的《非商业许可协议》，风险自负。
