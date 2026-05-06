# GitHub Actions と CI/CD の基礎

CI/CD の概念と、GitHub Actions のワークフローファイル（`.github/workflows/*.yml`）の書き方をまとめた。

---

## CI/CD とは

### CI（継続的インテグレーション）

コードを push するたびに**自動でテスト・ビルドを実行**する仕組み。

```
push → テスト実行 → ビルド確認 → 問題があれば通知
```

「インテグレーション」は「複数人の変更を main に統合する」という意味。手動でテストを実行するのを忘れたり、ローカル環境の差異でバグを見落としたりするのを防ぐ。

### CD（継続的デリバリー / デプロイ）

テストが通ったら**自動でデプロイまで走らせる**仕組み。

- **Continuous Delivery（継続的デリバリー）**: デプロイできる状態を常に保つ。本番反映は手動承認
- **Continuous Deployment（継続的デプロイ）**: テストが通れば自動で本番まで反映

個人開発では後者（テスト通過 → そのまま本番反映）が多い。

### パイプライン

CI/CD の一連の処理フロー全体を指す言葉。

```
push → lint → test → build → deploy
```

各ステップが前のステップの成功を前提に続く「パイプ」のようなイメージ。

---

## GitHub Actions の構成要素

### ワークフロー（Workflow）

`.github/workflows/` 以下に置く YAML ファイル1つ = ワークフロー1つ。複数置ける。

```
.github/
└── workflows/
    ├── deploy.yml   # デプロイ用
    └── test.yml     # テスト用
```

### イベント（Event）

ワークフローを起動するトリガー。`on:` で指定する。

```yaml
on:
  push:                        # push 時
    branches: [main]           # main ブランチのみ
  pull_request:                # PR 作成・更新時
    branches: [main]
  schedule:
    - cron: '0 9 * * 1'        # 毎週月曜 9:00 UTC
  workflow_dispatch:           # 手動実行ボタンを表示
```

### ジョブ（Job）

`jobs:` 以下に定義する実行単位。**デフォルトで並列実行**される。

```yaml
jobs:
  test:       # ジョブ名（任意）
    runs-on: ubuntu-latest
    steps: ...

  deploy:
    needs: test   # test ジョブが成功してから実行
    runs-on: self-hosted
    steps: ...
```

`needs:` で依存関係を指定すると直列にできる。

### ステップ（Step）

ジョブ内の処理の1単位。**上から順番に実行**される。1つ失敗すると以降はスキップ。

```yaml
steps:
  - name: チェックアウト         # name は省略可。ログに表示される
    uses: actions/checkout@v4   # 公式 Action を使う

  - name: テスト実行
    run: uv run pytest          # シェルコマンドを実行
```

`uses` と `run` の使い分け:
- `uses`: 誰かが作った Action（処理のまとまり）を再利用する
- `run`: シェルコマンドをそのまま書く

### ランナー（Runner）

ジョブを実行するマシン。

| 種類 | 説明 |
|---|---|
| `ubuntu-latest` / `windows-latest` / `macos-latest` | GitHub が管理するクラウドマシン。無料枠あり |
| `self-hosted` | 自分のサーバーに Runner をインストールして使う |

Self-hosted は GitHub に課金せずに使えるが、サーバーの管理は自前。

---

## ワークフロー YAML の書き方

### 基本構造

```yaml
name: ワークフロー名（GitHub UI に表示される）

on:
  push:
    branches: [main]

jobs:
  job-name:
    runs-on: ubuntu-latest
    steps:
      - name: ステップ名
        run: echo "hello"
```

### 環境変数

```yaml
jobs:
  deploy:
    runs-on: self-hosted
    env:
      APP_DIR: /home/<username>/app   # ジョブ全体で使える変数

    steps:
      - name: 変数を使う
        run: echo "$APP_DIR"
        env:
          STEP_VAR: ステップ限定の変数  # このステップだけで使える
```

### Secrets（秘密情報）

API キーなどは `${{ secrets.変数名 }}` で参照。GitHub の **Settings → Secrets** で登録する。

```yaml
steps:
  - name: デプロイ
    run: curl -H "Authorization: Bearer $TOKEN" https://api.example.com
    env:
      TOKEN: ${{ secrets.API_TOKEN }}  # ログに *** でマスクされる
```

### 公式 Action の使い方

`uses: owner/repo@version` の形式。

```yaml
steps:
  - uses: actions/checkout@v4          # リポジトリをチェックアウト
  - uses: actions/setup-python@v5      # Python のセットアップ
    with:
      python-version: '3.12'           # Action へのパラメータは with: で渡す
```

よく使う公式 Actions:

| Action | 用途 |
|---|---|
| `actions/checkout@v4` | リポジトリのコードを取得 |
| `actions/setup-python@v5` | Python バージョンを指定してセットアップ |
| `actions/setup-node@v4` | Node.js のセットアップ |
| `actions/cache@v4` | 依存関係のキャッシュ |
| `actions/upload-artifact@v4` | ビルド成果物を保存 |

### `$GITHUB_PATH` — PATH の追加

Runner の実行環境は `.bashrc` を読まないため、インストール済みのツールが PATH に入っていないことがある。`$GITHUB_PATH` に追記することで**後続ステップ全体に反映**できる。

```yaml
- name: uv を PATH に追加
  run: echo "$HOME/.local/bin" >> $GITHUB_PATH

- name: uv を使う（PATH が通っている）
  run: uv sync
```

`export PATH=...` をステップ内に書いてもそのステップ限りで消える。`$GITHUB_PATH` への追記が正しい方法。

### `$GITHUB_ENV` — 後続ステップへの変数渡し

```yaml
- name: バージョンを取得して変数にセット
  run: echo "VERSION=$(git describe --tags)" >> $GITHUB_ENV

- name: 変数を使う
  run: echo "デプロイバージョン: $VERSION"
```

ステップ間で値を渡したいときに使う。

### `working-directory`

```yaml
- name: テスト実行
  working-directory: ./backend   # このステップだけ作業ディレクトリを変更
  run: uv run pytest
```

### `if` — 条件付き実行

```yaml
- name: 失敗時の通知
  if: failure()          # 前のステップが失敗した場合のみ実行
  run: echo "失敗した"

- name: main のみ実行
  if: github.ref == 'refs/heads/main'
  run: ./deploy.sh
```

---

## 豆知識

**ワークフローファイルは複数置ける**

`test.yml` と `deploy.yml` を分けることで、「テストは PR でも走らせ、デプロイは main push のみ」といった使い分けができる。1ファイルに全部書く必要はない。

**ジョブはデフォルトで並列**

`needs:` を書かなければジョブは同時に走る。テストとリントを並列にしてパイプライン全体を速くできる。

**Self-hosted Runner はセキュリティリスクに注意**

パブリックリポジトリで Self-hosted Runner を使うと、悪意ある PR が fork から runner 上でコードを実行できてしまう。**プライベートリポジトリか、信頼できる contributor のみのリポジトリ**に限定して使うのが安全。

**`actions/checkout` を使わないケースがある**

Self-hosted Runner で「リポジトリは既にサーバーに clone 済みで、pull するだけでよい」場合は `actions/checkout` が不要。runner がサーバー上で直接 `git fetch + reset --hard` するほうがシンプル（[自宅サーバーへの自動デプロイ](github-actions-self-hosted-deploy.md) での構成）。

**ログは自動的にマスクされる**

`${{ secrets.XXX }}` で渡した値はログ出力時に自動で `***` に置き換わる。誤って `echo $SECRET` してもログに漏れない。

**`workflow_dispatch` で手動実行ボタンを追加できる**

```yaml
on:
  push:
    branches: [main]
  workflow_dispatch:   # これを追加するだけで GitHub UI に "Run workflow" ボタンが出る
```

緊急の再デプロイや、push なしで試し実行したいときに便利。
