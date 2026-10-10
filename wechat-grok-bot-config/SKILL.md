---
name: WeChat Grok Bot 配置
description: >-
  Use when the user wants to connect WeChat to Grok Bot (ClawBot / iLink
  bridge): first-time bind, webhook routines, forwarder, or watchdog setup.
---
# WeChat → Grok Bot 配置（UX 导引）

把微信（腾讯官方 ClawBot / iLink）接到 Grok Bot：用户在微信里给 ClawBot 发消息 → box 上转发器收到 → POST 到 Webhook routine → agent 用 `send.py` 回微信。

## 背景资料（新 bot 先看这些）

- 桥接代码与命令行工具：[psylch/grokbot-wechat](https://github.com/psylch/grokbot-wechat)。`bind.py` 扫码绑定，`forward.py` / `run_forward.sh` 转发器，`send.py` 回复和推送（exit 3 已排队，exit 4 被新消息取代），`watchdog.sh` 看门狗，`routines/` 里是两个 routine 模板。它的 README 写了架构、payload 格式、24 小时推送窗口和踩过的坑，动手前先读。
- iLink 协议拆解：[WeChat ClawBot iLink protocol](https://www.chihao.li/blog/en/wechat-clawbot-ilink-protocol)
- 腾讯官方参考实现：[@tencent-weixin/openclaw-weixin](https://www.npmjs.com/package/@tencent-weixin/openclaw-weixin)

## 原则

- 一次只让用户做一件事；每步完成再进下一步。
- 用用户语言回复（多数用户用中文）。
- Webhook URL / key / Authorization header **只能在电脑（Mac/PC）Grok Bot 客户端打开复制**；手机 App 当前点不开这些链接。**从第一次提到 Webhook 就明说这点**，别等用户在手机上撞了才说。
- 你自己看不到 Webhook URL/密钥的明文；用 secret-request 让用户填，或让他们把值填进 box 上的 `.env`（不要让他们把 key 贴到聊天里）。
- `credentials.json`、Webhook key 绝不贴聊天、不进 git。

## 前置（不要问用户）

- 不要问用户「微信开通 ClawBot 了吗」、版本号之类，他们不该需要知道。直接走到扫码。
- 只有扫码失败时再处理：扫了没反应、提示不支持、或微信里找不到 ClawBot → 用一句人话告诉他「你的微信可能还没拿到这个功能，先升级到最新版再扫；仍不行说明还在灰度，暂时接不了」。
- 配 Webhook 需要电脑版 Grok Bot，这一点到步骤 2 前再说，不要开场就问。
- 转发器默认跑在你的 box，不必问用户。

代码仓库（若本地已有可复用副本则用副本）：`https://github.com/psylch/grokbot-wechat`。安装目录下文写作 `$BRIDGE`（例如 `/workspace/grokbot-wechat`）。

---

## 步骤 0：安装（你做，不用用户动手）

```bash
git clone https://github.com/psylch/grokbot-wechat.git "$BRIDGE" && cd "$BRIDGE"
pip install -r requirements.txt
cp .env.example .env
```

先把 `.env` 里 Webhook 留空。用户显示名写进 `OWNER_NAME`。安装好后告诉他：「代码就绪，接下来用微信扫码绑定。」

---

## 步骤 1：扫码绑定（UX 最重要）

**你先做**

1. 在 box 上启动（后台）：`python3 "$BRIDGE"/bind.py`。
2. 等 `qr.png` 生成（同目录还有 `qr_url.txt`）。
3. **立刻把二维码图片附件发给用户**（`SendToUser` type attachment，`file://$BRIDGE/qr.png`），不要只说「去目录里找」。

**对用户说（模板）**

> 请用**你想连接的那个微信**扫下面这张码，手机上点确认。  
> 若手机显示一串数字，把数字发回来（我会写入验证文件）。  
> 二维码会过期；过期了跟我说，我换一张再发你。

**你等**

- 监控 `bind.py` 输出：`confirmed` / `BOUND OK` → 继续；`need_verifycode` → 用户发来的数字写入 `$BRIDGE/verify_code.txt`；`expired` → 等新 `qr.png` 再附件发一次。
- 成功后确认存在 `credentials.json`（权限 600），**不要**把内容读给用户看。
- 告诉用户：「绑定成功。接下来在电脑上配 Webhook（手机打不开）。」

---

## 步骤 2：入站 Webhook routine（必须电脑）

**先明确告知**

> 下一步要复制 Webhook 的 URL 和 Authorization。**这三个入口只能在电脑版 Grok Bot 里打开**（手机 App 点链接无法打开面板）。请切到 Mac/PC 继续。

**你做**

1. 用 `UpdateRoutine` 创建一个 Webhook routine（名如「微信消息入口」），指令基于仓库 `routines/inbound-webhook.md`，把路径换成 `$BRIDGE`，并加上：
   - 每次先读 `$BRIDGE/active.md`（进行中事项）再回复，能看出接哪件事就不要再问「是谁/哪个站」。
   - 用 `python3 $BRIDGE/send.py "…" --reply-to-ms <trigger_ms>` 回微信。
2. 保存成功后，把自动化状态里的三个链接发给用户（用官方 pill 链接）：
   - Webhook URL
   - Webhook key
   - Authorization header

**对用户说（模板）**

> 请在**电脑**上依次点开下面三个链接，复制内容：  
> 1) Webhook URL  
> 2) Webhook key（或直接复制 Authorization header）  
> **不要贴到聊天里**。复制好后回我一句「拿到了」，我会用安全输入框收 key（或指导你把 URL+密钥写进 box 上的 `.env`）。

**收密**

- 优先：`secret-request` 收 `WECHAT_WEBHOOK_KEY`（和如果需要的 URL）；或让用户在桌面把值贴进 `$BRIDGE/.env` 后说「填好了」。
- 写入 `.env`：`WECHAT_WEBHOOK_URL=...`、`WECHAT_WEBHOOK_KEY=...`。确认文件权限不对外可读。

---

## 步骤 3：自检

```bash
cd "$BRIDGE" && python3 forward.py --selftest
```

期望 HTTP 200，并且入站 routine 被触发一次。失败：

- 4xx：URL/密钥错，请用户在**电脑**上重新打开链接核对（别在手机上试）。
- 网络/超时：稍后重试。

通过后告诉用户：「Webhook 通了。」

---

## 步骤 4：启动转发器

```bash
cd "$BRIDGE" && nohup setsid ./run_forward.sh >/dev/null 2>&1 &
tail -n 20 forward.log
```

请用户在微信里给 ClawBot 发一句「你好」。你应收到 webhook 触发并用 `send.py --reply-to-ms …` 回一句。通了再进看门狗。

---

## 步骤 5：看门狗（每 15 分钟）

创建**定时** routine（不是 Webhook），名如「微信转发看门狗」，cron 约 `7,22,37,52 * * * *`（或用户时区下每 15 分钟），指令基于 `routines/watchdog.md`：

```
运行 bash $BRIDGE/watchdog.sh
- OK：保持安静，不要发消息
- RESTARTED：一般不通知；同一天超 3 次再提醒
- FAILED：查 forward.out / forward.log，修不了就简短告诉用户
```

**别把看门狗做成 Webhook**。看门狗是定时任务，不需要 URL/key。若用户混淆，明确说：「入站用 Webhook（刚才填的那个）；看门狗是另一个定时 routine，不用填链接。」

---

## 步骤 6：进行中事项（推荐）

创建 `$BRIDGE/active.md`（表格：时间｜事项｜链接｜负责｜状态｜在等什么）。

规则（写进记忆/入站 routine）：

- App 侧用 `send.py`（尤其 `--app`）推任务相关微信时，同步更新 `active.md`。
- 微信触发的轮次先读 `active.md`，避免上下文丢失后重复追问。

---

## 步骤 7：收尾（用人话说明整套在干啥）

配置完成后，用简单非技术的话跟用户讲清楚，不要堆术语。模板可参考：

> 搞定了。以后你在微信里跟我说话就行。  
> 背后有两件事在帮你：一个是「消息入口」，专门接你的微信；另一个是「看门狗」，每隔一会儿检查一下连接有没有挂，挂了会自己修好，一般不用你管。  
> 真正收发微信的是我这边一直开着的一个小程序，不是你要填的那些。  
> 主动提醒前面一般会带 `[grok bot]`。大概一天没说话，可能会收到一句续一下的提醒，回个 `1` 就行。  
> 以后如果要换密钥，只能在电脑版 Grok Bot 里打开那个入口去复制，手机打不开。

再补一句邀请：让他随便发一句试试。不要再问「ClawBot 开了吗」之类的。

---

## 常见坑（对用户要用人话）

| 现象 | 怎么办 |
|---|---|
| 手机点不开 Webhook 链接 | 换电脑客户端；你也看不到明文 URL/密钥 |
| 扫码后要数字 | 把数字发你，你写入 `verify_code.txt` |
| 二维码过期 | 你重发新 `qr.png` |
| 主动推送失败 | 24h 窗口关了，已排队；用户再说一句会补发 |
| 回了几轮互相打架 | 确保用 `--reply-to-ms`；exit 4 就停 |
| 看门狗和入站搞混 | 入站=Webhook；看门狗=定时，不需 URL |
| 两个 forward 互偷消息 | 不要手动再启一个；不要和 `poll.py` 同时跑 |

---

## 不要做

- 不要在没附二维码图的情况下让用户去扫码。
- 不要开场问用户微信版本或是否开通 ClawBot。
- 不要假设手机能打开 webhook 面板。
- 不要让用户把 Bearer key 贴进聊天。
- 不要把看门狗建成 Webhook routine。
- 不要在转发器已跑时再启一个或跑 `poll.py`。
