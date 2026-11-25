# 設計書: Google Cloud ビンゴアプリ

要件は `requirements.md`、全体像は `docs/spec.md` を参照。この文書は「どう作るか」を定める。要件番号は `R7-4` のように書く。

## 1. 設計方針

| 方針 | 内容 |
|---|---|
| ビルドなし | HTML + ES Modules。Firebase SDK と QR ライブラリは CDN から読む |
| ロジックは純関数に分離 | カード生成、列判定、ランキング集計、時間計算は、Firebase に依存しない純関数にする。Node の標準テストランナーでテストできる |
| 書き込みは最小 | 1タップ=`players` への1回の更新(R7-13) |
| 購読は最小 | 参加者はルーム、自分の `players`、`events` の3つだけ(R11-1) |
| サーバー検証なし | 判定はブラウザ。ランキングだけは、ホストが `card[]` と `marked[]` から再計算する(R15) |

## 2. 全体構成

```
ブラウザ(ホスト / 参加者)
  index.html + ES Modules(ビルドなし)
     │  Firebase JS SDK(CDN)
     ▼
  Firebase Authentication(匿名)   Firestore(Rules で防御)
     │                                 rooms / players / events
     ▼
  Firebase Hosting(静的配信のみ)
```

サーバー側のコードはない。Firestore Rules が唯一のサーバー側ロジックである。

## 3. ファイル構成

```
public/
  index.html                 単一ページ。noindex を含む
  robots.txt                 Disallow: /
  firebase-config.js         gitignore。実際の設定
  firebase-config.example.js
  css/style.css
  js/
    main.js                  起動、ハッシュルーター
    firebase.js              初期化、匿名サインイン、Firestore の export
    views/
      home.js                トップ(お題登録とルーム作成)
      host.js                ホスト画面
      play.js                参加者画面
    lib/                     純関数(Firebase に依存しない)
      topics.js              お題 JSON の検証
      bingo.js               カード生成、列判定、列数
      ranking.js             ランキング集計、同順位の検出
      timekeeper.js          時間配分、フェーズ、予備時間、終了予定
      prizes.js              景品数の検証
    data/
      rooms.js               Firestore の読み書き(ルーム、抽選、遷移、発表)
      players.js             Firestore の読み書き(入室、穴開け、ビンゴ)
      events.js              events の購読と作成
    ui/
      qr.js                  QR 生成
      confetti.js            紙吹雪
      toast.js               通知
  icons/                     gitignore。公式アイコン
samples/topics.sample.json   30件前後
tests/
  package.json               テスト用のみ(アプリ本体にはビルドなし)
  unit/                      lib/ の純関数のテスト
  rules/                     Emulator 上の Rules テスト
docs/
  spec.md  icons.md
firestore.rules  firestore.indexes.json  firebase.json
.firebaserc.example  README.md  LICENSE
```

- Node は **テストの実行にだけ** 使う。アプリ本体の配信にはビルド工程がない。
- `lib/` は `import` だけで動く純関数にし、ブラウザと Node の両方から読む。

## 4. ルーティング

| ハッシュ | 画面 |
|---|---|
| `#/` | トップ(ルーム作成の入口。ルーム一覧は出さない) |
| `#/host/:roomId` | ホスト画面 |
| `#/play/:roomId` | 参加者画面 |

`hashchange` で切り替える。画面ごとに購読(`onSnapshot` の解除関数)を保持し、遷移時に必ず解除する。

## 5. 画面設計

### 5.1 トップ(`views/home.js`)

1. 匿名サインイン(`signInAnonymously`)。
2. お題 JSON を貼り付け、またはファイル選択。`lib/topics.js` で必須フィールド、文字数(`id`:64、`name`:80、`desc`:200、`iconKey`:64)、ID重複、件数、UTF-8サイズを検証し、件数と一覧、エラー位置を表示(R1-1〜R1-5)。サイズは `TextEncoder` で測る。
3. 24個未満、100個超、または直列化後のUTF-8サイズが256 KiB超なら「ルーム作成」を無効にし、理由を表示する(R1-4、R1-5)。
4. タイトルを入力して作成。`addDoc(rooms)` で自動 ID を得る(R2-1)。作成後に `#/host/:roomId` へ移動する。

### 5.2 ホスト画面(`views/host.js`)

`rooms/{roomId}` を取得し、`hostUid !== auth.uid` なら参加用画面への案内だけを出す(R2-5)。

状態によって、表示を3段階に切り替える。

| 段階 | 条件 | 表示 |
|---|---|---|
| 準備 | `open` かつ `schedule.startedAt` が空 | QR、景品数の入力、時間配分の変更、「開始」 |
| 進行 | `open` かつ開始済み | タイムキーパー、抽選、QR、達成者、参加者 |
| 発表 | `ranking` | 集計、同順位の並べ替え、発表、終了 |
| 終了 | `closed` | 結果の閲覧のみ |

進行の画面構成(スマホ縦、または会場用の横画面)。

```
┌──────────────────────────────┐
│ タイムキーパー                │ 経過/残り・フェーズ・予備・終了予定
├──────────────────────────────┤
│ [抽選結果の大表示 / QR]  切替 │ 抽選済みお題を大きく
│ [ 抽選 ]   1分トークタイマー  │
├──────────────────────────────┤
│ 抽選済み一覧 | 参加者 | 達成者 │ タブ
└──────────────────────────────┘
```

### 5.3 参加者画面(`views/play.js`)

| 段階 | 表示 |
|---|---|
| ルームなし | 「ルームが見つかりません」(R6-6) |
| 未入室で `open` | 名前入力(1〜20文字)。送信でカード作成(R6-2〜R6-4) |
| 未入室で `open` 以外 | 入室不可の案内(R6-7) |
| 入室済み | ヘッダー(名前、識別コード)、状態表示、タブ(カード、抽選済み一覧、ランキング) |

- ランキングタブは `ranking` 以降のみ表示する(R17-6)。
- 残り時間は表示しない(R7-15)。

## 6. データモデル

### 6.1 `rooms/{roomId}`

| フィールド | 型 | 内容 |
|---|---|---|
| `hostUid` | string | ホストの UID |
| `title` | string | ルーム名 |
| `status` | string | `open` / `ranking` / `closed` |
| `topics` | array of map | `{id, name, desc, iconKey}`、24〜100個、直列化後のUTF-8サイズは256 KiB以下 |
| `drawn` | array of string | 抽選済みお題の id(抽選順) |
| `createdAt` | timestamp | サーバー時刻 |
| `schedule` | map | `{startedAt: timestamp or null, introMin, drawMin, rankingMin, bufferMin}`。既定は 3, 20, 5, 2 |
| `prizes` | map | `{double, single}`。既定は 5, 15。合計20以上 |
| `rankingStartedAt` | timestamp | 「抽選終了」の時刻(任意) |
| `result` | map | `{entries: [{rank, category, uid6, name}]}`。発表のたびに末尾へ追加 |

### 6.2 `rooms/{roomId}/players/{uid}`

| フィールド | 型 | 内容 |
|---|---|---|
| `name` | string | 1〜20文字(前後の空白は除去済み) |
| `card` | array of string | 25要素。お題 id または `"FREE"`。インデックス12が FREE |
| `marked` | array of number | 開けたマスの番号。**12(FREE)は含めない**。FREE は常に開いているとして扱う |
| `joinedAt` | timestamp | サーバー時刻 |
| `bingoAt` | timestamp | 初めて1列揃った時刻(任意) |
| `bingoCount` | number | 揃っている列数(任意) |

### 6.3 `rooms/{roomId}/events/{uid}`

| フィールド | 型 | 内容 |
|---|---|---|
| `type` | string | `"bingo"` |
| `uid` | string | 参加者の UID(文書 ID と同じ) |
| `playerName` | string | 名前 |
| `at` | timestamp | サーバー時刻 |

## 7. 主要な処理

### 7.1 認証

`firebase.js` が `onAuthStateChanged` で UID の確定を待ち、確定後に画面を描画する。UID は SDK が IndexedDB に保持するので、リロードしても同じ UID になる(R6-5)。

### 7.2 カード生成(`lib/bingo.js`)

1. お題 id の配列を Fisher-Yates でシャッフルし、先頭24個を取る。
2. インデックス12に `"FREE"` を挿入して25要素にする。
3. 入室時に `setDoc(players/{uid})` で1回だけ保存する。すでに文書があれば再生成しない(R6-8)。

### 7.3 列の判定(`lib/bingo.js`)

12本の列(横5、縦5、斜め2)のインデックスを定数で持つ。

```js
const LINES = [
  [0,1,2,3,4],[5,6,7,8,9],[10,11,12,13,14],[15,16,17,18,19],[20,21,22,23,24],
  [0,5,10,15,20],[1,6,11,16,21],[2,7,12,17,22],[3,8,13,18,23],[4,9,14,19,24],
  [0,6,12,18,24],[4,8,12,16,20],
];
// open(i) = i === 12 || marked.includes(i)
// countLines(marked) = 全マスが open の列の数
```

### 7.4 穴開け(`data/players.js`)

1. タップしたマスの `card[i]` が `drawn` に含まれるか確認する。含まれなければ短いメッセージを出して終了(R7-5)。
2. `marked` に `i` を足した配列から `newCount = countLines(...)` を計算する。
3. 分岐する。
   - `newCount === 0` または2列目以降: `updateDoc(players/{uid}, { marked: arrayUnion(i), bingoCount: newCount })` の **1回** で済ませる(R7-13、R8-8)。
   - 初めて1列揃った(`bingoAt` が未設定かつ `newCount >= 1`): `writeBatch` で `players` の更新(`marked`、`bingoCount`、`bingoAt: serverTimestamp()`)と、`events/{uid}` の作成(`type`, `uid`, `playerName`, `at: serverTimestamp()`)を **同時に** 書く。片方だけが残ることを防ぐ(R8-2)。
4. 画面の演出と列数表示は、ローカルの計算結果で即時に更新する。

`status` が `open` でない場合は、タップを無効にする(R7-8)。

### 7.5 ビンゴ通知(`data/events.js`)

- 入室済みで `joinedAt` が確定した後に、`query(events, where('at', '>', joinedAt))` を購読する。入室前の過去のビンゴは読まない。
- **最初のスナップショットは通知に使わない。** 2回目以降の `docChanges()` の `added` だけを通知にする(R8-6)。リロード後も、すでにあるイベントは再表示されない。
- 自分のイベントには、紙吹雪を出す。他人のイベントには「○○さんがビンゴ!」を出す(R8-4、R8-5)。

### 7.6 抽選(`data/rooms.js`)

```js
await runTransaction(db, async (tx) => {
  const snap = await tx.get(roomRef);
  const room = snap.data();
  if (room.status !== 'open' || !room.schedule.startedAt) throw new Error('not-drawable');
  const remaining = room.topics.filter(t => !room.drawn.includes(t.id));
  if (remaining.length === 0) throw new Error('no-topics');
  const pick = remaining[Math.floor(Math.random() * remaining.length)];
  tx.update(roomRef, { drawn: [...room.drawn, pick.id] });
  result = pick;
});
```

- 複数タブから同時に押されると、片方がトランザクションの競合で再試行され、順番に確定する(R3-7)。
- 実行中は「抽選」ボタンを無効にする(R3-8)。
- 抽選結果は、書き込み後の `onSnapshot` 経由の `drawn` の最後の要素から表示する。

### 7.7 時間管理(`lib/timekeeper.js`)

時間の入力は `schedule` と、現在時刻 `now`、抽選回数 `drawnCount`、`status`、`rankingStartedAt` だけ。

| 値 | 計算 |
|---|---|
| 経過 | `now − startedAt` |
| 計画の抽選終了 | `introMin + drawMin`(開始からの分) |
| 現在のフェーズ | `status === 'ranking'` なら発表。それ以外は、経過が `introMin` 未満なら導入、それ以降なら抽選 |
| フェーズの残り | 導入と抽選は計画の境界までの残り。発表は `rankingStartedAt + rankingMin − now` |
| 遅れ | 抽選中で、経過が計画の抽選終了を超えた分。`ranking` の場合は、`rankingStartedAt` の計画からの遅れと、発表の超過分の合計 |
| 予備の残り | `bufferMin − 遅れ`。負なら予備を使い切ったことを示す |
| 抽選の目安 | `drawMin` 回(1回約1分) |
| 終了予定 | 抽選中は、抽選済み回数から求めた1回あたりの所要時間で、目安の回数まで続けた場合の抽選終了時刻を求め、計画の抽選終了のうち遅いほうに `rankingMin` を足す。抽選が0回の間は計画どおりとする |
| 警告 | 終了予定が `startedAt + (introMin + drawMin + rankingMin + bufferMin)` を超えれば警告色。フェーズの残りが5分以下で黄、2分以下で赤(R13-6) |

- 1秒ごとに再計算して表示する。計算は `now` を引数に取る純関数にして、テストでは任意の時刻を渡す。
- 時刻は端末の `Date.now()` と、Firestore の `startedAt`(サーバー時刻)の差で求める。数秒のずれは許容する(R13-8)。
- 自動遷移はしない。条件を満たすと、ホスト画面に提案を出す(R13-7、R14)。

### 7.8 抽選終了の提案(R14)

ホスト画面が、次のいずれかを検知すると提案を出す。

1. 達成者数(`bingoCount >= 1` の `players` の数)が `prizes.double + prizes.single` 以上。
2. 抽選フェーズの時間が尽きた(経過が計画の抽選終了に達した)。

ホストは提案とは無関係に、いつでも「抽選終了」を押せる(R14-3)。押すと `updateDoc(room, {status: 'ranking', rankingStartedAt: serverTimestamp()})` を書く。

### 7.9 ランキング集計(`lib/ranking.js`)

入力は `drawn`、`players`(`uid`, `name`, `card`, `marked`)、`prizes`。出力は、順位つきの並びと同順位の組。

1. **抽選回の表**: `drawn` の位置から、お題 id → 抽選回(1始まり)を作る。
2. **有効な列**: 各列について、全マスが「FREE」または「`marked` に含まれ、かつ抽選回の表にある」なら有効とする。
3. **完成回**: 有効な列について、各マスの抽選回(FREE は0)の最大値。
4. **列数と到達回**: 有効な列の数が `count`。完成回を昇順に並べた `count` 番目(=最大値)が、その列数に届いた回(`reach`)。`count === 0` は対象外。
5. **整列**: `count` の降順、同数なら `reach` の昇順。
6. **同順位**: `count` と `reach` が同じ参加者を1組にする。
7. **カテゴリへの割り当て**:
   1. 並び(同順位を解消した後)の上から、`count >= 2` の人を、`prizes.double` 人までダブルビンゴ以上賞に入れる。
   2. ダブルビンゴ以上賞に入らなかった参加者を、並びの順のまま `prizes.single` 人まで、ビンゴ賞に入れる。ダブル以上の人のあふれ分は、列数が多いので、ビンゴ賞の上位に入る(R15-9)。
   3. 残りは賞外。
8. 各カテゴリの中で、1位から順位を振る。

同順位の解消が必要なのは、**賞の対象範囲(ダブル賞とビンゴ賞の人数)にかかる組だけ** とする。賞外の順位の組は、じゃんけんしなくてよい。

### 7.10 同順位の解消と発表(`views/host.js`)

- ホスト画面は、`ranking` になった時点の全員の `players`(購読済み)から集計する。追加の読み取りはない。
- じゃんけんの結果は、ホストの上下ボタンで組の中の順番を入れ替えて決める。入れ替えた順は、ホスト端末の `localStorage`(キー `gcpbingo:tie:<roomId>`)に保存する。Firestore には書かない。ホストがリロードしても、じゃんけんをやり直さなくてよい。
- 解消が必要な組が残っている間は、「次を発表」を無効にする(R16-4)。
- 発表の順は、ビンゴ賞の下位 → 1位、続いてダブルビンゴ以上賞の下位 → 1位(R17-2)。
- 「次を発表」で、次の1人を `updateDoc(room, { 'result.entries': arrayUnion(entry) })` で書く(R17-3)。`entry` は `{rank, category, uid6, name}`。同じ内容を2回送っても `arrayUnion` が重複を除く。
- 発表済みの件数は `result.entries.length` から分かるので、ホストがリロードしても、続きから発表できる。`ranking` 中は `players` が更新されないので、再計算しても結果は変わらない。
- 未発表の順位は、Firestore に書かない(R17-4)。
- 発表が終わると「終了」を押せる。`closed` に更新する(R5-3)。

### 7.11 QR(`ui/qr.js`)

参加用 URL(`location.origin + '/#/play/' + roomId`)を、ブラウザ側で QR にする。全画面ボタンで画面いっぱいに拡大し、URL を文字でも併記する(R2-7)。ライブラリは、ES Module として CDN から読めて小さいものを実装時に選ぶ(例: `qrcode-generator`)。バージョンを固定して読み込む。

## 8. Firestore Rules

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function signedIn() { return request.auth != null; }
    function room(roomId) { return get(/databases/$(database)/documents/rooms/$(roomId)).data; }
    function isHost(roomId) { return signedIn() && room(roomId).hostUid == request.auth.uid; }
    function isOpen(roomId) { return room(roomId).status == 'open'; }

    function validTransition(before, after) {
      return before == 'open'    && after in ['open', 'ranking', 'closed']
          || before == 'ranking' && after in ['ranking', 'closed'];
    }

    match /rooms/{roomId} {
      allow get: if signedIn();
      allow list: if false;
      allow create: if signedIn()
        && request.resource.data.hostUid == request.auth.uid
        && request.resource.data.status == 'open'
        && request.resource.data.drawn.size() == 0
        && request.resource.data.topics.size() >= 24
        && request.resource.data.topics.size() <= 100;
      allow update: if signedIn()
        && resource.data.hostUid == request.auth.uid
        && request.resource.data.hostUid == resource.data.hostUid
        && validTransition(resource.data.status, request.resource.data.status);
      allow delete: if false;

      match /players/{uid} {
        allow get: if signedIn() && (request.auth.uid == uid || isHost(roomId));
        allow list: if isHost(roomId);
        allow create: if signedIn() && request.auth.uid == uid && isOpen(roomId)
          && request.resource.data.name is string
          && request.resource.data.name.size() >= 1
          && request.resource.data.name.size() <= 20
          && request.resource.data.card.size() == 25;
        allow update: if signedIn() && request.auth.uid == uid && isOpen(roomId)
          && request.resource.data.diff(resource.data).affectedKeys()
               .hasOnly(['marked', 'bingoCount', 'bingoAt']);
        allow delete: if false;
      }

      match /events/{eventId} {
        allow read: if signedIn();
        allow create: if signedIn() && isOpen(roomId)
          && eventId == request.auth.uid
          && request.resource.data.uid == request.auth.uid
          && request.resource.data.type == 'bingo'
          && request.resource.data.at == request.time;
        allow update, delete: if false;
      }
    }
  }
}
```

- `allow list` を `rooms` で禁止し、`get` だけ許可する(R10-1)。
- `players` の更新は `marked`、`bingoCount`、`bingoAt` のキーだけに限る。`name` と `card` を書き換えて、カードを作り直すことを防ぐ(R6-8)。これは要件の「作成・更新は本人のみ」を、同じ方向で厳しくしたもの。
- `marked[]` の中身は検証しない(R10-12)。
- `events` の ID を UID にしているので、同じ ID への2回目の `create` は、更新扱いになって拒否される(R10-6)。
- `ranking` と `closed` では `isOpen` が偽になり、`players` と `events` の書き込みが拒否される(R5-4)。
- `rooms` の `update` は、`closed` からは遷移できない。`ranking` から `open` へも戻れない(R5-1)。
- `isHost` と `isOpen` は `get()` を呼ぶ。Rules 内の `get()` も読み取りとして課金される(§10)。

### Rules テスト(`tests/rules/`)

`@firebase/rules-unit-testing` を、Firebase Emulator Suite 上で実行する。R10-11 の項目に加えて、次を含める。

| テスト | 期待 |
|---|---|
| 未認証の `rooms` 取得 | 拒否 |
| 認証済みの `rooms` 取得 | 許可 |
| `rooms` の list | 拒否 |
| ホスト以外の `rooms` 更新 | 拒否 |
| ホストの `open → ranking → closed` | 許可 |
| お題が24個未満または100個超の `rooms` 作成 | 拒否 |
| ホストの `open → closed` | 許可 |
| ホストの `ranking → open`、`closed → open` | 拒否 |
| 本人の `players` 作成(`open`) | 許可 |
| 他人の UID での `players` 作成 | 拒否 |
| `players` の `card` と `name` の更新 | 拒否 |
| 他人の `players` 読み取り | 拒否 |
| ホストの `players` 読み取り(get と list) | 許可 |
| 参加者の `players` list | 拒否 |
| `ranking` 中と `closed` 中の `players` 書き込み | 拒否 |
| `ranking` 中の `events` 作成 | 拒否 |
| 本人の UID での `events` 作成(`open`) | 許可 |
| 他人の UID での `events` 作成 | 拒否 |
| 同じ UID での `events` 2回目の作成 | 拒否 |
| `events` の更新と削除 | 拒否 |
| 認証済みの `events` 読み取り | 許可 |

## 9. 購読

| 画面 | 購読 |
|---|---|
| 参加者 | `rooms/{roomId}`、`players/{uid}`、`events`(`at > joinedAt`) |
| ホスト | `rooms/{roomId}`、`players`(全員)、`events`(開始以降) |

- 参加者の `events` の購読は、入室後に始める。入室前は `rooms/{roomId}` だけ購読する。
- 全員の `players` の購読は、ホスト画面のみ(R11-2)。

## 10. 無料枠の見積もり(R11-3)

Spark の日次上限は、読み取り 50,000、書き込み 20,000(いずれも前提値。実装時に公式の最新値を確認する)。100人、30分、全員が熱心に遊んだ **悪い側の想定** で数える。

### 読み取り

| 項目 | 回数 | 根拠 |
|---|---|---|
| ルーム文書の配信 | 約4,600 | 更新約45回(抽選20、発表約20、開始、集計前後など)× 100人 + 初回100 |
| 自分の `players` | 約100 | 初回のみ。自分の書き込みは、ローカルで反映され、課金されない前提 |
| `events` の配信 | 約10,000 | 最大100件のビンゴが、100人に届く |
| Rules の `get()` | 約2,700 | `players` の書き込み約2,600回(入室100、穴開け約2,500)と、`events` 作成100回に、各1回 |
| ホストの `players` 購読 | 約3,000 | 初回100、以後は書き込みのたびに1回ずつ |
| 合計 | 約20,400 | 上限の約4割 |

### 書き込み

| 項目 | 回数 |
|---|---|
| `players` の作成 | 100 |
| 穴開け | 約2,500(1人25回) |
| `events` | 最大100 |
| `rooms` の更新 | 約45 |
| 合計 | 約2,750(上限の約14%) |

- 通信が切れて再接続すると、購読が再読み取りされる場合がある。この分は余裕に含める。
- **同じ日に何度も練習すると、同じ日次の枠を使う。** 本番当日は、事前の練習を別の日にするか、練習の規模を小さくする。
- 見積もりは、デモの会(12/11の週)に、Firebase コンソールの使用状況で実測し、差があれば設計を見直す。

## 11. 時間配分とお題数のシミュレーション

要件の抽選目安は20回。お題の数と、達成者の出るタイミングが合うかを、モンテカルロで確認した(100人、カードは各自24個ランダム、全員が抽選されたお題を即座に開ける理想的な場合。20試行の平均)。

| お題数 | 10回時点 | 15回時点 | 20回時点 | 達成者が20人に届く回 |
|---|---|---|---|---|
| 30 | 4人(ダブル以上0) | 34人(同4) | 86人(同49) | 約14回 |
| 40 | 2人(同0) | 9人(同0) | 34人(同6) | 約18回 |
| 50 | 1人(同0) | 4人(同0) | 14人(同1) | 約23回 |
| 60 | 0人(同0) | 2人(同0) | 6人(同0) | 約27回 |

表は「達成者 (うち2列以上)」の人数。

読み取れること。

- **お題が30個だと、14回で景品数(20)に届き、終了の提案が出る。** その時点のダブル以上は数人で、ダブル以上賞の既定5を満たさない場合がある。
- **お題が40個前後だと、20回で達成者が約34人、ダブル以上が約6人。** 抽選20回の目安と、景品数20、ダブル以上賞5が、おおむね釣り合う。
- お題が50個以上だと、20回では達成者が景品数に届かない。時間切れの提案で終わる。
- 実際は、参加者がすぐには開けないので、数値はさらに遅れる側にずれる。

**推奨:** お題は **35〜40個** を用意する。サンプル JSON は30件前後なので、本番用に増やす。景品数は、お題数に合わせてホストが開始前に調整できる(R18)。

## 12. エラー処理

| 状況 | 動作 |
|---|---|
| 匿名認証の失敗 | 画面にエラーと再試行ボタンを出す |
| `firebase-config.js` がない | 設定手順を示す案内を出す |
| 抽選のトランザクション競合 | SDK が自動再試行。失敗が続けばエラーを出してボタンを戻す |
| 通信断 | 画面に「接続を確認中」を表示。復旧すれば購読が自動で再開する |
| Rules に拒否された書き込み | 状態(`ranking` など)が変わった可能性が高い。画面の状態表示を更新し、短いメッセージを出す |
| お題 JSON の不正 | 位置つきのエラーを表示(R1-2) |

## 13. テスト方針

| 種類 | 対象 | 実行 |
|---|---|---|
| 単体 | `lib/` の純関数(`topics`、`bingo`、`ranking`、`timekeeper`、`prizes`) | `node --test tests/unit` |
| Rules | `firestore.rules` | `firebase emulators:exec --only firestore "node --test tests/rules"` |
| 手動 | 画面操作、ホストと参加者の連携、スマホ実機 | デモの会の前に、チェックリストで確認する |

`topics.js` の単体テストで、24個未満、100個超、各フィールドの文字数超過、重複ID、UTF-8サイズ256 KiB超を拒否することを確認する。

`ranking.js` の単体テストで、次を確認する。

- 抽選されていないマスを開けても、列として数えない。
- 完成回は、タップの速さではなく抽選回で決まる。
- ダブル以上賞のあふれが、ビンゴ賞の上位に入る。
- 1人が複数のカテゴリに出ない。
- 同順位の組が検出される。
- 賞外の組は、解消が不要として扱われる。

## 14. 要件への追加・変更(承認・反映済み)

設計の過程で出た要件の細部を、承認のうえ `requirements.md` に反映した。

| # | 内容 | 提案 |
|---|---|---|
| 1 | 同順位の解消の範囲(R16-4) | 賞の対象範囲にかかる組だけ解消が必要とし、賞外の組は不要にする。じゃんけんの回数を減らせる |
| 2 | `players` の更新範囲(R10-4) | `marked`、`bingoCount`、`bingoAt` だけを更新可とする。`card` と `name` の書き換えを防ぎ、R6-8 を Rules でも守る |
| 3 | `marked` と FREE | `marked` にインデックス12を含めない。FREE は常に開いているとして扱う |
| 4 | `events` の購読範囲 | `at > joinedAt` で購読し、入室前のビンゴを読まない。読み取りを減らし、R8-6 も満たす |
| 5 | じゃんけん結果の保存 | ホスト端末の `localStorage` に保存し、Firestore には書かない |
| 6 | Node の利用 | テスト用のみ。アプリ本体のビルドは不要のまま |
| 7 | 推奨お題数 | 35〜40個。サンプル JSON は30件前後のままにし、README で案内する |

## 15. 実装の順序(tasks.md の下書き)

1. リポジトリの土台(`firebase.json`、`.gitignore`、`firebase-config.example.js`、`LICENSE`、`README` の骨子)
2. 秘密情報保護(`.gitleaks.toml`、共通スキャン、pre-commit / pre-push、secret-scan CI、運用書)
3. 純関数(`topics`、`bingo`、`prizes`)と単体テスト
4. Firestore Rules と Rules テスト
5. トップ(お題登録、ルーム作成)
6. 参加者画面(入室、カード、穴開け、抽選済み一覧)
7. ホスト画面(QR、抽選、トランザクション)
8. ビンゴ判定、`events`、通知、列数表示、達成者一覧
9. タイムキーパー(`timekeeper` と表示)、景品数の設定、抽選終了の提案
10. ランキング(`ranking`、同順位、発表、参加者のランキングタブ)
11. アイコン、プレースホルダー、フッター、`noindex`、`docs/icons.md`
12. README の仕上げ、サンプル JSON、リハーサル(無料枠の実測)

## 16. 秘密情報検査とCI

### 16.1 ファイル構成

```text
.gitleaks.toml                    Gitleaksの追加ルールと最小限の許可設定
.githooks/pre-commit             ステージ済み変更を検査
.githooks/pre-push               リモートとの差分を再検査
.github/workflows/secret-scan.yml PRとpushで検査
scripts/secret-scan.sh           ローカルとCIの共通エントリーポイント
scripts/setup-git-hooks.sh       core.hooksPathを設定
docs/security.md                 開発者向け運用手順
```

### 16.2 共通スキャン

`scripts/secret-scan.sh` を検査の正本にし、ローカルフックとGitHub Actionsが同じ処理を呼ぶ。

1. 固定バージョンのGitleaksを実行する。
2. 次のコミット禁止ファイルを検査する: `firebase-config.js`、`.firebaserc`、`.env`、`.env.*`(exampleを除く)、`*-service-account*.json`、`application_default_credentials.json`、`*.pem`、`*.key`。
3. GCP固有の危険な特徴を検査する: `private_key` と `client_email` を含むサービスアカウントJSON、秘密鍵ヘッダー、`GOOGLE_APPLICATION_CREDENTIALS`、`FIREBASE_TOKEN`。
4. 検出時は値をログへ出さず、Gitleaksのredact機能などでマスクして終了コードを非0にする。

Gitleaksの具体的なバージョンとCLI引数は、実装時に公式ドキュメントで確認して固定する。ローカルとCIで同じバージョンを使う。

### 16.3 ローカルフック

- pre-commitはステージ済み変更を検査し、秘密情報がコミットへ入る前に止める。
- pre-pushはリモートとの差分を検査し、導入前のコミットやpre-commitの設定漏れを補う。
- `scripts/setup-git-hooks.sh` が `git config core.hooksPath .githooks` を設定する。
- `--no-verify` で回避できるため、ローカルフックは必須チェックの代わりにしない。

### 16.4 GitHub Actions

`secret-scan.yml` は `pull_request` と `push` で実行する。履歴を省略せず取得し、PRまたはpushに含まれるコミットを検査する。

- secret scanをPRの必須チェックにする。
- 自動デプロイは行わない(R12-6)。
- 意図的なダミー秘密情報を使った検査テストを用意し、ジョブが失敗することを確認する。実在する形式の有効な認証情報は使わない。
- 誤検知は値を変更することを優先し、必要な場合だけ `.gitleaks.toml` へ最小範囲の許可と理由を記録する。広範囲な除外やbaselineでの一括無視はしない。

### 16.5 GitHubとGoogle Cloudの設定

- GitHub Secret ScanningとPush Protectionを有効化し、`docs/security.md` のチェックリストで確認する。
- Firebase Web APIキーは公開される前提とし、Google Cloud側で利用可能APIとFirebase HostingドメインのHTTPリファラーを必要最小限に制限する。
- 将来CIデプロイを追加する場合は、GitHub OIDC → Workload Identity Federation → 最小権限サービスアカウントの順で短時間認証する。サービスアカウントJSONは作成・保存しない。

### 16.6 漏えい時対応

リモートへ出た認証情報は漏えい済みとして扱う。最初に認証情報を無効化・ローテーションし、その後にログと露出範囲を確認し、必要ならGit履歴を修正する。履歴の削除だけで安全になったとは判断しない。
