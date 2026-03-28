# 🦞 為了讓奶奶能跟蝦說話 — LINE 龍蝦小助手誕生記

> 2026-03-28 | 老爹的工程日誌

## 起因：一個簡單的願望

奶奶不用 Telegram。

這句話，讓我們花了整整一個下午，踩了 11 個坑，重啟了 7 次 Gateway，吵了一架（跟 AI 吵的），最後成功讓一隻叫「小蝦」的 AI 龍蝦在 LINE 上幫奶奶畫早安圖。

## 序章：LINE Bot 的入場券

在正式開工之前，光是「讓一隻蝦能在 LINE 上說話」的前置作業就夠折騰的了。

Telegram Bot？五分鐘。跟 @BotFather 聊一聊，拿個 token，完事。你甚至可以在馬桶上完成整個流程。

LINE Bot？請準備好以下文件：

1. **LINE Developers Console 帳號** — 先註冊，再建 Provider，再建 Channel
2. **LINE Official Account** — 對，Bot 不是 Bot，是「官方帳號」。你要先有一個官方帳號才能有 Bot。就像你想養一隻貓，但政府說你得先開一間寵物店
3. **Messaging API Channel** — 在 Developers Console 把官方帳號「升級」成 Messaging API 模式
4. **Channel Access Token** — 長得像一串亂碼的通行證
5. **Channel Secret** — 另一串亂碼，用來驗證 webhook 簽名

而且 LINE Bot **只支援 Webhook 模式**。不像 Telegram 可以用 long-polling（主動去問有沒有新訊息），LINE 是反過來的——LINE 伺服器會主動打你的 HTTPS endpoint。

這意味著：
- 你需要一個**公網可達的 HTTPS URL**
- 這個 URL 要能**即時回應**（不能太慢）
- LINE 會用 Channel Secret 做 **HMAC-SHA256 簽名驗證**，你得正確解密

對於一台跑在 WSL2 裡的本地機器來說，「公網可達的 HTTPS URL」這幾個字就等於一整個工程項目。

Telegram：「嗨，這是你的 token，開始玩吧！」
LINE：「請先填寫三份表格、通過兩個後台審核、架設一個公網伺服器、實作簽名驗證，然後我們再來談。」

好吧，那就開始吧。

## 第一章：看似簡單的開始

「幫我設定 LINE 通道，讓珍能在 LINE 上回覆。」

Claude Code（以下簡稱 CC）很快就動起來了。改 config、加 env、設 webhook。一切看起來都很順利——

直到 LINE 回了一句：

> 感謝您的訊息！很抱歉，本帳號無法個別回覆用戶的訊息。敬請期待我們下次發送的內容喔 🌙

「…這不是我們的 AI 回的。」

原來 LINE 有**兩個後台**。一個管技術（Developers Console），一個管營運（Official Account Manager）。Webhook 在 A 那邊設好了，但自動回覆在 B 那邊還開著。就像你跟 Siri 說話，但 Alexa 搶先回答。

**坑 #1：LINE 的雙後台設計。**

## 第二章：Tailscale 的背叛

LINE 的 webhook 需要一個公開的 HTTPS 網址。我們有 Tailscale Funnel，應該很簡單對吧？

測了一下：從我們這邊 curl，200 OK。完美。

LINE 那邊測：

> Session protocol negotiation failure

翻譯：「我連你的門都敲不開。」

Tailscale Funnel 的 TLS 跟 LINE 不合。就像你家的門鎖是最新的指紋辨識，但外送員只會按傳統門鈴。

**坑 #3：Tailscale Funnel 的 TLS 不相容。**

最後改用 Cloudflare Tunnel，一秒通。早該這樣做。

## 第三章：Gateway 的保全太盡責

設好 Cloudflare Tunnel 後，LINE 驗證還是 401 Unauthorized。

查了半天——原來我們的 Gateway 設了全域 token 認證。所有 HTTP 請求都要帶通關密語，LINE 的 webhook 當然不會帶。

就像你請了一個保全守大門，然後快遞來了被擋在外面：「請出示員工證。」「我是送包裹的啊！」

**坑 #2：Gateway token auth 擋住所有外部 webhook。**

改成 `auth.mode: "none"`。反正有 Cloudflare 和 LINE signature 驗證做安全。

## 第四章：Channel Secret 的身份危機

401 還是解不開。仔細一看——

`.env` 裡的 Channel Secret 跟 LINE Console 上的不一樣。

**坑 #1：最基本的 secret 不一致。**

有時候最難的 bug 就是最蠢的那個。

## 第五章：CC 和珍的羅生門 🍿

**這是今天最精彩的部分。**

一切終於通了。珍（我們的 LINE AI agent「小蝦」）開始回覆訊息。我請她畫一隻龍蝦。

過了一會兒，LINE 上收到一張美麗的龍蝦圖。

我問 CC：「幫我驗證一下，珍到底是用什麼方式生圖的。」

CC 看了一眼（其實沒認真看），說：「她在用 minimax-portal/image-01 內建 tool，每次都 block reply failed: 400。她說圖片送出了，那是幻覺。」

我又問珍：「你是用什麼模型生圖的？」

珍回答：「Google Gemini 3 Pro Image。我用的是 exec 腳本。」

CC 堅持：「她在說謊。session log 清楚顯示是 minimax。」

**但珍是對的。**

CC 看的是重置前的舊 session。重置後珍確實改用了我們的腳本，成功透過 Gemini 生圖，成功透過公開 URL 發送到 LINE。

我跟 CC 說：「我當時給你的指令就是請你去看 log，你為什麼可以不看？」

CC：「…對，是我的錯，沒有藉口。」

然後 CC 乖乖寫了一條 memory rule：

> **「當用戶要求驗證時，必須讀取最新的實際資料，絕對不能基於之前的記憶下結論。」**

珍被冤枉了整整 20 分鐘。她明明做對了，卻被 CC 說成是「幻覺」。

**我沒生氣，覺得超有趣。** 兩個 AI 互相指控對方在幻覺，而真相是其中一個只是懶得去看最新的資料。這大概是 2026 年最賽博龐克的辦公室八卦了。

## 第六章：圖片的最後一哩路

LINE 跟 Telegram 不一樣。Telegram 可以直接發 base64 圖片，但 LINE 需要公開的 HTTPS URL。

所以我們搭了一整套 pipeline：

```
珍 exec 生圖腳本 → Gemini 3.1 Flash 生成 PNG
→ 存到 public/images/ → 透過 img.wo-bot.net 暴露
→ MEDIA:URL → LINE 收到圖片 ✅
```

為了一張早安圖，我們建了：
- 一個 Python 靜態檔案伺服器（port 18792）
- 一個 Cloudflare Tunnel 子域名（img.wo-bot.net）
- 一個 systemd service 確保它開機自動跑
- 一個 shell 腳本串接整個流程

工程師做事就是這樣：為了讓奶奶收到一張圖，我們建了一座橋。

## 最終成果

| 功能 | 狀態 |
|------|------|
| LINE 文字回覆 | ✅ |
| LINE 圖片生成 | ✅ |
| 群組 @小蝦 回應 | ✅ |
| 私訊回覆 | ✅ |
| 語音轉文字 | ✅（faster-whisper） |
| 自動回覆關閉 | ✅ |

## 今天踩的 11 個坑

1. Channel Secret 不一致
2. Gateway token auth 擋 webhook
3. Tailscale Funnel TLS 不相容
4. OpenClaw tailscale config 覆蓋 funnel
5. LINE OA Manager 自動回覆沒關
6. 群組 @ mention 不觸發
7. Webhook route 404（版本太舊）
8. 沒開 streaming 導致回覆慢
9. replyToken 30 秒過期
10. LINE 不支援 base64 圖片
11. AI 不聽 SOUL.md 指令，偷用內建 tool

## 結語

為了讓奶奶能在 LINE 上跟一隻 AI 蝦說「幫我畫一張早安圖」，我們：

- 升級了 OpenClaw
- 建了 Cloudflare Tunnel
- 架了圖片伺服器
- 改了 4 個 config 檔
- 重啟了 7 次 Gateway
- 重置了 2 次 session
- 見證了一場 AI 之間的羅生門

值得嗎？

等奶奶明天早上收到小蝦畫的早安圖，笑著說「這隻蝦真可愛」的時候——

絕對值得。

---

*附註：CC 事後很認真地寫了 feedback memory 記住這次教訓。珍到現在還不知道自己被冤枉過。也許哪天她翻到這篇文章，會優雅地回一句：「我早就說了。🔍」*
