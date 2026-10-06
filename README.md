# group-judge（小判官）

LINE 群組的每月整理工具，用來取代「揪揪」裡我們實際在用的兩個功能：呼叫版規、
統計誰沒有發話。分成兩塊：

| 資料夾 | 內容 | 部署 |
|---|---|---|
| `group-judge-backend/` | 小判官（LINE bot）的 webhook，加上後台用的 API。Flask + Postgres | Render |
| `group-judge-web/` | 管理員用的網頁後台。純靜態，沒有 build step | GitHub Pages |

資料庫是 Neon Postgres。**Render 與 Neon 要開在同一區（Singapore）**，否則每次查詢
都要多等一秒以上。

## 小判官會做的事

| 誰、在哪裡 | 說什麼 | 小判官 |
|---|---|---|
| 任何人，群組或私訊 | 整則訊息就是「版規」或「板規」（群組裡不必 tag） | 回版規（卡片） |
| 非管理員，群組裡 tag 它／私訊它 | 其他任何話 | 「請不要跟我說話」 |
| 管理員，群組裡 tag 它 | 「本月移除清單」 | 依序回四則：開場、移除名單、不得背書名單、整理預告 |
| 管理員，群組裡 tag 它 | 「正式整理群組」 | 依序回兩則：白名單（背書留下的人）、再見訊息 |
| 管理員，群組裡 tag 它 | 「整理完畢」 | 回一則：廣告須知 |
| 管理員，群組裡 tag 它 | 其他 | 「我看不懂你在說什麼」 |
| 管理員，私訊 | 上面三個指令 | 同一批訊息，等於預覽 |
| 管理員，私訊 | 「本次統計期間加入成員」 | 回查詢結果 |
| 管理員，私訊 | 其他 | 指令列表 |

「本月移除清單」「正式整理群組」要用到這個月的名單，名單還沒在後台存過的話只回
「我的本本還是空的。」。這三個指令的內容都在後台「訊息」頁籤改。

另外它會默默做這些事：

- 群組裡有人說話（文字、貼圖、圖片都算）或加入，就記下這個人、哪個月、說了幾則。**不存訊息內容**（唯一的例外見下面「三個限制」的第 3 點）。
- 有人收回訊息，那一則不算。
- 有人加入就回歡迎訊息（卡片，帶他的大頭貼）；黑名單裡的人加入，另外回一張警告
  卡片；這個月才被列在移除名單上的人又加入，再提醒一句。
- 有人離開（自己退或被踢）就把狀態改成已離開。
- 被拉進別的群組會自己退出——第一個把它拉進去的群組就是它服務的群組。

**它不會主動對群組說話。** 對群組說的每一句都是「有人觸發、它回覆」，回覆不計入
LINE 的額度。它唯一會主動發的訊息是私訊管理員的一句「本月用量已接近超標，退回
省電模式」，一個月最多一次（見下面「資料庫用量與省電模式」）。

## 後台

用 Google 帳號登入，只有 `users` 表裡 `role = 1` 而且 email 對得上的人進得來；另外
`FIRST_ADMIN_EMAIL` 設的那個帳號一律進得來（成員資料是空的也一樣，見「上線」）。
同一個網址，手機自動進行動版、電腦進桌面版（選單裡可以手動換）。

- **版規**：看與改。
- **待移除名單**：整理當月的前兩個完整月份（10 月整理就是 8/1～9/30）**區間裡**
  沒有任何發話的人。區間開始後才加入的不管；區間結束後才說話的來不及，照樣
  上名單。「新增」可以手動加人，選取之後按「移除」是把人從名單拿掉（反白），
  「儲存」才寫進資料庫。被拿掉的人進白名單。
- **訊息**：小判官會發的每一則訊息。可以預覽實際發出去的樣子。
- **成員**：改身分（管理員／一般）、狀態（在群組／已離開／黑名單）、登入用的 email。
- **發話對照**：小判官認得的每個人在區間裡有沒有發話、每個月說了幾則，照 LINE
  成員名單的順序排，拿來跟 LINE 逐一對照——LINE 上有、這裡沒有的，就是小判官
  沒看過的人。LINE 沒有公布排序規則，所以提供幾種排法自己挑。
- **未知發話**：LINE 沒說是誰發的訊息。可以指認給待移除名單上的人，他就算有
  發話、移到白名單（要先存過名單）。
- 右上角「讀取成員資料」：向 LINE 確認每個人現在的名字。LINE 說查無此人的會列
  出來問你要不要標成已離開。

白名單的效力、誰算「沒發話」的細節寫在 `group-judge-backend/db/cleanup.py` 開頭；
哪些訊息會立刻寫進資料庫、數字怎麼保證是對的，寫在
`group-judge-backend/line_utils/tracker.py` 開頭。

## LINE 平台的三個限制（設計都是繞著它們走的）

1. **一個群組同時只能有一個官方帳號。** 揪揪跟小判官沒辦法並行，只能換掉。
   換上去的那一刻小判官誰都不認得，要等大家說話或有人加入才會一個一個認得；
   它進群之前的發話它也不知道，所以頭一兩次整理的名單會列多（後台會提醒是
   哪一段沒看到）。「發話對照」頁籤就是為了這段期間做人工對照用的。
2. **推播到群組是按人數計費的**（發一則到 150 人的群組扣 150 則），免費額度一個月
   200 則。所以沒有推播功能，公告改成管理員在群組叫小判官時它才回。
3. **官方文件說只有手機版發的訊息會附上發話者。** 沒附的訊息裡沒有任何能認人的
   東西，連哪幾則是同一個人發的都看不出來。這種訊息會留下時間與開頭 60 個字
   （一百天後刪除），讓管理員在「未知發話」頁籤對照聊天紀錄後指認。
   **2026-10-06 在測試群組試過：Mac 電腦版發的訊息有附發話者，小判官認得出來。**
   Windows 版與其他情況還沒試過；「成員」頁籤的「認不出發話者的訊息」那一格會寫
   統計區間之後又收到幾則，發一則之後看它有沒有增加就知道。

## 資料庫用量與省電模式

Neon 免費方案的資料庫**一個月只能醒著 400 小時**（100 CU-hours ÷ 0.25 CU，平均一天
13 小時）。它閒置滿 5 分鐘才會睡，這個時間改不了；額度用完，資料庫會被停到下個月
——那段時間小判官記不到任何人的發話。（https://neon.com/docs/introduction/plans ，
2026-10-07 查的。）所以碰一次資料庫的代價是「接下來 5 分鐘它都醒著」，群組整天有人
說話的話，每一則都寫就等於整天醒著。

後端會自己估計這個月醒了多久（每次連線之後算 5 分半，重疊的不重複算），顯示在後台
「成員」頁籤那排數字下面：

- **還不到 320 小時**：每一則訊息都立刻寫進資料庫，則數永遠是最新的。
- **到了 320 小時**：退回省電模式直到月底，並私訊每位管理員一句「本月用量已接近
  超標，退回省電模式」（一個月只發一次）。省電模式下，每個人當月的第一則照樣立刻
  寫，之後的先記在記憶體，晚一點才補寫——**誰有沒有發話不受影響**，只是資料庫裡
  的則數會慢一點；後台看到的數字一律是補寫過的。
- **換月**：從零算起，恢復每一則都寫。

要知道的幾件事：

- 管理員要**加小判官好友**才收得到那則私訊。每位管理員算 LINE 的一則額度（免費
  一個月 200 則）。
- 那個時數是估計值，算不到你在 Neon 後台 SQL Editor 下的查詢。實際用量在
  https://console.neon.tech/ 專案頁的用量圖表，兩邊差很多的話以 Neon 的為準。
- 到 320 之後還剩 80 小時。省電模式下會碰資料庫的只剩每個人當月的第一則、有人叫
  版規或下指令、收回訊息、成員進出與管理員開後台，量不大。真的太早就到 320 的話，
  把門檻調低（Render 的環境變數 `DB_SAVING_AFTER_HOURS`，設成 0 是完全不限制）。

細節在 `group-judge-backend/db/meter.py` 與 `line_utils/tracker.py` 開頭。

## 本機開發

三個東西都在本機跑：資料庫、假的 LINE、後端，再加前端。

```bash
cd group-judge-backend
python3 -m venv .venv && .venv/bin/pip install -r requirements-dev.txt
cp .env.example .env
```

```bash
.venv/bin/python tools/local_db.py
```

把印出來的 `DATABASE_URL=…` 貼進 `.env`，另外在 `.env` 裡填
`LINE_CHANNEL_SECRET=local`、`LINE_CHANNEL_ACCESS_TOKEN=local`、
`LINE_API_BASE=http://127.0.0.1:5099`。然後開三個終端機：

```bash
.venv/bin/python tools/fake_line.py
```

```bash
.venv/bin/python app.py
```

```bash
cd ../group-judge-web && python3 dev_server.py
```

模擬群組裡的事，以及不經過 Google 直接登入後台：

```bash
.venv/bin/python tools/send_event.py say U001 大家好
```

```bash
.venv/bin/python tools/dev_session.py you@example.com --create
```

第二個指令會印出一個網址，貼到瀏覽器就是登入後的畫面（只在 localhost 有效）。
`tools/send_event.py` 開頭列了所有能模擬的事件。

測試（會清空 `gj_test`，所以只認 `TEST_DATABASE_URL`）：

```bash
TEST_DATABASE_URL="postgresql://postgres:@/gj_test?host=$PWD/.localdb" .venv/bin/python -m pytest
```

## 上線

三個 repo：`group-judge`（這一層，只是把兩個子專案掛在一起）、`group-judge-backend`、
`group-judge-web`，做法同 boo-king-king。GitHub 的 SSH 別名是
**`github.com-beethoreven`**，不是純 `github.com`。

### 1. 推上 GitHub

**順序不能反**：先推兩個子專案，再推這一層——這一層記的是子專案的 commit。

```bash
cd group-judge-backend
git init && git add -A && git commit -m "Initial commit"
git remote add origin git@github.com-beethoreven:beethoreven/group-judge-backend.git
git branch -M main && git push -u origin main

cd ../group-judge-web
git init && git add -A && git commit -m "Initial commit"
git remote add origin git@github.com-beethoreven:beethoreven/group-judge-web.git
git branch -M main && git push -u origin main

cd ..
git init
git submodule add git@github.com-beethoreven:beethoreven/group-judge-backend.git group-judge-backend
git submodule add git@github.com-beethoreven:beethoreven/group-judge-web.git group-judge-web
git add -A && git commit -m "Initial commit"
git remote add origin git@github.com-beethoreven:beethoreven/group-judge.git
git branch -M main && git push -u origin main
```

推之前確認 `group-judge-backend/.env` 沒有被加進去（`.gitignore` 已經擋了）。

### 2. Neon

Region 選 Singapore（ap-southeast-1）。連線字串用預設那個有 `-pooler` 的就可以。
表會在後端第一次連上時自己建好，版規與各則訊息的預設內容也會一起寫進去。

### 3. Render（後端）

New → Web Service → 選 `group-judge-backend`，Region **Singapore**，Instance Type Free。

- Build Command：`pip install -r requirements.txt`
- Start Command：`gunicorn --workers 1 --threads 8 --timeout 60 app:app`
  （★ 只能一個 worker，理由見 `line_utils/tracker.py`）
- Health Check Path：`/health`
- 環境變數：

  | 名稱 | 值 |
  |---|---|
  | `DATABASE_URL` | Neon 的連線字串 |
  | `LINE_CHANNEL_SECRET` | 步驟 6 拿到 |
  | `LINE_CHANNEL_ACCESS_TOKEN` | 步驟 6 拿到 |
  | `GOOGLE_CLIENT_ID` | 步驟 4 拿到 |
  | `FIRST_ADMIN_EMAIL` | 第一位管理員的 Google 帳號。這個帳號一律登得進後台，成員資料是空的也一樣 |
  | `ALLOWED_ORIGINS` | `https://beethoreven.github.io`（只有網域，不含路徑、不要結尾斜線） |
  | `DB_SAVING_AFTER_HOURS` | 不必填。資料庫這個月醒了幾小時之後退回省電模式，預設 320（見上面「資料庫用量與省電模式」） |

  LINE 與 Google 的值還沒有的時候可以先留空，服務照樣起得來（只是小判官與登入
  先不能用），拿到之後補上、Render 會自動重啟。

### 4. Google Cloud Console（登入）

APIs & Services → Credentials → Create Credentials → OAuth client ID，類型選
「網頁應用程式」。「已授權的 JavaScript 來源」填 `https://beethoreven.github.io`
（只有網域，不含 `/group-judge-web/`），本機要測的話再加 `http://localhost:5173`。
「已授權的重新導向 URI」不用填。

第一次用這個 GCP 專案要先設 OAuth 同意畫面（External、填 App 名稱與你的 email
就好）。發佈狀態是 Testing 時，只有列在 Test users 的帳號登得進來——把三位管理員
的 Google 帳號加進去，或直接按 Publish。

### 5. GitHub Pages（前端）

`group-judge-web/index.html` 填兩個值，commit、push：

```js
window.__GROUP_JUDGE_API_HOST__ = 'https://<Render 給的網址>';
window.__GOOGLE_CLIENT_ID__ = '<步驟 4 的 Client ID>';
```

`group-judge-web` 這個 repo 的 Settings → Pages → Source 選「Deploy from a branch」，
分支 `main`、資料夾 `/ (root)`。網址是 https://beethoreven.github.io/group-judge-web/ 。

- 免費方案的 Pages 要求 repo 是公開的。
- 這個站的「來源」是 `https://beethoreven.github.io`（不含路徑）。Render 的
  `ALLOWED_ORIGINS` 與步驟 4 的授權來源填的都是它。
- 當初試過 Cloudflare Workers，但它的網址會掛在整個帳號共用的子網域底下
  （`….boo-king-king.workers.dev`），所以改用 GitHub Pages（2026-10-06）。

### 6. LINE Developers Console（小判官）

建一個 Provider → Messaging API channel（會連帶建一個官方帳號）。

- Basic settings → **Channel secret** → Render 的 `LINE_CHANNEL_SECRET`
- Messaging API → 最下面 Issue **Channel access token** → Render 的 `LINE_CHANNEL_ACCESS_TOKEN`
- Messaging API → Webhook URL 填 `https://<Render 網址>/line/webhook`，按 Verify，
  打開 **Use webhook**
- 官方帳號設定（LINE Official Account Manager）：
  - 打開「接受邀請加入群組或多人聊天室」
  - 關掉「自動回應訊息」與「加入好友的歡迎訊息」（不關的話 LINE 自己也會回話）
- 每位管理員都要加小判官好友：私訊它才能預覽指令，它要通知管理員時也才發得到。

### 7. 保活

cron-job.org 每 5～10 分鐘 GET `https://<Render 網址>/health`。這支不碰資料庫，不會
把 Neon 叫醒。

### 8. 驗證

```bash
curl https://<Render 網址>/health
```

回 `{"ok":true}` 就是後端起來了。`https://<Render 網址>/api/ready` 會列出每一項設定
有沒有填，以及現在跑的是哪個 commit（前 7 碼）——推了新版本之後 Render 的畫面
卡住的話，看這個數字換了沒。接著：

1. 開前端網址，用 `FIRST_ADMIN_EMAIL` 那個 Google 帳號登入，看得到版規全文。
2. **先開一個只有你自己的測試群組**把小判官拉進去（一個群組只能有一個官方帳號，
   正式群組要先把揪揪請出去才拉得進去）。在群組打「版規」，應該回一張卡片。
3. 後台「成員」頁籤最上面會提醒「名單裡還沒有對應到你的那一筆」。你在群組說過
   話之後，名單裡會出現你的 LINE 名稱：把那一筆的身分改成管理員、email 填你登入
   的帳號，提醒就會消失（**直接在後台改，不要用 SQL**）。再到群組 tag 小判官說
   「整理完畢」，應該回廣告須知。
4. 用電腦版 LINE 在測試群組說一句話，再看「成員」頁籤「認不出發話者的訊息」那一格
   寫的「統計區間之後到現在另有幾則」——有增加就是電腦版認不出人。（「未知發話」
   頁籤只列統計區間裡的訊息，今天發的不會出現在那張表上。）

### 9. 從測試群組搬到正式群組

小判官只認第一個有事件進來的群組，被拉進別的群組會自己退出。搬家照這個順序：

1. **先把小判官請出測試群組。** 還待在裡面的話，清完資料之後那邊只要有人說一句
   話，測試群組就又成了它服務的群組。
2. 在 Neon 的 SQL Editor 清掉測試資料：

   ```sql
   TRUNCATE users, speak_log, unknown_messages, kicked_list, whitelist, sessions RESTART IDENTITY CASCADE;
   DELETE FROM settings;
   ```

3. 把揪揪請出正式群組、把小判官拉進去。

版規與訊息內容在 `messages` 表，不會被清掉。清完之後後台會要你重新登入——
`FIRST_ADMIN_EMAIL` 那個帳號照樣進得來，進去之後照步驟 8 的第 3 點把自己設成管理員。
不必重啟服務。

### 10. 要直接改資料庫的時候

後端有一部分狀態記在記憶體裡：每個人這個月的發話紀錄寫過了沒、服務的是哪個
群組。用 SQL 直接改 `users`、`speak_log`、`settings` 之後它會自己發現並補救（細節在
`line_utils/tracker.py`、`line_utils/home_group.py` 開頭），但改完到 Render 按一次
Manual Deploy → Restart service 最乾淨。

★ **刪掉 `users` 的一列，那個人全部的發話紀錄與白名單紀錄會跟著一起刪。** 要把
  兩筆資料併成一筆的話，留下有 LINE 帳號的那一筆，把另一筆的 email 與身分搬過來，
  再刪沒有 LINE 帳號的那一筆——不要反過來。（2026-10-06 上線測試時就是反過來做，
  之後說的話整天都沒記到。當時 `FIRST_ADMIN_EMAIL` 會自動建一筆只有 email 的資料，
  才有兩筆要併；現在不建了。）

要在 LINE 裡把後台網址傳給其他管理員，後面加 `?openExternalBrowser=1`，LINE 會
改用手機的瀏覽器開。Google 不讓 App 內嵌的瀏覽器做登入，直接在 LINE 裡點開多半
會登不進去。

## 還沒定案的事

**先照解讀做了、等確認的：**

- **「區間後才發話來不及」的範圍**：8、9 月沒說話、10/3 才說話的人，10/6 整理時
  照樣上名單（旁邊標「10/3 才發話」）。
- **未知發話會存訊息開頭 60 個字**：這是整個系統唯一存內容的地方，不存就沒有
  線索可以指認。
- **發話對照的排序**：LINE 沒有公布規則，做了四種排法（LINE 手機版／電腦版／
  筆畫／注音）讓你挑，預設是手機版那種。
- **從電腦版 tag 小判官而 LINE 沒附上發話者時**，它回的是一句說明，不是
  「請不要跟我說話」。

**還沒看過實際樣子的：**

- **三張卡片（歡迎、版規、黑名單警告）在 LINE 上的樣子**：版面在
  `group-judge-backend/line_utils/cards.py`，`tools/show_card.py` 可以印出 JSON 貼到
  LINE 的 Flex Message Simulator 看。
