# Package by Feature 構成

機能単位でパッケージを切る「Package by Feature」の考え方と、レイヤー単位構成との違いをまとめる。

---

## Package by Layer との違い

パッケージ（ディレクトリ）の切り方には大きく 2 通りある。

### Package by Layer（レイヤー単位）

技術的な役割でまとめる。`controller` / `service` / `repository` といった層ごとにディレクトリを作り、全機能のクラスをそこへ放り込む。

```
com.example/
├── controller/
│   ├── RoomController.kt
│   ├── TaskController.kt
│   └── UserController.kt
├── service/
│   ├── RoomService.kt
│   ├── TaskService.kt
│   └── UserService.kt
└── repository/
    ├── RoomRepository.kt
    ├── TaskRepository.kt
    └── UserRepository.kt
```

「Room の機能を直す」とき、`controller` / `service` / `repository` の **3 つの離れたディレクトリ**を行き来することになる。機能は層をまたいで散らばる。

### Package by Feature（機能単位）

業務上の機能でまとめる。`room` / `task` / `user` といった feature ごとにディレクトリを作り、その機能に必要なものを全部そこへ入れる。

```
com.example/
├── room/
│   ├── RoomController.kt
│   ├── RoomService.kt
│   └── RoomRepository.kt
├── task/
│   ├── TaskController.kt
│   ├── TaskService.kt
│   └── TaskRepository.kt
└── user/
    └── ...
```

「Room の機能を直す」とき、触るファイルは `room/` の中にほぼ集まる。**1 つの変更が 1 つのディレクトリで完結**しやすい。

### 何が違うのか

| 観点 | Package by Layer | Package by Feature |
|---|---|---|
| まとめる軸 | 技術的な層 | 業務上の機能 |
| 1 機能の変更で触る範囲 | 複数ディレクトリに散る | 1 ディレクトリに集まる |
| 機能の追加・削除 | 各層に分散して作業 | ディレクトリ単位で完結 |
| ディレクトリ数の増え方 | 横（層）に固定、縦に肥大 | 機能ごとに増える |

---

## feature 内のレイヤー

Package by Feature でも層の概念は捨てない。**feature ディレクトリの中に、さらにレイヤーを持つ**のが実用的な構成。掃除アプリのバックエンドは `layout` feature をこう切っている。

```
layout/
├── presentation/      # 入口（Controller・DTO）
│   ├── RoomController.kt
│   └── RoomDtos.kt
├── application/       # ユースケース（操作の流れ）
│   ├── AddRoomUseCase.kt
│   └── ListRoomsUseCase.kt
├── domain/            # ドメインモデルと抽象（純粋なビジネスルール）
│   ├── Room.kt
│   ├── RoomType.kt
│   └── RoomRepository.kt   # ← インターフェース（ポート）
└── infrastructure/    # 技術的な実装（DB・外部連携）
    ├── RoomMapper.kt
    ├── RoomMapper.xml
    └── RoomRepositoryImpl.kt   # ← domain のインターフェースを実装
```

依存の向きは `presentation → application → domain` が基本で、`infrastructure` は `domain` のインターフェースを実装する形で内向きに依存する（依存性逆転）。`domain` は他のどの層にも依存しない純粋な Kotlin に保つ。

### feature 内で完結させる範囲

その機能だけが使うモデル・SQL・ユースケースは、すべてその feature の中に閉じ込める。外から見える入口は `presentation`（API）と、他 feature に提供する必要がある場合の限定的な公開窓口だけにする。

### 共通処理（shared / common）の置き場所

複数 feature がどうしても共有するもの（共通の例外・ユーティリティ・横断設定）は、`shared/` や `common/` といった専用ディレクトリにまとめる。ただし「とりあえず共通」に何でも入れると肥大化するので、**本当に複数 feature で使うものだけ**を慎重に置く。

---

## メリット・デメリット

### メリット

- **凝集度が高い**: 関連するコードが 1 か所に集まる。機能を理解するのにディレクトリを 1 つ見れば済む。
- **変更の局所化**: 1 機能の修正が 1 ディレクトリで完結しやすく、影響範囲が読みやすい。
- **削除が容易**: 機能を捨てるときディレクトリごと消せる。レイヤー構成のように各層から関連コードを探し回らなくてよい。
- **並行作業しやすい**: 別々の feature を別々の人が触ってもコンフリクトしにくい。

### デメリット

- **共通化の判断が難しい**: 「これは共通か、ある feature 固有か」の線引きに迷う。早すぎる共通化も、重複の放置もどちらも問題になる。
- **feature 間の線引き**: どこまでを 1 つの feature とするか（粒度）の設計判断が必要。大きすぎても小さすぎても扱いにくい。

### どんな規模で効くか

機能数が増えて、レイヤー構成だと各層のディレクトリが肥大化し始めた頃から効いてくる。逆にごく小さなプロジェクトでは、レイヤー構成の方が単純で十分なこともある。掃除アプリは feature が増えていく前提なので、最初から Package by Feature を採用している。

---

## feature 間の依存の課題

Package by Feature の弱点は、**feature 同士が直接 import し合うと境界が崩れる**こと。たとえば `task` feature が `room` feature の内部クラスを直接 import し始めると、両者が密結合になり、循環依存や「片方を消すともう片方が壊れる」状態に陥る。せっかく機能を分離した意味が薄れてしまう。

これを防ぐには「feature 間は直接 import しない」というルールと、依存の向きを制御する仕組みが要る。その具体策が次の Capability パターン。

参考: [Capability パターン](capability-pattern.md)
