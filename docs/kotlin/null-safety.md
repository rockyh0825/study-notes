# Kotlin の null 安全

Kotlin の null 安全（nullable 型と関連演算子 `?.` / `?:` / `!!` / `let`）をまとめる。

---

## nullable 型と非 null 型

Kotlin では型レベルで「null が入りうるか」を区別する。型名の末尾に `?` が付くかどうかが分かれ目。

```kotlin
val name: String = "Taro"   // 非 null 型: null を代入できない
val nick: String? = null    // nullable 型: null を代入できる
```

```kotlin
val name: String = null   // コンパイルエラー（null を許さない型に null を入れている）
```

非 null 型の変数には、そもそも null を代入できない。代入しようとした時点でコンパイルが通らない。

### なぜ NullPointerException を減らせるのか

Java では「どの変数が null になりうるか」が型からは分からず、実行してみて初めて NPE で落ちる。Kotlin は **null になりうる箇所を `?` でコンパイル時に明示**させ、nullable な値をそのまま使おうとするとコンパイルエラーにする。

```kotlin
val nick: String? = getNickname()
println(nick.length)   // コンパイルエラー: null かもしれないので直接呼べない
```

「null チェックを忘れる」というミス自体をコンパイラが防いでくれるので、実行時の NPE が激減する。チェックを通すための演算子が次の `?.` / `?:` / `!!` / `let`。

---

## 安全呼び出し `?.`

`?.` は「レシーバが null でなければメソッド/プロパティを呼び、null なら何もせず null を返す」演算子。

```kotlin
val nick: String? = getNickname()
val len: Int? = nick?.length
// nick が null なら len も null、null でなければ length が入る
```

`nick?.length` の結果は `Int?`（nullable）になる点に注意。null の可能性が残るので、戻り値も nullable 型になる。

### チェーンした場合

途中のどこかが null なら、それ以降は評価されず全体が null になる。

```kotlin
val city: String? = user?.address?.city
// user が null → 全体 null
// address が null → 全体 null
// 全部非 null → city が入る
```

ネストした null チェックを `if` で何段も書かずに済むのが利点。

---

## エルビス演算子 `?:`

`?:` は「左辺が null のときに右辺（既定値）を使う」演算子。`?.` で出てきた null を、ここで具体的な値に変換する。

```kotlin
val nick: String? = getNickname()
val display: String = nick ?: "名無し"
// nick が null なら "名無し"、そうでなければ nick

// ?. との併用: null なら 0 を使う
val len: Int = nick?.length ?: 0
```

### 早期リターン・例外スローとの組み合わせ

右辺には `return` や `throw` も書ける。「null だったら処理を打ち切る」というガード節を簡潔に書ける。

```kotlin
fun greet(user: User?) {
    val u = user ?: return                       // null なら何もせず抜ける
    val name = u.name ?: throw IllegalArgumentException("name is required")
    println("Hello, $name")
}
```

---

## 非 null アサーション `!!`

`!!` は「これは絶対に null ではない」とプログラマが断言し、nullable 型を非 null 型に変換する演算子。

```kotlin
val nick: String? = getNickname()
val len: Int = nick!!.length   // 非 null として扱う
```

ただし、もし実際に null だった場合は **その場で NPE を投げる**。つまり Kotlin がせっかく防いでくれている NPE を、自分の手で呼び戻す操作になる。

### 乱用が危険な理由

`!!` は「コンパイラのチェックを黙らせる」だけで、null の可能性そのものは消えない。安易に付けると Java と同じ NPE 地獄に戻ってしまう。基本は `?.` / `?:` / `let` で安全に処理し、`!!` は「直前で null チェック済みだとロジック上は保証できるが型では表現しきれない」ごく限定的な場面だけに留めるのがよい。

---

## `let` と null 処理

`let` はスコープ関数の 1 つで、`?.let { }` の形で「null でないときだけブロックを実行する」パターンによく使う。

```kotlin
val nick: String? = getNickname()

nick?.let {
    // nick が null でないときだけここに入る
    // ブロック内では it が非 null の String として使える
    println("ニックネームは ${it.length} 文字")
}
```

`?.let { }` は「null なら何もしない、非 null ならブロック内で値を使う」を 1 つの式で書ける。ブロック内の `it` はすでに非 null 型なので、中で改めて `?.` を付ける必要がない。

### 他のスコープ関数との簡単な比較

| 関数 | ブロック内の参照 | 戻り値 | よく使う用途 |
|---|---|---|---|
| `let` | `it` | ブロックの最後の式 | null チェック・値の変換 |
| `run` | `this` | ブロックの最後の式 | レシーバを使った計算 |
| `also` | `it` | レシーバ自身 | ログ出力などの副作用 |
| `apply` | `this` | レシーバ自身 | オブジェクトの初期化 |

null 処理では「`it` で受けられて、変換結果を返せる」`let` が最も使いやすい。

---

## プラットフォーム型

Java のコードを Kotlin から呼ぶと、Java 側の戻り値は null になりうるか分からない。Kotlin はこれを**プラットフォーム型**（`String!` のように `!` 付きで表示される）として扱い、null 安全のチェックを一旦保留する。

```kotlin
// 例: Java のメソッド String getName() を呼ぶと、戻り値は String!（プラットフォーム型）
val name = javaObject.name   // 型は String!（nullable とも非 null とも決めない）
```

プラットフォーム型は「非 null として扱ってもよいし、nullable として扱ってもよい」状態。非 null として使った後に実は null だと、実行時 NPE になる。Java と境界を接する箇所では、**明示的に `String?` として受けて null チェックする**のが安全。Kotlin だけで完結するコードと違い、ここはコンパイラが守ってくれないので注意がいる。
