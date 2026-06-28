# Capability パターン

feature 間の直接依存を避けるための Capability パターン（依存の逆転）をまとめる。

---

## 解決したい課題

Package by Feature で機能を分割しても、**feature 同士が直接 import し合う**とモジュール境界はあっさり崩れる。

```ts
// task feature が room feature の中身を直接 import している（アンチパターン）
import { RoomRepository } from "../room/infrastructure/RoomRepository";
```

これが起きると次の問題が生まれる。

- **双方向依存**: `task → room` だけでなく `room → task` も生まれ、どちらが上位か分からなくなる。
- **循環依存**: 互いを import し合い、片方を変更するともう片方も壊れる。ビルドツールによっては循環自体がエラーになる。
- **削除・差し替えが困難**: feature を 1 つ消したいのに、他 feature が内部を握っていて消せない。

Package by Feature は「機能を独立した塊に保つ」のが前提なのに、直接 import はその前提を破壊する。だから「feature 間は直接 import しない」というルールが要る。ではどう連携するのか、を解くのが Capability パターン。

参考: [Package by Feature 構成](package-by-feature.md)

---

## 使う側視点の interface（capability）

核となる発想は「**使う側**が、自分に必要な操作を interface として定義する」こと。提供側の都合ではなく、利用側が欲しい機能（= capability）を宣言する。

たとえば `task` feature が「部屋名を表示したい」とする。`room` の実装を import する代わりに、`task` が**欲しい操作だけ**を interface で定義する。

```ts
// capabilities/RoomNameProvider.ts
// 「使う側（task）」が必要とする操作だけを宣言する
export interface RoomNameProvider {
  getRoomName(roomId: string): Promise<string | null>;
}
```

この interface を `capabilities/` という中立な場所に置く。`task` はこの interface にだけ依存し、`room` の実装は知らない。

そして**提供側の `room` feature が、その interface を実装する**。

```ts
// room/RoomNameProviderImpl.ts
import { RoomNameProvider } from "../capabilities/RoomNameProvider";

export class RoomNameProviderImpl implements RoomNameProvider {
  async getRoomName(roomId: string) {
    // room 内部のリポジトリを使って実装
    return this.repo.findName(roomId);
  }
}
```

### 依存の向きがどう逆転するか

直接 import だと依存はこうなる。

```
task ──直接 import──▶ room      （task が room の内部に依存）
```

Capability パターンでは、両者が中立な interface を向く。

```
task ──依存──▶ RoomNameProvider (interface) ◀──実装── room
```

`task` も `room` も「interface」という抽象に依存し、`room` から `task` への矢印（実装の提供）が**逆向き**になる。これが依存の逆転（DIP）。`task` は `room` の存在すら知らなくてよくなる。

---

## DI 配線

interface と実装をどこかで結びつける必要がある。これを各 feature の中でやると依存が漏れるので、**配線専用の 1 か所に集約**する。フロントエンドなら `di.ts` のようなファイル。

```ts
// di.ts — 実装の組み立てはここだけが知っている
import { RoomNameProviderImpl } from "./room/RoomNameProviderImpl";
import { createTaskService } from "./task/TaskService";

const roomNameProvider = new RoomNameProviderImpl(/* ... */);

// task には interface 型として実装を渡す
export const taskService = createTaskService({ roomNameProvider });
```

- `task` は**コンストラクタや引数で interface を受け取るだけ**で、どの実装が来るかを知らない。
- 「`RoomNameProvider` の実装は `RoomNameProviderImpl`」という知識は `di.ts` だけが持つ。
- バックエンド（Spring）なら、この役割は DI コンテナが担う。`RoomRepository`（interface）に `RoomRepositoryImpl` を `@Repository` 登録して注入するのが同じ構図。

### テスト時のモック差し替え

配線が 1 か所に集約され、利用側が interface しか見ていないので、テストでは実装をモックに差し替えるだけでよい。

```ts
const fakeRoomNameProvider: RoomNameProvider = {
  getRoomName: async () => "テスト部屋",
};
const service = createTaskService({ roomNameProvider: fakeRoomNameProvider });
```

`room` の本物の DB やネットワークを動かさずに `task` を単体テストできる。

---

## メリット・注意点

### メリット

- **疎結合**: feature が他 feature の実装を知らないので、片方の変更が他方に波及しにくい。
- **差し替え可能**: 実装を別物に入れ替えても、利用側は interface しか見ていないので影響を受けない。テストのモック化も容易。
- **境界の明確化**: feature 間でやり取りできる操作が「capability の interface」に明文化される。何が公開され、何が内部かが一目で分かる。
- **循環依存の回避**: 依存が中立な interface へ一方向に流れるため、循環が生まれにくい。

### 注意点

- **interface が増えすぎる**: 連携のたびに interface を作ると、ボイラープレートが膨らむ。
- **過剰設計の懸念**: feature 連携がほとんどない小規模アプリでは、抽象の層がかえって読みにくさを生む。

### 採用する判断基準

「feature 同士の連携が実際に発生し、かつ直接 import による結合を避けたい」ときに使う。連携がそもそも無いなら不要。掃除アプリでは、機能が増えて feature 間連携が出てくる前提で、最初からこのパターンを設計方針に据えている。

参考: [Package by Feature 構成](package-by-feature.md)
