# セキュリティ運用ガイド

この文書は、Google Cloud ビンゴの開発・公開・運用で、秘密情報や個人情報をリポジトリへ混入させないための手順をまとめたものです。正式な受け入れ基準は `.kiro/specs/gcp-bingo/requirements.md`、実装方式は `.kiro/specs/gcp-bingo/design.md` を正とします。

## 1. 基本方針

- 公開リポジトリには、長期間有効な認証情報を置かない。
- `.gitignore`、ローカルフック、CI、GitHub Push Protectionを重ねて使う。
- ローカルフックは補助とし、GitHub Actionsのsecret scanを必須の最終チェックとする。
- 将来CIデプロイを追加する場合も、サービスアカウント鍵は使わず、GitHub ActionsのOIDCとGoogle Cloud Workload Identity Federationを使う。
- 秘密情報がリモートへ出た場合は、履歴の修正より先に、その認証情報を無効化・ローテーションする。

## 2. コミットしてはいけないもの

| 種類 | 例 |
|---|---|
| サービスアカウント鍵 | `*-service-account*.json`、JSON内の `private_key` / `client_email` |
| Application Default Credentials | `application_default_credentials.json` |
| 秘密鍵 | `*.pem`、`*.key`、`-----BEGIN PRIVATE KEY-----` |
| CLI・CIトークン | `FIREBASE_TOKEN`、アクセストークン、リフレッシュトークン |
| 実環境のローカル設定 | `firebase-config.js`、`.firebaserc`、`.env`、`.env.*` |
| 個人の認証ファイル | Google Cloud CLI、Firebase CLI、ブラウザ、OSの認証キャッシュ |

次のファイルはコミットできます。

- `firebase-config.example.js`: プレースホルダーだけを含む
- `.firebaserc.example`: プレースホルダーだけを含む
- `.env.example`: 必要になった場合のみ。実値を含めない

## 3. Firebase Web APIキーの扱い

FirebaseのWeb設定はブラウザで使用するため、デプロイ後は利用者から確認できます。Web APIキーをパスワードのように隠すことだけでは保護になりません。

このプロジェクトでは次の方針にします。

- 実値をリポジトリへコミットしない。
- Firestore Security Rulesをアクセス制御の正本にする。
- Google Cloud側で、APIキーの利用可能APIとHTTPリファラーを必要最小限に制限する。
- サービスアカウント鍵をブラウザコードへ置かない。
- `firebase-config.example.js` には、`YOUR_PROJECT_ID` などのプレースホルダーだけを書く。

## 4. 検査の多層化

### 4.1 `.gitignore`

誤って追跡対象にすることを防ぎます。ただし、すでに追跡されたファイルや、名前を変えた秘密情報は防げません。

### 4.2 pre-commit

ステージされた変更をGitleaksと禁止ファイル検査へ通します。コミットを作る前に止めるため、ローカルで最も早く気づけます。

### 4.3 pre-push

リモートとの差分に含まれるコミットを再検査します。pre-commitの設定漏れや、フック導入前のコミットを補います。

pre-pushは `--no-verify` で回避でき、clone時に自動設定されないため、単独では防衛線になりません。

### 4.4 GitHub Actions

次のイベントでsecret scanを実行します。

- Pull Request
- featureブランチへのpush
- `main` へのpush

次のいずれかを検出した場合は、ジョブを失敗させます。

- Gitleaksの検出
- コミット禁止ファイル
- GCPサービスアカウントJSONの特徴
- 秘密鍵ヘッダー
- CLI・CIトークンの特徴

CIはローカルフックを設定していない端末や、`--no-verify` で作られたコミットも検査します。

### 4.5 GitHub Secret Scanning / Push Protection

GitHubリポジトリのSecurity設定でSecret ScanningとPush Protectionを有効にします。GitHubが認識するトークンをpush時点で拒否する補助防衛線です。

## 5. ローカルセットアップ

実装後は、リポジトリをcloneした各端末で次を実行します。

```bash
./scripts/setup-git-hooks.sh
```

このスクリプトはGitのフック先を、リポジトリで管理する `.githooks/` に設定します。

```bash
git config core.hooksPath .githooks
```

手動検査は共通スクリプトから実行します。

```bash
./scripts/secret-scan.sh
```

Gitleaksのバージョンは固定し、ローカルとCIで同じバージョンを使います。具体的なバージョンと引数は実装時に公式ドキュメントを確認して決めます。

## 6. 誤検知の扱い

- 検査を無効化して通さない。
- まず、プレースホルダーやテストデータへ置き換えられないか確認する。
- 許可が必要な場合は、値そのものではなく、最小範囲のルールまたはファイルパスを `.gitleaks.toml` で許可する。
- 許可には、なぜ秘密情報ではないかをコメントとして残す。
- 広範囲な正規表現やリポジトリ全体の除外は作らない。

## 7. 将来のCIデプロイ

今回はGitHub Actionsによる自動デプロイを作りません。将来追加する場合は次の構成にします。

```text
GitHub Actions
  → GitHub OIDCトークン
  → Google Cloud Workload Identity Federation
  → 短時間だけ有効なGoogle Cloud認証
  → Firebase Hosting / Firestore Rulesをデプロイ
```

- サービスアカウントJSONをGitHub Secretsへ登録しない。
- デプロイ専用サービスアカウントには必要最小限の権限だけを付与する。
- GitHubのEnvironment保護と、デプロイ対象ブランチの制限を使う。

## 8. 秘密情報をpushした場合

1. **直ちに認証情報を無効化・ローテーションする。**
2. GitHub Actions、Issue、PRコメント、ログなど、別の露出先も確認する。
3. 新しい認証情報が必要なら、正しい保管先へ登録する。リポジトリには置かない。
4. 必要に応じてGit履歴から削除する。
5. secret scanを再実行する。
6. 原因と再発防止策を記録する。

履歴から削除しても、push済みの値は漏えいした前提で扱います。

## 9. 参加者データ

参加者名、UID、カード、ビンゴ時刻はFirestoreに保存されます。

- 入室画面で、本名ではなくニックネームの利用を案内する。
- ルームIDを知る人だけにURLを共有する。
- イベント後の保存期間と削除手順は、実装・運用タスクで確定する。
- 調査ログへ名前、UID、カード内容を必要以上に出さない。

## 10. 実装時の確認項目

- [ ] `.gitignore` に禁止ファイルを追加した
- [ ] `firebase-config.example.js` と `.firebaserc.example` はプレースホルダーだけである
- [ ] Gitleaksのバージョンを固定した
- [ ] `scripts/secret-scan.sh` を作成した
- [ ] pre-commitとpre-pushが共通スクリプトを呼ぶ
- [ ] GitHub ActionsがPRとpushで動く
- [ ] 意図的なダミー秘密情報を検出するCIテストがある
- [ ] GitHub Secret Scanning / Push Protectionを確認した
- [ ] サービスアカウント鍵がリポジトリ、GitHub Secrets、ログにない
- [ ] 漏えい時のローテーション手順をREADMEから参照できる
