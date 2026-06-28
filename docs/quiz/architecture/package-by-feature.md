# クイズ: Package by Feature 構成

機能単位のパッケージ構成（Package by Feature）と、レイヤー単位構成（Package by Layer）の違い・トレードオフを確認するクイズです。

---

### Q1. Package by Layer と Package by Feature の違いを、ディレクトリの切り方の観点で説明してください。

??? success "解答"
    - Package by Layer は `controller/` `service/` `repository/` のように「技術レイヤー」でまとめる
    - Package by Feature は `user/` `order/` `cleaning/` のように「機能（ドメイン）」でまとめ、その中にレイヤーを持つ
    - 切る軸が「技術の種類」か「機能の単位」かが本質的な違い
    - 参考: （公式ドキュメントを後で追記）

### Q2. 1つの機能追加・変更を行うとき、Package by Layer と Package by Feature では触るファイルの「散らばり方」がどう違いますか。

??? success "解答"
    - Package by Layer では controller / service / repository と複数のディレクトリにまたがって変更が散らばる
    - Package by Feature では関係ファイルが同じ feature ディレクトリに集まっているため変更が局所化する
    - これが Package by Feature の「凝集度が高い」と言われる理由
    - 参考: （公式ドキュメントを後で追記）

### Q3. Package by Feature の代表的なメリットを3つ挙げてください。

??? success "解答"
    - 凝集度が高い（関連コードが1か所にまとまる）
    - 変更の局所化（1機能の変更が他に波及しにくい）
    - 削除が容易（機能を消すときディレクトリごと消せる）
    - 加えて、機能単位でオーナーシップ・モジュール境界を引きやすい
    - 参考: （公式ドキュメントを後で追記）

### Q4. Package by Feature のデメリット・難しさは何ですか。

??? success "解答"
    - 共通化の判断が難しい（どこまでを shared/common に切り出すか）
    - feature 間の線引きが曖昧になりやすい（どの feature に属すか迷う）
    - feature 同士が直接依存し始めると境界が崩れる
    - 参考: （公式ドキュメントを後で追記）

### Q5. feature をまたいで使う共通処理は、どこに置くのが定石ですか。

??? success "解答"
    - `shared/` や `common/` のような共通領域に置く
    - ただし「本当に複数 feature で共有されるもの」だけに限定する（早すぎる共通化は避ける）
    - 特定 feature 固有のロジックを安易に shared に上げると、結局 shared が巨大化して凝集度が下がる
    - 参考: （公式ドキュメントを後で追記）

### Q6. Package by Feature における最大の課題「feature 間の依存」は、なぜ問題になり、どう解決しますか。

??? success "解答"
    - feature 同士が直接 import し合うと、双方向依存・循環依存が生まれモジュール境界が崩れる
    - 結果として「機能ごとに独立」という Package by Feature の前提が壊れる
    - 解決策として、interface 経由で依存方向を制御する Capability パターン（依存の逆転）を用いる
    - 参考: [Capability パターン](capability-pattern.md)

### Q7. Package by Feature は、どんなプロジェクト規模・状況で効果が大きいですか。

??? success "解答"
    - 機能数が多く、機能ごとに変更・追加・削除が頻繁に起きる中〜大規模で効きやすい
    - 複数人・複数チームで機能単位に分担する場合に境界が明確になる
    - 逆にごく小規模ではレイヤー構成の方がシンプルで十分なこともある
    - 参考: （公式ドキュメントを後で追記）
