# 技術方針

- Firebase Hosting + Firestore + 匿名認証のみ。Cloud Functions / Cloud Run は使わない。
- ビルド不要の HTML + ES Modules。Firebase SDK は CDN から読み込む。
- 無料枠(Spark)に収める。
