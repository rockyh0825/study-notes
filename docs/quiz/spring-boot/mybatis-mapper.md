# クイズ: MyBatis Mapper の書き方

Mapper インターフェース・XML/アノテーションの使い分け・resultMap・動的 SQL・JPA との違いを確認するクイズです。

---

### Q1. `@Mapper` を付けたインターフェースの役割は何ですか。SQL の実装はどこに書きますか。

??? success "解答"
    - `@Mapper` インターフェースは SQL 実行の「入口」となり、メソッド呼び出しと SQL を対応づける
    - 実装は開発者が書かず、MyBatis が実行時にプロキシ（実装）を生成する
    - SQL 本体はアノテーション（`@Select` 等）か、対応する XML マッパーに書く
    - 参考: https://mybatis.org/mybatis-3/java-api.html

### Q2. Mapper メソッドの引数バインドで `@Param` が必要になるのはどんなときですか。

??? success "解答"
    - 引数が複数あるとき、SQL 内で `#{name}` のように名前で参照するには `@Param("name")` を付ける
    - 引数が1つ（かつ単純型/オブジェクト）の場合は `@Param` なしでも参照できることが多い
    - `@Param` を付けないと `#{param1}` `#{arg0}` のような暗黙名でしか参照できず可読性が落ちる
    - 参考: https://mybatis.org/mybatis-3/sqlmap-xml.html

### Q3. アノテーション方式（`@Select` 等）と XML マッパー方式は、どう使い分けるのが定石ですか。

??? success "解答"
    - 短く静的な SQL はアノテーション方式が手軽
    - 複雑な SQL・動的 SQL・長い SQL は XML 方式が読みやすく保守しやすい
    - XML の `namespace` は対応する Mapper インターフェースの完全修飾名と一致させ、`id` はメソッド名に対応させる
    - プロジェクトで方式を混在させると追いにくくなるため、方針を統一すると良い
    - 参考: https://mybatis.org/mybatis-3/sqlmap-xml.html

### Q4. `resultMap` は何のために使いますか。`mapUnderscoreToCamelCase` 設定との関係は。

??? success "解答"
    - `resultMap` は DB のカラム名と Java/Kotlin のプロパティ名の対応を明示的に定義する
    - 単純なスネークケース↔キャメルケース変換だけなら `mapUnderscoreToCamelCase=true` 設定で済み、`resultMap` を書かなくてよい場合が多い
    - ネストした関連（`association`=1対1、`collection`=1対多）など複雑なマッピングでは `resultMap` が必要
    - 参考: https://mybatis.org/mybatis-3/sqlmap-xml.html#Result_Maps

### Q5. `association` と `collection` の違いは何ですか。1対多を JOIN 1クエリで取得する際の注意点は。

??? success "解答"
    - `association` は 1対1（または多対1）の関連、`collection` は 1対多の関連をマッピングする
    - JOIN で1対多を取ると親行が子の数だけ重複するため、`resultMap` に `id` 要素を定義して親を正しく集約させる必要がある
    - `id` が無いと MyBatis が行を正しくグルーピングできず、親が重複したり子が欠落することがある
    - 参考: https://mybatis.org/mybatis-3/sqlmap-xml.html#Result_Maps

### Q6. 動的 SQL の `<if>` / `<where>` / `<foreach>` はそれぞれ何のためのタグですか。`<where>` を使う利点は。

??? success "解答"
    - `<if>` は条件に応じて SQL 片を含めるか決める
    - `<where>` は中身があるときだけ `WHERE` を付け、先頭の余計な `AND` / `OR` を自動除去する
    - `<foreach>` はコレクションを反復し、IN 句などを組み立てる（`IN (#{item}, ...)`）
    - `<where>` を使うと「条件が0個のときに `WHERE` が残る」「先頭が `AND` になる」といった文字列連結の不具合を避けられる
    - 参考: https://mybatis.org/mybatis-3/dynamic-sql.html

### Q7. アノテーション方式で動的 SQL を書きたい場合はどうしますか。

??? success "解答"
    - `@SelectProvider` / `@InsertProvider` / `@UpdateProvider` / `@DeleteProvider` を使い、SQL を組み立てるプロバイダクラスのメソッドを指定する
    - プロバイダ内で `SQL {}` ビルダー DSL を使い、条件に応じて SQL を構築する
    - ただし複雑になるなら XML の動的 SQL の方が読みやすいことが多い
    - 参考: https://mybatis.org/mybatis-3/java-api.html

### Q8. MyBatis と JPA（Hibernate）の根本的な違いは何ですか。MyBatis を選ぶ動機は。

??? success "解答"
    - JPA は ORM でオブジェクトとテーブルを対応づけ、SQL を自動生成・抽象化する
    - MyBatis は SQL マッパーで、SQL を開発者が明示的に書き、結果をオブジェクトにマッピングする
    - 「発行される SQL を完全に制御・把握したい」「複雑なクエリを素の SQL で書きたい」場合は MyBatis が向く
    - 反面、CRUD のボイラープレートは JPA の方が少ない、というトレードオフがある
    - 参考: https://mybatis.org/mybatis-3/
