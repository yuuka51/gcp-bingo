# Google Cloud ビンゴ 構成図・DFD・フローチャート

この文書は、`requirements.md` と `design.md` の内容を図で確認するための資料です。

> 現在のリポジトリは仕様・設計段階です。以下は、実装後に採用する構成を表しています。Cloud Functions / Cloud Run などのサーバー処理はありません。

## 1. インフラ構成図

```mermaid
flowchart LR
  subgraph Clients[利用者のブラウザ]
    H[ホスト画面<br/>HTML + JavaScript ES Modules]
    P1[参加者画面<br/>スマートフォン]
    P2[参加者画面<br/>最大約100人]
  end

  CDN[CDN<br/>Firebase SDK / QRライブラリ]

  subgraph Firebase[Firebase Spark プラン]
    Hosting[Firebase Hosting<br/>静的ファイル配信]
    Auth[Firebase Authentication<br/>匿名認証 / UID発行]
    Rules[Firestore Security Rules<br/>認証・権限・状態を検証]
    DB[(Cloud Firestore<br/>rooms / players / events)]
  end

  H -->|HTTPS: index.html / JS / CSS| Hosting
  P1 -->|HTTPS: index.html / JS / CSS| Hosting
  P2 -->|HTTPS: index.html / JS / CSS| Hosting

  H -.->|ES Modules| CDN
  P1 -.->|ES Modules| CDN
  P2 -.->|ES Modules| CDN

  H -->|匿名サインイン| Auth
  P1 -->|匿名サインイン| Auth
  P2 -->|匿名サインイン| Auth

  H -->|ホスト操作・リアルタイム購読| Rules
  P1 -->|本人の操作・リアルタイム購読| Rules
  P2 -->|本人の操作・リアルタイム購読| Rules
  Auth -->|認証済みUID| Rules
  Rules -->|許可された読み書きのみ| DB

  Notes[サーバー処理なし<br/>Functions / Cloud Run なし]
  Notes -.-> Hosting
```

### コンポーネントの役割

| コンポーネント | 役割 |
|---|---|
| Firebase Hosting | HTML、JavaScript、CSS、プレースホルダー画像などを静的配信 |
| Firebase Authentication | 匿名UIDを発行し、ホストと参加者を識別 |
| Cloud Firestore | ルーム、カード、穴の状態、ビンゴ通知、発表済み結果を保存・配信 |
| Firestore Security Rules | ホストだけがルームを更新できること、参加者が本人のデータだけを書けること、締切後に書けないことを保証 |
| ブラウザ | カード生成、ビンゴ判定、時間計算、ランキング集計、QR生成を実行 |
| CDN | バージョン固定したFirebase SDKとQR生成ライブラリを配信 |

## 2. DFD(データフロー図)

### 2.1 コンテキストDFD

システム全体と外部実体の関係を先に示します。

```mermaid
flowchart LR
  H["ホスト"]
  U["参加者"]
  APP(("ブラウザアプリ"))
  AUTH[("Firebase Authentication<br/>匿名UID")]
  FS[("Cloud Firestore<br/>rooms / players / events")]
  LS[("ホスト端末 localStorage<br/>じゃんけん順")]

  H <-->|"進行操作 / 結果表示"| APP
  U <-->|"入室・穴開け / カード・通知・結果"| APP
  APP <-->|"匿名サインイン / UID"| AUTH
  APP <-->|"Rulesを通した読み書き・購読"| FS
  APP <-->|"同順位の並べ替え"| LS
```

### 2.2 詳細DFD

矢印のラベルを短いフローIDにし、線の交差と長文による崩れを抑えています。

```mermaid
flowchart LR
  subgraph EXT["外部実体"]
    direction TB
    H["ホスト"]
    U["参加者"]
  end

  subgraph PROC["ブラウザ内の処理"]
    direction TB
    P1(("P1 匿名認証"))
    P2(("P2 ルーム作成・進行"))
    P3(("P3 入室・カード管理"))
    P4(("P4 抽選・穴開け・ビンゴ"))
    P5(("P5 ランキング・発表"))
  end

  subgraph STORE["データストア"]
    direction TB
    D1[("D1 Firebase Auth")]
    D2[("D2 rooms / roomId")]
    D3[("D3 players / uid")]
    D4[("D4 events / uid")]
    D5[("D5 host localStorage")]
  end

  H -->|F1| P1
  U -->|F1| P1
  P1 <-->|F2| D1

  H <-->|F3| P2
  P2 <-->|F4| D2

  U <-->|F5| P3
  P3 <-->|F6| D3

  H <-->|F7| P4
  U <-->|F8| P4
  P4 <-->|F9| D2
  P4 -->|F10| D3
  P4 -->|F11| D4
  D4 -->|F12| P4

  H <-->|F13| P5
  D2 -->|F14| P5
  D3 -->|F15| P5
  P5 <-->|F16| D5
  P5 -->|F17| D2
  D2 -->|F18| U
```

| フロー | 内容 |
|---|---|
| F1 | ページを開き、匿名認証を開始 |
| F2 | 匿名サインインとUIDの受け取り |
| F3 | お題、時間、景品数、開始・終了操作と進行表示 |
| F4 | ルームの作成・更新、状態・抽選結果の購読 |
| F5 | 名前の入力、カードと識別コードの表示 |
| F6 | カード、穴の状態、列数の保存・復元 |
| F7 | 抽選操作と達成者一覧 |
| F8 | 抽選済みマスのタップ、穴の表示、ビンゴ通知 |
| F9 | `drawn[]` のトランザクション更新と購読 |
| F10 | `marked`、`bingoCount`、`bingoAt` の更新 |
| F11 | 初回ビンゴイベントを1件作成 |
| F12 | 新規ビンゴイベントを通知へ変換 |
| F13 | 集計、じゃんけん結果、発表操作とランキング画面 |
| F14 | 抽選順、状態、景品数、発表済み結果を読む |
| F15 | 全参加者のカードと穴の状態を読む |
| F16 | 同順位の並べ替えをホスト端末へ保存・復元 |
| F17 | 発表した1人だけを `result.entries` へ追加 |
| F18 | 発表済み順位だけを参加者へ配信 |


### 主なデータと公開範囲

| データ | 書く人 | 読める人 |
|---|---|---|
| `rooms/{roomId}` | ホストのみ | ルームIDを知る認証済みユーザー |
| `players/{uid}` | 本人のみ(`open`中) | 本人とホスト |
| `events/{uid}` | 本人が初回ビンゴ時に1回だけ | 認証済みユーザー |
| じゃんけん順 | ホスト端末のみ | ホスト端末のみ。Firestoreには保存しない |
| 未発表ランキング | Firestoreには保存しない | ホストのブラウザ内だけ |
| 発表済みランキング | ホストが1人ずつ `rooms.result` に追加 | 参加者を含む認証済みユーザー |

## 3. イベント全体のフローチャート

```mermaid
flowchart TD
  A([ホストがアプリを開く]) --> B[お題JSONを読み込む]
  B --> C{有効なお題が<br/>24個以上?}
  C -- いいえ --> C1[エラーと不足数を表示] --> B
  C -- はい --> D[ルームを作成]
  D --> E[参加URLとQRを会場に表示]
  E --> F[景品数を設定<br/>ダブル以上賞 + ビンゴ賞]
  F --> G{景品数の合計が<br/>20以上?}
  G -- いいえ --> G1[開始を無効化し<br/>不足数を表示] --> F
  G -- はい --> H[参加者がQRから入室<br/>名前入力・カード作成]
  H --> I[ホストが開始<br/>30分の時計を開始]

  I --> J[ホストが抽選]
  J --> K[未抽選のお題を<br/>トランザクションで1つ確定]
  K --> L[参加者に抽選結果を配信]
  L --> M{参加者がカードに<br/>該当マスを持つ?}
  M -- いいえ --> Q
  M -- はい --> N[参加者が該当マスをタップ]
  N --> O[穴の状態と列数を保存]
  O --> P{初めて1列揃った?}
  P -- はい --> P1[本人に紙吹雪<br/>全員にビンゴ通知]
  P -- いいえ --> P2[ダブル以上なら<br/>本人とホストの列数を更新]
  P1 --> Q
  P2 --> Q

  Q{終了条件?}
  Q -- まだ --> J
  Q -- 景品数到達 または 時間切れ --> R[ホストへ抽選終了を提案]
  R --> S{ホストが終了を承認?}
  S -- 続ける --> J
  S -- 終了 --> T[status = ranking<br/>入室と穴開けを締切]

  T --> U[全員のカードと穴を再検証]
  U --> V[列数優先・同数は完成回順で集計]
  V --> W{賞の対象範囲に<br/>同順位がある?}
  W -- はい --> X[会場でじゃんけん<br/>ホストが順番を反映] --> W
  W -- いいえ --> Y[ビンゴ賞の下位から<br/>1人ずつ発表]
  Y --> Z[ダブル以上賞の下位から<br/>1人ずつ発表]
  Z --> AA[status = closed]
  AA --> AB([イベント終了])

  I -. 中止時は発表せず終了可 .-> AA
```

## 4. マスをタップした時のフローチャート

```mermaid
flowchart TD
  A([参加者がマスをタップ]) --> B{status は open?}
  B -- いいえ --> B1[操作を受け付けない]
  B -- はい --> C{FREE?}
  C -- はい --> C1[常に開いているため<br/>書き込みなし]
  C -- いいえ --> D{お題IDが<br/>rooms.drawn にある?}
  D -- いいえ --> D1[穴を開けず<br/>未抽選メッセージ]
  D -- はい --> E{すでに marked?}
  E -- はい --> E1[何もしない]
  E -- いいえ --> F[marked に追加して<br/>12本の列を再計算]
  F --> G{初めて1列揃った?}
  G -- はい --> H[writeBatchで<br/>players更新 + events作成]
  H --> I[本人に紙吹雪<br/>全員に通知]
  G -- いいえ --> J[playersを1回更新<br/>marked + bingoCount]
  J --> K{2列以上になった?}
  K -- はい --> L[ダブル/トリプルを<br/>本人とホストに表示]
  K -- いいえ --> M[穴の表示だけ更新]
```

## 5. 状態遷移

```mermaid
stateDiagram-v2
  [*] --> open: ルーム作成
  open --> ranking: ホストが「抽選終了」
  ranking --> closed: 発表終了後にホストが終了
  open --> closed: 中止時にホストが直接終了
  closed --> [*]

  note right of open
    入室・抽選・穴開けが可能
    時間切れでも自動遷移しない
  end note

  note right of ranking
    入室・穴開けは不可
    集計・じゃんけん・発表
  end note

  note right of closed
    読み取りのみ
    openには戻れない
  end note
```
