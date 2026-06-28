# MyBatis Mapper の書き方

Spring Boot + MyBatis での Mapper の書き方（インターフェース・XML/アノテーション・resultMap・動的SQL）をまとめる。

---

## Mapper インターフェース

MyBatis では、`@Mapper` を付けた**インターフェース**が SQL 実行の入口になる。実装は自分で書かず、`mybatis-spring-boot-starter` が実行時に自動生成して Spring の Bean として登録してくれる。

```kotlin
@Mapper
interface RoomMapper {
    fun insert(room: Room)

    fun selectByUserId(
        @Param("userId") userId: UUID,
    ): List<Room>
}
```

- メソッド 1 つが SQL 1 つに対応する。どの SQL かは、メソッド名（`id`）で XML（またはアノテーション）と結びつく。
- インターフェースを定義するだけで、`@Autowired` / コンストラクタ注入で使える。

### 引数のバインド（`@Param` の使いどころ）

SQL 側からは `#{名前}` でメソッド引数を参照する。

- **引数が 1 つでオブジェクト**（例: `Room`）なら、SQL から `#{name}` `#{gridX}` のようにそのプロパティを直接参照できる。`@Param` は不要。
- **引数が複数、または単純型 1 つ**のときは、どの引数を指すか名前で区別するために `@Param("userId")` を付ける。SQL からは `#{userId}` で参照する。

### 戻り値の型

| 取得結果 | 戻り値の型 |
|---|---|
| 1 行（見つからなければ null） | `Room?` |
| 複数行 | `List<Room>` |
| 件数など | `Int` / `Long` |
| INSERT/UPDATE/DELETE の影響行数 | `Int`（不要なら戻り値なしでもよい） |

掃除アプリでは一覧取得を `List<Room>` で受けている。0 件なら空リストが返る（null ではない）。

---

## XML vs アノテーション

SQL の置き場所には 2 通りある。

### アノテーション方式

メソッドに直接 SQL を書く。

```kotlin
@Select("SELECT id, name FROM room WHERE id = #{id}")
fun selectById(@Param("id") id: UUID): Room?
```

- **向き**: 短い・固定の SQL。1 〜 2 行で済むもの。
- **不向き**: SQL が長い、動的に組み立てる、`resultMap` を細かく指定したい場合。Kotlin の文字列リテラルに長い SQL を埋めると読みにくい。

### XML マッパー方式

SQL を `.xml` ファイルに分離する。

```xml
<mapper namespace="com.cleaningapp.layout.infrastructure.RoomMapper">
    <insert id="insert">
        INSERT INTO room (id, user_id, name, type, grid_x, grid_y, grid_w, grid_h, created_at, updated_at)
        VALUES (#{id}, #{userId}, #{name}, #{type}, #{gridX}, #{gridY}, #{gridW}, #{gridH}, #{createdAt}, #{updatedAt})
    </insert>
</mapper>
```

- **向き**: 長い SQL、動的 SQL（`<if>` 等）、複雑な `resultMap`。
- **不向き**: ごく短い SQL（ファイルが増えるだけ）。

### namespace とインターフェースの対応

XML の `namespace` は **Mapper インターフェースの完全修飾名**と一致させる。さらに各 SQL タグの `id`（`insert` / `selectByUserId`）を**メソッド名**と一致させることで、MyBatis がメソッドと SQL を結びつける。

application.yml で XML の置き場所を指定する。

```yaml
mybatis:
  mapper-locations: classpath:mapper/*.xml
```

掃除アプリでは、可読性と将来の動的 SQL を見据えて XML 方式を採用している。

---

## resultMap

`resultMap` は「SQL の結果カラム」を「Kotlin のオブジェクト」へどう詰めるかの対応表。

### カラム名 ↔ プロパティ名のマッピング（snake_case ↔ camelCase）

DB のカラムは snake_case（`grid_x`）、Kotlin のプロパティは camelCase（`gridX`）にしたい。これを 1 件ずつ書かずに自動変換するのが `map-underscore-to-camel-case` 設定。

```yaml
mybatis:
  configuration:
    map-underscore-to-camel-case: true
```

これで `grid_x` → `gridX`、`user_id` → `userId` のように自動でつながる。

### val（setter なし）の data class へマッピングする

ここが Kotlin 特有のポイント。MyBatis のデフォルトは「空コンストラクタでインスタンスを作り、setter で 1 つずつ値を入れる」方式。しかし `data class` を **全プロパティ `val`** で定義すると **setter が存在しない**ため、この方式では詰められない。

```kotlin
data class Room(
    val id: UUID,
    val userId: UUID,
    val name: String,
    val type: RoomType,
    val gridX: Int,
    // ... 全部 val
)
```

そこで `<resultMap>` の `<constructor>` を使い、**コンストラクタ経由**でマッピングする。

```xml
<resultMap id="RoomResult" type="com.cleaningapp.layout.domain.Room">
    <constructor>
        <idArg column="id" javaType="java.util.UUID"/>
        <arg column="user_id" javaType="java.util.UUID"/>
        <arg column="name" javaType="java.lang.String"/>
        <arg column="type" javaType="com.cleaningapp.layout.domain.RoomType"/>
        <arg column="grid_x" javaType="_int"/>
        <arg column="grid_y" javaType="_int"/>
        <arg column="grid_w" javaType="_int"/>
        <arg column="grid_h" javaType="_int"/>
        <arg column="created_at" javaType="java.time.Instant"/>
        <arg column="updated_at" javaType="java.time.Instant"/>
    </constructor>
</resultMap>
```

注意点:

- `<arg>` は**コンストラクタの引数の順番どおり**に並べる必要がある。プロパティ名ではなく順番で割り当たるので、`data class` の宣言順と一致させる。
- 主キーには `<idArg>` を使う（キャッシュ最適化のヒントになる）。
- `javaType` で型を明示する。プリミティブ int は `_int` のように先頭にアンダースコアを付ける。

`<select>` から `resultMap="RoomResult"` を指定して使う。

```xml
<select id="selectByUserId" resultMap="RoomResult">
    SELECT id, user_id, name, type, grid_x, grid_y, grid_w, grid_h, created_at, updated_at
    FROM room
    WHERE user_id = #{userId}
    ORDER BY created_at
</select>
```

### ネストした関連（`association` / `collection`）

1 対 1 は `<association>`、1 対多は `<collection>` で、子オブジェクトを入れ子にマッピングできる。1 対多を JOIN した 1 クエリで取ると、親が子の数だけ重複して返るため、MyBatis が `id` を手がかりに重複をまとめる。`<idArg>` / `<id>` を正しく指定しておくことが、このまとめ処理が効く前提になる。

---

## 動的SQL

XML 方式では、条件に応じて SQL を組み立てるタグが使える。

### `<if>` / `<where>`

```xml
<select id="search" resultMap="RoomResult">
    SELECT * FROM room
    <where>
        <if test="userId != null">
            AND user_id = #{userId}
        </if>
        <if test="type != null">
            AND type = #{type}
        </if>
    </where>
</select>
```

`<where>` は、中の条件が 1 つでも成立すれば `WHERE` を付け、先頭の余分な `AND` / `OR` を自動で取り除いてくれる。条件が全部 false なら `WHERE` 自体を出さない。

### `<foreach>` で IN 句を組み立てる

```xml
<select id="selectByIds" resultMap="RoomResult">
    SELECT * FROM room
    WHERE id IN
    <foreach item="id" collection="ids" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>
```

リストの要素を区切り文字でつなぎ、`IN (?, ?, ?)` を組み立てる。

### アノテーション方式の動的 SQL

アノテーションでも `@SelectProvider` などで、別クラスのメソッドに SQL 文字列を生成させる方式がある。ただし可読性は XML の方が高いので、動的 SQL が必要になったら XML へ寄せるのが扱いやすい。

---

## JPA との違い（Flyway が必須になる理由）

MyBatis は「SQL を自分で書く」ライブラリで、JPA（Hibernate）のような **エンティティからのテーブル自動生成（`ddl-auto`）を持たない**。つまり起動してもテーブルは勝手に作られない。

そのため、スキーマは**マイグレーションツールで明示的に管理**する必要がある。掃除アプリでは Flyway を使い、起動時に `classpath:db/migration` の `V*.sql` を順に適用してテーブルを作っている。

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
```

JPA なら `ddl-auto: update` でお手軽にテーブルが増える反面、本番でスキーマがいつ変わるか読みにくい。MyBatis + Flyway は「SQL もスキーマも自分が把握している」状態を保てるのがメリット。手間と引き換えに透明性が高い。
