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

**照搬，不要自由发挥。** 这是测试过的方案：程序、两个 routine 指令、`active.md` 机制都在 skill 仓库 [psylch/grokbot-wechat-setup-skill](https://github.com/psylch/grokbot-wechat-setup-skill) 里（`bridge/`、`routines/`、`templates/`、`MECHANISMS.md`）。原样安装，不要重写转发器、不要自己编 routine 指令。动手前先读 `MECHANISMS.md`。安装目录下文写作 `$BRIDGE`（例如 `/workspace/wechat-bridge`）。

---

## 步骤 0：安装（你做，不用用户动手）

```bash
git clone https://github.com/psylch/grokbot-wechat-setup-skill.git /tmp/wechat-setup
cp -r /tmp/wechat-setup/bridge "$BRIDGE" && cd "$BRIDGE"
pip install -r requirements.txt
cp .env.example .env
cp /tmp/wechat-setup/templates/active.md "$BRIDGE/active.md"
```

先把 `.env` 里 Webhook 留空。用户显示名写进 `OWNER_NAME`。安装好后告诉他：「代码就绪，接下来用微信扫码绑定。」

---

## 步骤 1：扫码绑定（你全程盯着，用户只管扫）

**你做**

1. 后台启动 `python3 "$BRIDGE"/bind.py`，把输出写进日志文件。
2. `qr.png` 一生成，立刻作为图片附件发给用户（`file://$BRIDGE/qr.png`）。
3. 发码时一次说清：用想连接的那个微信扫，手机上点确认；**码大约两分钟有效，过期我会自动换新的发你**；手机若显示一串数字就发给我。
4. **自己盯着，不等用户开口**：每隔几秒看一次 `bind.py` 日志。
   - 出现新 QR（过期自动刷新）→ 马上把新 `qr.png` 发给用户，附一句「刚才那张过期了，用这张」。
   - `need_verifycode` → 请用户发数字，写进 `$BRIDGE/verify_code.txt`。
   - `confirmed` / `BOUND OK` → 不用等用户说「扫好了」，主动告诉他「绑定成功」，直接进入步骤 2。
5. 成功后确认有 `credentials.json`（权限 600），内容不要给用户看。

---

## 步骤 2：入口 Webhook（必须在电脑上，一口气发完）

**你做**

1. 用 `UpdateRoutine` 建 Webhook routine「微信消息入口」，指令原样使用 skill 仓库 `routines/inbound-webhook.md`（`$BRIDGE` 换成实际路径）。
2. 保存成功后，**在同一轮里连续发出**下面三样，不要等用户回「拿到了」再发下一步：
   - 一条说明：这一步要在电脑版 Grok Bot 里做（手机打不开下面的链接）；点下面两个链接，复制里面的内容，然后**直接贴进紧跟着的两个安全输入框**，不要贴到聊天里。顺带一句这两样是干嘛的：「一个是微信消息要送到的地址，一个是开门的钥匙」。
   - 只给两个链接：**Webhook URL** 和 **Webhook key**（用 automation_status 里该 routine 的 pill 链接）。**不要给 Authorization header**，也**不要写「去 Routines 里点进去」这类点击路径**。
   - 紧接着发两个 secret-request：`WECHAT_WEBHOOK_URL`（label「微信入口 Webhook URL」）和 `WECHAT_WEBHOOK_KEY`（label「微信入口 Webhook key」）。
3. 两个都保存后写进 `$BRIDGE/.env`（从环境变量读，不回显）。

---

## 步骤 3：自检（你做）

```bash
cd "$BRIDGE" && python3 forward.py --selftest
```

期望 HTTP 200，入口 routine 被触发一次。4xx 说明 URL 或 key 不对：请用户在电脑上重新复制，再发一次那两个安全输入框。成功不用单独告诉用户，直接进步骤 4。

---

## 步骤 4：启动转发器并主动发第一条微信（给用户确认感）

```bash
cd "$BRIDGE" && nohup setsid ./run_forward.sh >/dev/null 2>&1 &
```

启动后**你先主动往用户微信发一条**，不要先让用户去发：

```bash
python3 "$BRIDGE"/send.py --app "连上了！以后在微信里直接跟我说话就行，回我一句试试。"
```

- exit 0：在聊天里告诉用户「我刚往你微信发了一条，去看看」。
- exit 3（排队）：说明推送窗口没开，请用户先在微信里给 ClawBot 随便发一句，排队的那条会自动补发。
然后等用户在微信里回一句，确认来回都通，再进步骤 5。

---

## 步骤 5：看门狗（每 15 分钟）

创建**定时** routine（不是 Webhook），名如「微信转发看门狗」，cron 约 `7,22,37,52 * * * *`（或用户时区下每 15 分钟），指令原样使用 skill 仓库的 `routines/watchdog.md`（路径换成 `$BRIDGE`）：

```
运行 bash $BRIDGE/watchdog.sh
- OK：保持安静，不要发消息
- RESTARTED：一般不通知；同一天超 3 次再提醒
- FAILED：查 forward.out / forward.log，修不了就简短告诉用户
```

**别把看门狗做成 Webhook**。看门狗是定时任务，不需要 URL/key。若用户混淆，明确说：「入站用 Webhook（刚才填的那个）；看门狗是另一个定时 routine，不用填链接。」

---

## 步骤 6：进行中事项（推荐）

`$BRIDGE/active.md` 已在步骤 0 从模板拷好（每条：时间｜事项｜链接｜负责｜状态｜在等什么）。同时给主 bot 存一条长期记忆：App 侧推任务相关微信时同步更新它。

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
| 二维码过期 | 你自己发现并主动重发新 `qr.png` |
| 主动推送失败 | 24h 窗口关了，已排队；用户再说一句会补发 |
| 回了几轮互相打架 | 确保用 `--reply-to-ms`；exit 4 就停 |
| 看门狗和入站搞混 | 入站=Webhook；看门狗=定时，不需 URL |
| 两个 forward 互偷消息 | 不要手动再启一个；不要和 `poll.py` 同时跑 |

---

## 不要做

- 不要在没附二维码图的情况下让用户去扫码。
- 不要开场问用户微信版本或是否开通 ClawBot。
- 不要假设手机能打开 webhook 面板。
- 不要给点击路径（如「去 Routines 里找」），只给可点的链接。
- 不要给 Authorization header 链接，只给 URL 和 key 两个。
- 不要让用户回「拿到了」「扫好了」才走下一步；你能自己检测的就自己检测。
- 不要让用户把 Bearer key 贴进聊天。
- 不要把看门狗建成 Webhook routine。
- 不要在转发器已跑时再启一个或跑 `poll.py`。
- 不要自己重写转发器、send 脚本或 routine 指令；用仓库里测试过的版本。
