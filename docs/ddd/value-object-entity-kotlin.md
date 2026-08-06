# 値オブジェクトとエンティティ（Kotlinでの表現）

DDD（ドメイン駆動設計）における値オブジェクト（Value Object）とエンティティ（Entity）の違いと、Kotlinコードベースでの表現方法を整理する。

## ドメインモデルとドメインオブジェクトの違い

- **ドメインモデル**: 業務ドメインの概念・ルール・関係性を表す設計上の抽象。実装そのものではなく「何を表現すべきか」という概念モデル。
- **ドメインオブジェクト**: ドメインモデルをコードとして実装した個々の要素。値オブジェクトやエンティティ、ドメインサービスなどが具体的なドメインオブジェクトにあたる。

つまりドメインモデルが設計図で、ドメインオブジェクトはその設計図をもとに作られた実際のクラス群、という関係になる。

## 値オブジェクト（Value Object）とは

- 属性の**値そのもの**によって同一性が決まるオブジェクト。IDを持たない。
- **不変（Immutable）**であるべき。生成後に状態を変更しない。
- 属性がすべて等しければ「同じもの」とみなす（構造的等価性）。
- 例: `Money`, `EmailAddress`, `DateRange` など。

## エンティティ（Entity）とは

- **識別子（ID）**によって同一性が決まるオブジェクト。属性の値が変化しても、IDが同じであれば「同じもの」とみなす。
- ライフサイクルを持ち、状態が時間とともに変化しうる。
- 例: `User`, `Order`, `Article` など。

## Kotlinでの表現方法

### 値オブジェクト

`data class` を使うと `equals`/`hashCode`/`copy` が属性値ベースで自動生成されるため、値オブジェクトの実装に適している。すべてのプロパティを `val` にして不変にする。

```kotlin
data class Money(val amount: Long, val currency: String) {
    init {
        require(amount >= 0) { "amount must not be negative" }
    }
}
```

### エンティティ

エンティティは属性が変化しうるため、`data class` をそのまま使うと属性値ベースの `equals` になってしまい、同一性の定義がずれる。ID(識別子)のみに基づいて `equals`/`hashCode` を実装するのが基本。

```kotlin
class User(
    val id: UserId,
    var name: String,
) {
    override fun equals(other: Any?): Boolean =
        other is User && other.id == id

    override fun hashCode(): Int = id.hashCode()
}
```

IDである `UserId` 自体は値オブジェクトとして `data class` で表現することが多い。

```kotlin
data class UserId(val value: String)
```

## まとめ

| | 値オブジェクト | エンティティ |
|---|---|---|
| 同一性の基準 | 属性値 | ID |
| 可変性 | 不変 | 可変（ライフサイクルを持つ） |
| Kotlinでの表現 | `data class` + `val` | ID基準の `equals`/`hashCode` を持つクラス |
