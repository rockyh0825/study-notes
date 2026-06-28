# Kotlin の data class

Kotlin のデータクラス（`data class`）の特徴と、自動生成されるメソッド・便利機能をまとめる。

---

## data class とは

`data class` は「データを保持すること」が主目的のクラス。Java で言えば、フィールド・ゲッター・`equals`/`hashCode`/`toString` を持つ POJO（いわゆる値オブジェクトや DTO）を 1 行で書けるようにしたもの。

```kotlin
data class User(val id: Int, val name: String)
```

これだけで、後述する `equals()` / `hashCode()` / `toString()` / `copy()` / `componentN()` がコンパイラによって自動生成される。通常の `class` で同じことをやると、これらを全部手書きすることになる。

### 通常の class との違い

```kotlin
// 通常のクラス: equals は「同じインスタンスか」で比較される（参照の同一性）
class PlainUser(val id: Int, val name: String)

val a = PlainUser(1, "Taro")
val b = PlainUser(1, "Taro")
println(a == b)  // false（中身が同じでも別インスタンス）
```

```kotlin
// data class: equals は「プロパティの値が同じか」で比較される（値の同一性）
data class User(val id: Int, val name: String)

val a = User(1, "Taro")
val b = User(1, "Taro")
println(a == b)  // true（値が同じなら等しい）
```

このように、`data class` は「値が同じなら等しい」という直感に沿った比較ができる。DTO・ドメインモデル・値オブジェクトのように「中身こそが本体」のデータに向いている。

### 宣言の制約

- **プライマリコンストラクタに最低 1 つのプロパティが必要**。
- プライマリコンストラクタのパラメータは `val` か `var` を付けてプロパティにする必要がある。
- `abstract` / `open` / `sealed` / `inner` にはできない。

```kotlin
data class Empty()             // NG: プロパティが 1 つもない
data class Bad(name: String)   // NG: val/var が付いていない（ただのパラメータ）
data class Good(val name: String)  // OK
```

掃除アプリのドメインモデルもこの形で定義している。全フィールドを `val` にして不変にし、変更は後述の `copy()` で行う。

```kotlin
data class Room(
    val id: UUID,
    val userId: UUID,
    val name: String,
    val type: RoomType,
    val gridX: Int,
    val gridY: Int,
    val gridW: Int,
    val gridH: Int,
    val createdAt: Instant,
    val updatedAt: Instant,
)
```

---

## 自動生成されるメソッド

### `equals()` / `hashCode()`

プライマリコンストラクタのプロパティの**値**を使って比較・ハッシュ計算する。`equals()` が等しい 2 つのオブジェクトは `hashCode()` も必ず一致するので、`HashSet` や `HashMap` のキーとしても正しく動く。

```kotlin
val set = setOf(User(1, "Taro"), User(1, "Taro"))
println(set.size)  // 1（値が同じものは重複扱い）
```

### `toString()`

プロパティを並べた読みやすい文字列を返す。ログやデバッグで中身がそのまま見える。

```kotlin
println(User(1, "Taro"))  // User(id=1, name=Taro)
```

### `componentN()`

`component1()`, `component2()`, ... というメソッドが、プロパティの宣言順に生成される。これが後述の分解宣言を支える。

### 対象になるプロパティに注意

自動生成の対象は**プライマリコンストラクタで宣言したプロパティだけ**。クラス本体（`{ }` の中）で宣言したプロパティは `equals` / `toString` などに含まれない。

```kotlin
data class User(val id: Int) {
    var nickname: String = ""  // これは equals/toString に含まれない
}

val a = User(1).apply { nickname = "T" }
val b = User(1).apply { nickname = "X" }
println(a == b)  // true（nickname は比較対象外）
```

---

## copy

`copy()` は「一部のプロパティだけ変更した新しいインスタンス」を作るメソッド。元のインスタンスは変更しない（イミュータブルなまま値を更新する）。

```kotlin
val user = User(1, "Taro")
val renamed = user.copy(name = "Jiro")

println(user)     // User(id=1, name=Taro)   ← 元は変わらない
println(renamed)  // User(id=1, name=Jiro)
```

名前付き引数で「変えたいプロパティだけ」を指定し、残りは元の値が引き継がれる。

```kotlin
// updatedAt だけ現在時刻に更新した新しい Room を作る
val updated = room.copy(updatedAt = Instant.now())
```

`val` 中心の不変設計では、状態を「書き換える」のではなく「新しい値に差し替える」。`copy()` はその差し替えを安全・簡潔に書くための道具になる。

---

## 分解宣言（Destructuring）

`data class` のインスタンスは、複数の変数へ一度に分解して代入できる。これが**分解宣言**。

```kotlin
val user = User(1, "Taro")
val (id, name) = user
// id   == user.component1()  == 1
// name == user.component2()  == "Taro"
```

`val (id, name)` の `id` は `component1()`、`name` は `component2()` に対応する。**変数名ではなく宣言順**で割り当たる点に注意（プロパティ名と一致している必要はない）。

### 使いどころ

```kotlin
// Map のループ（entry が key/value に分解される）
for ((key, value) in map) {
    println("$key = $value")
}

// 戻り値で複数の値を返す
fun minMax(list: List<Int>): Pair<Int, Int> = list.min() to list.max()
val (min, max) = minMax(numbers)
```

### 不要な値は `_` で読み飛ばせる

使わない要素は `_`（アンダースコア）でスキップできる。

```kotlin
val (_, name) = user   // id は使わないので読み飛ばす
```
