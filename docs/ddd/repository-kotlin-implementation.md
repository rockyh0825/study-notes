# Kotlinでのリポジトリ実装

## リポジトリのインターフェースはドメイン層に置く

DDDでは、リポジトリの**インターフェース**をドメイン層に定義し、具体的な実装（JPAやSQLなど）はインフラ層に置く。こうすることで、ドメイン層は永続化技術に依存しなくなる。

```kotlin
// domain層
interface UserRepository {
    fun findById(id: UserId): User?
    fun save(user: User)
}
```

## インフラ層での実装

インフラ層では、Spring Data JPAなどのフレームワークを使ってインターフェースを実装する。

```kotlin
// infrastructure層
interface UserJpaRepository : JpaRepository<UserEntity, Long>

class UserRepositoryImpl(
    private val jpaRepository: UserJpaRepository
) : UserRepository {
    override fun findById(id: UserId): User? =
        jpaRepository.findById(id.value).orElse(null)?.toDomain()

    override fun save(user: User) {
        jpaRepository.save(user.toEntity())
    }
}
```

## ドメインモデルとJPAエンティティの分離

`UserEntity`（JPAエンティティ）と`User`（ドメインモデル）は別クラスとして定義し、リポジトリ実装内で相互変換する。これにより、ドメインモデルがJPAアノテーションに汚染されず、ドメイン層の独立性を保てる。

## nullを返す戻り値の意味

`findById`のようなメソッドがnull許容型（`User?`）を返す場合、それは「該当するエンティティが存在しないかもしれない」ことを型で表現している。呼び出し側はnullチェックを強制される。
