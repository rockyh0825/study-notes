# リポジトリ（Repository）パターン

## リポジトリとは

リポジトリは、ドメインオブジェクトの永続化・再構築をドメイン層から見て抽象化する仕組み。DBなどのインフラ技術の詳細をドメイン層から隠蔽し、「集約をどこかに保存する／集約をどこかから取り出す」という操作をインターフェースとして表現する。

## どういう時に切るか

基本的には**集約（Aggregate）単位**でリポジトリを1つ用意する。集約ルートに対して1つのリポジトリを対応させ、集約の内部にある子エンティティや値オブジェクトに対しては個別のリポジトリを作らない。

## 利点

- **依存性逆転**: ドメイン層はリポジトリのインターフェースにのみ依存し、DB等の具体的な永続化技術を知らなくて済む
- **差し替えやすさ**: 実装（RDB、NoSQL、外部APIなど）を後から変更してもドメイン層のコードに影響しない
- **テスト容易性**: インターフェースなのでフェイク実装に差し替えてユニットテストが書きやすい

## テストの作り方

- リポジトリは`interface`として定義し、ドメイン層・アプリケーション層はそのインターフェースにのみ依存させる
- **ユニットテスト**: インメモリのフェイク実装（Mapなどで保持するだけの実装）をリポジトリとして注入し、DBなしで高速にテストする
- **結合テスト**: 実際のDB実装が正しく動くかは、Testcontainers等で実DB（MySQL等）を起動して検証する

## Kotlinでの実装例

```kotlin
// ドメイン層: インターフェースのみ定義
interface UserRepository {
    fun findById(id: UserId): User?
    fun save(user: User)
}

// インフラ層: JPAを使った実装
class JpaUserRepository(
    private val jpaRepository: UserJpaRepository
) : UserRepository {
    override fun findById(id: UserId): User? =
        jpaRepository.findById(id.value).orElse(null)?.toDomain()

    override fun save(user: User) {
        jpaRepository.save(UserEntity.fromDomain(user))
    }
}

// テスト用: インメモリのフェイク実装
class InMemoryUserRepository : UserRepository {
    private val store = mutableMapOf<UserId, User>()

    override fun findById(id: UserId): User? = store[id]

    override fun save(user: User) {
        store[user.id] = user
    }
}
```

## まとめ

リポジトリは集約単位で切り、インターフェースをドメイン層に置くことでインフラ技術への依存を排除する。テストではインメモリのフェイク実装で高速なユニットテストを行い、実装の正しさは結合テストで別途担保する。
