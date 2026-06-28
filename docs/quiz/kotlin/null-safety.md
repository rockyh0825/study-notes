# クイズ: Kotlin の null 安全

nullable 型と `?.` / `?:` / `!!` / `let` / プラットフォーム型を確認するクイズです。

---

### Q1. `String` と `String?` の違いは何ですか。Kotlin はどのタイミングで null 由来のエラーを防いでいますか。

??? success "解答"
    - `String` は非 null 型で null を代入できない。`String?` は nullable 型で null を許容する
    - Kotlin は「コンパイル時」に非 null 型への null 代入や、nullable 型への安全でないアクセスを弾く
    - これにより実行時の NullPointerException を大幅に減らせる（null の有無が型に表れる）
    - 参考: https://kotlinlang.org/docs/null-safety.html

### Q2. 安全呼び出し `?.` の挙動を説明してください。`a?.b?.c` のチェーンで途中が null だった場合はどうなりますか。

??? success "解答"
    - `?.` はレシーバが null なら呼び出しを行わず、式全体として null を返す
    - チェーンでは途中のいずれかが null になった時点で評価が止まり、式全体が null になる
    - 例: `user?.address?.city` は `user` か `address` が null なら全体が null
    - 参考: https://kotlinlang.org/docs/null-safety.html#safe-calls

### Q3. エルビス演算子 `?:` は何をしますか。`?.` と組み合わせた典型例を挙げてください。

??? success "解答"
    - `?:` は左辺が null のときに右辺（既定値）を返す
    - `val name = user?.name ?: "guest"` のように `?.` と併用して「null なら既定値」を表現する
    - 右辺に `return` や `throw` を置けるため、`?: return` / `?: throw IllegalStateException()` で早期リターン・例外スローが書ける
    - 参考: https://kotlinlang.org/docs/null-safety.html#elvis-operator

### Q4. `!!` は何をする演算子ですか。null だった場合どうなりますか。なぜ乱用が危険なのですか。

??? success "解答"
    - `!!` は「null でない」と断言し、nullable 型を非 null 型へ変換する
    - 実際に null だった場合は NullPointerException を投げる
    - 乱用するとせっかくの null 安全（コンパイル時チェック）を握りつぶし、実行時 NPE を生むため危険。本当に null でないと保証できる限定的な場面でのみ使う
    - 参考: https://kotlinlang.org/docs/null-safety.html#the-operator

### Q5. `?.let { }` は何のためのパターンですか。ブロック内で値はどう参照しますか。

??? success "解答"
    - 「null でないときだけ処理を実行する」ためのパターン
    - レシーバが null なら `let` のブロックは実行されない
    - ブロック内では非 null になった値を `it`（または引数名）として参照できる
    - 例: `user?.let { sendMail(it.email) }`
    - 参考: https://kotlinlang.org/docs/scope-functions.html#let

### Q6. プラットフォーム型（platform type, `String!` 表記）とは何ですか。なぜ注意が必要ですか。

??? success "解答"
    - Java など null 情報を持たないコードから来た型で、Kotlin が null 許容かどうかを判断できない型（`String!` と表記される）
    - Kotlin は非 null としても nullable としても扱うことを許すため、実際に null が来ると実行時 NPE になりうる
    - Java 側の `@Nullable` / `@NotNull` アノテーションがあれば Kotlin はそれを尊重する。境界では明示的に nullable として受けるのが安全
    - 参考: https://kotlinlang.org/docs/null-safety.html#nullability-and-java-interop

### Q7. `?.let { }` を使った null チェックと、`if (x != null) { ... }` によるスマートキャストの違い・使い分けは。

??? success "解答"
    - `if (x != null)` はローカルの `val` ならブロック内でスマートキャストが効き、非 null として扱える
    - `?.let` はレシーバを `it` として受け、メソッドチェーンの一部や式として書きやすい
    - `var` やカスタム getter などスマートキャストが効かないケースでは `?.let` が安全な選択になることがある
    - どちらも可読性で選んでよいが、ネストが深くなる場合は `?.let` や早期リターン（`?: return`）の方が読みやすい
    - 参考: https://kotlinlang.org/docs/null-safety.html
