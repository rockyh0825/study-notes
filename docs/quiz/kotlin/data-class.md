# クイズ: Kotlin の data class

`data class` の自動生成メソッド・`copy`・分解宣言・通常クラスとの違いを確認するクイズです。

---

### Q1. `data class` を宣言すると、コンパイラが自動生成するメソッドを4つ挙げてください。

??? success "解答"
    - `equals()` / `hashCode()` / `toString()` / `componentN()` の4つ（加えて `copy()` も生成される）
    - `equals()`・`hashCode()` はプロパティ「値」ベースで比較・ハッシュ化される
    - `toString()` は `User(id=1, name=foo)` のようにプロパティを並べた読みやすい形式になる
    - `componentN()` は分解宣言（destructuring）用に1プロパティにつき1つ生成される
    - 参考: https://kotlinlang.org/docs/data-classes.html

### Q2. 自動生成される `equals()` / `hashCode()` / `toString()` の対象になるのはどのプロパティですか。クラス本体（`{}` 内）で宣言したプロパティは含まれますか。

??? success "解答"
    - 対象は「プライマリコンストラクタで宣言したプロパティ」のみ
    - クラス本体（ボディ）で宣言したプロパティは `equals()` 等の対象に含まれない
    - つまり同じコンストラクタ引数を持つインスタンスは、ボディのプロパティが違っても `equals()` では等しいと判定される
    - 参考: https://kotlinlang.org/docs/data-classes.html#properties-declared-in-the-class-body

### Q3. 通常の `class` と `data class` の最大の違いは何ですか。なぜ DTO や値オブジェクトに `data class` が向いているのですか。

??? success "解答"
    - 通常クラスの `equals()` は参照（同一インスタンスか）で比較するが、`data class` は値で比較する
    - DTO・値オブジェクトは「同じ値なら等しい」とみなしたい用途であり、値ベースの比較・`toString`・`copy` が標準で手に入るため相性が良い
    - 逆に「同一性（アイデンティティ）」が重要なエンティティには通常クラスが向く場合もある
    - 参考: https://kotlinlang.org/docs/data-classes.html

### Q4. `data class User(val id: Int, val name: String)` に対し `user.copy(name = "bob")` を呼ぶと何が起きますか。元のインスタンスはどうなりますか。

??? success "解答"
    - 指定したプロパティ（`name`）だけを差し替えた「新しいインスタンス」が生成される
    - 元のインスタンスは変更されない（イミュータブルなまま値を更新するパターン）
    - 指定しなかったプロパティ（`id`）は元の値がそのままコピーされる
    - 名前付き引数で「変えたいものだけ」を書けるのが利点
    - 参考: https://kotlinlang.org/docs/data-classes.html#copying

### Q5. 分解宣言 `val (id, name) = user` は内部的に何を呼び出していますか。順序は何で決まりますか。

??? success "解答"
    - `component1()` / `component2()` …（`componentN()`）を順に呼び出している
    - 順序は「プライマリコンストラクタでのプロパティ宣言順」で決まる（変数名ではない）
    - そのため変数名を変えても、対応するのは宣言位置であって名前ではない点に注意
    - 参考: https://kotlinlang.org/docs/destructuring-declarations.html

### Q6. 分解宣言で不要な値を読み飛ばしたいときはどう書きますか。ループや Map の反復での使用例は。

??? success "解答"
    - 不要な値は `_`（アンダースコア）でスキップできる: `val (_, name) = user`
    - `for ((key, value) in map) { ... }` のように Map のエントリ反復で使える（`Map.Entry` に `component1/2` があるため）
    - 戻り値で複数の値を返したいときに `Pair` / `Triple` や data class と組み合わせて受け取れる
    - 参考: https://kotlinlang.org/docs/destructuring-declarations.html

### Q7. `data class` を宣言するうえでの制約を挙げてください（プライマリコンストラクタ・abstract など）。

??? success "解答"
    - プライマリコンストラクタに最低1つのパラメータが必要
    - プライマリコンストラクタのパラメータは `val` または `var` でなければならない
    - `abstract` / `open` / `sealed` / `inner` にはできない
    - 参考: https://kotlinlang.org/docs/data-classes.html
