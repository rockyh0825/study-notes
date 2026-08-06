# 値オブジェクト（Value Object）

## 概要

DDDにおける値オブジェクトは、属性の値そのものによって同一性が決まるオブジェクト。IDを持たず、内容が同じであれば別のインスタンスでも「同じもの」として扱う。

## 特徴

- **不変（イミュータブル）**: 生成後に状態を変更しない。値を変えたい場合は新しいインスタンスを作る
- **同一性は値で判定**: 全属性が等しければ同じ値オブジェクトとみなす（IDでは判定しない）
- **副作用がない**: 不変なので複数箇所で安全に共有できる

## 使う理由・メリット

- **プリミティブ型執着（Primitive Obsession）を防げる**: 「金額」をただのintで扱うのではなく `Money` という値オブジェクトにすることで、通貨単位の取り違えなどのバグを型レベルで防げる
- **ドメインのルールをオブジェクト内に閉じ込められる**: `Email` 値オブジェクトのコンストラクタでフォーマット検証を行えば、不正なメールアドレスがドメイン内に存在しない状態を保証できる
- **不変なのでスレッドセーフで、テストしやすい**

## 実装例（Kotlin）

```kotlin
data class Money(val amount: Int, val currency: String) {
    init {
        require(amount >= 0) { "amount must not be negative" }
    }
    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "currency mismatch" }
        return Money(amount + other.amount, currency)
    }
}
```

`data class` を使うことで equals/hashCode/toString が値ベースで自動生成され、値オブジェクトの実装に適している。

## エンティティとの使い分け

- 「同じ人物か」「同じ注文か」のようにライフサイクルを通じて追跡したい対象 → エンティティ（IDで同一性判定）
- 「いくらか」「どの期間か」「どこの住所か」のように、値そのものに意味があり、置き換え可能な対象 → 値オブジェクト
