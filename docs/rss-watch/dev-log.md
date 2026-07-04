# RSS Watch 開発ログ

RSS Watch の開発を時系列で記録する。新しい記録は上に追記していく。

---

## 2026-07-04(方針決定)

### やったこと

- **プロジェクトの方針を壁打ちで決定**(詳細は [アーキテクチャ決定ログ](architecture-decisions.md))。
    - **フル Kotlin + Spring Boot** で実装し、自宅サーバーで運用する
    - 定期実行 + DB 直書きでも成立する規模だが、学習題材として **Kafka を挟んだイベント駆動構成**にする
    - topic 1 本(`rss.items`)を **sink(マイクロバッチ DB 書き込み)と live(SSE 配信)の 2 consumer group** で読む構成にする
- **収集対象フィードを選定し、取得できることを確認**。
    - 技術系 5 本: はてブ IT / Zenn / Qiita / Publickey / Hacker News
    - 求人系 4 本: HN Jobs / Who is hiring / We Work Remotely / Remote OK
    - 9 本すべて取得可能で、初回で 277 件集まる規模感を確認
- **キーワード抽出の方式を検証**。辞書ベース(正規化名 + エイリアス、約 60 分類)で、求人からの技術言及ランキングが意味のある結果になることを確認(初回データでは Python 6 件がトップ、次いで PostgreSQL / TypeScript / 機械学習)
- **開発環境の準備**: JDK 21 / Gradle / Docker(colima)の動作確認

### 設計判断

- Kafka はこの規模には明確にオーバーキル。それでも「実データが流れ続ける」「guid をキーにした冪等書き込みと相性が良い」という点でストリーム処理の学習題材として理想的と判断して採用
- Spring Boot vs Ktor は、日本の求人での需要と spring-kafka の学びを優先して Spring Boot に
- 蓄積・集計は DB の仕事、Kafka は輸送路、ブラウザへは SSE 中継、という役割分担を明確化

### ハマったこと / 気づき

- **日本語の求人 RSS はほぼ存在しない**。Wantedly や Findy は RSS を公開しておらず、求人系フィードは当面英語圏中心になる。日本の求人サイトで RSS を見つけたら設定に足す
- 正規表現の `\b` は日本語文中で効かない(日本語文字も `\w` のため)。`Pythonで` から `Python` を拾うには英数字 lookaround の独自境界が必要
- `Go` のような短い言語名は誤検出が多く、大文字小文字を区別する別枠での照合にした

### 次にやること

- Kotlin + Spring Boot プロジェクトの雛形作成(`~/dev/rss/kotlin/`)
- Docker Compose で Kafka(KRaft シングルブローカー)+ kafka-ui を立てる
- fetcher(Rome + `@Scheduled` + producer)→ sink / live consumer → SSE 画面の順に実装
- sink 停止 → catch-up の再送・冪等性実験をやってみる
