# 掃除アプリ 開発ログ

掃除管理アプリの開発を時系列で記録する。新しい記録は上に追記していく。

---

## 2026-06-27（スライス2）

### やったこと

- **`Furniture` / `Part` のドメインモデルを追加**（`domain` 層）。Room と同様、全フィールド `val` の data class。変更は `copy()` で行う。
- **`FurnitureRepository` / `PartRepository` の interface を追加**（domain 層のポート）。infrastructure 層の MyBatis 実装は次のスライスで追加する。
- **`RoomType.presetParts()` を実装**。部屋種別ごとにプリセットパーツの雛形（`PresetPartDefinition`）リストを返す。Kotlin の enum は関数を持てる。`when(this)` で全種別を網羅する（分岐漏れをコンパイラが検出できる）。
- **V2 Flyway マイグレーション**（`V2__layout_furniture_part.sql`）で `furniture` / `part` テーブルを作成。
- **`OwnerType` / `PresetPartDefinition` を独立ファイルに切り出し**（当初はそれぞれ `Part.kt` / `RoomType.kt` の中に定義していた）。1 ファイル 1 概念の原則。

### 設計判断

- **ポリモーフィック関連（Polymorphic Association）で Part の所属先を表現**。`owner_type`（TEXT）と `owner_id`（UUID）の組み合わせで、Part が Room にも Furniture にも属せる設計にした。詳細は [アーキテクチャ決定ログ](architecture-decisions.md) に記録。

- **`part.owner_type` に CHECK 制約を追加**。最初はアプリ層（`OwnerType` enum）だけで値域を担保していたが、`CHECK (owner_type IN ('ROOM', 'FURNITURE'))` を DB 層にも追加した。バグや直接 SQL 操作など、アプリ層を経由しない書き込みに対しても DB が最後の砦として機能する。この「二重防衛」の考え方は CHECK 制約の典型的な使い方。

### ハマったこと / 気づき

- **`AddRoomUseCase` はまだ `presetParts()` を呼んでいない**。ドメイン層のモデルと SQL テーブルは用意したが、「部屋を追加したら Part を seed する」ロジックはスライス3 で application 層に追加する予定。現状 `presetParts()` は使われていない。

- **`part` の FK 制約なし問題**。`owner_id` は `room(id)` か `furniture(id)` を指すが、2 つの親テーブルを 1 カラムで参照するためFK制約は付けられない（→ ポリモーフィック関連の本質的なトレードオフ）。Room/Furniture 削除時の Part 孤立を防ぐため、application 層で `PartRepository.deleteByOwnerId` を呼ぶ経路を次のスライスで確定させる必要がある。

### 次にやること

- `AddRoomUseCase` で `RoomType.presetParts()` を呼んで Part を seed する（スライス3）。
- `FurnitureRepository` / `PartRepository` の MyBatis 実装（Mapper interface + XML）を追加する。
- Room / Furniture 削除時に Part を連鎖削除する経路を確定させる。

---

## 2026-06-27（スライス1）

### やったこと

- **Spring Boot 雛形を Spring Initializr で生成**。構成は Gradle (Kotlin DSL) / 言語 Kotlin / Java 21 / Spring Boot 3.5.16。依存は `web` / `mybatis` / `postgresql` / `flyway` / `validation` を選択。
- **docker-compose で PostgreSQL 16 を起動**。`application.yml` に DB 接続・Flyway・MyBatis の設定を書いた。

    ```yaml
    spring:
      datasource:
        url: jdbc:postgresql://localhost:5432/cleaning_app
        username: cleaning
        password: cleaning
      flyway:
        enabled: true
        locations: classpath:db/migration
    mybatis:
      mapper-locations: classpath:mapper/*.xml
      configuration:
        map-underscore-to-camel-case: true   # snake_case ↔ camelCase
    ```

- **Flyway V1 マイグレーションで `room` テーブルを作成**（`V1__layout_initial.sql`）。UUID 主キー・カラムは snake_case・`user_id` にインデックス。起動時に Flyway が自動適用する（MyBatis は JPA と違いテーブル自動生成がないため、スキーマは Flyway で明示管理する）。
- **`room` の CRUD 縦スライス（vertical slice）を実装**。1 機能を presentation 〜 infrastructure まで貫いて作る。
    - `domain`: `Room`（全 `val` の data class）/ `RoomType`（enum）/ `RoomRepository`（ポートの interface）
    - `infrastructure`: `RoomMapper`（`@Mapper` interface）+ `RoomMapper.xml` / `RoomRepositoryImpl`（`RoomRepository` の実装）
    - `application`: `AddRoomUseCase` / `ListRoomsUseCase`
    - `presentation`: `RoomController` / DTO（`RoomDtos`）
- **API 契約を `openapi.yaml` にスライス1分だけ記述**（`POST /rooms` と `GET /rooms`）。出力用の `Room` と入力用の `RoomCreate` をスキーマとして分けた。

### ハマったこと / 気づき

- **Initializr のデフォルトが Spring Boot 4.1.0 で、MyBatis（mybatis-spring-boot-starter）が未対応だった**。`bootVersion=3.5.16` に固定して解決。
    - バージョン ID は **`3.5.16`** で指定する。`.RELEASE` を付ける（`3.5.16.RELEASE`）と Initializr が 500 を返す。
- **`val` 中心の data class は setter が無い**ため、MyBatis のデフォルト（空コンストラクタ + setter）ではマッピングできない。`<resultMap>` の `<constructor>` でコンストラクタ経由マッピングにして解決。`<arg>` はコンストラクタの引数順に並べる必要がある（→ [MyBatis Mapper の書き方](../spring-boot/mybatis-mapper.md)）。

### 設計判断

- **エンドポイントを `/floor-plan` から `/rooms` に変更**。単一リソースの操作ではなく部屋の集合を扱うので、コレクション指向の `/rooms` の方が REST として自然。
- **機能名 `floorplan` を `layout` に改名**。パッケージも `com.cleaningapp.layout` に統一。

### 次にやること

- `room` の更新・削除（`PUT` / `DELETE /rooms/{id}`）を openapi と実装に追加してスライスを完成させる。
- ユーザー識別（MVP の初回発行 UUID）を `user_id` に流し込む経路を整える。
- アーキテクチャテスト（Konsist）で feature 境界・レイヤー依存方向を検証する仕組みを入れる。
