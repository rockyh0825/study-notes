# study-notes

> 勉強のメモ・知識を蓄積するリポジトリ。分野ごとに整理し、MkDocsでWeb公開する。

---

## このリポジトリの責務

- 勉強内容をMarkdownでメモ・要約として蓄積する
- 分野（ラベル）ごとにノートを整理する
- Claude等と壁打ちして理解を深めた内容をまとめる
- MkDocsでGitHub Pagesとして公開し、ブラウザで綺麗に閲覧できるようにする

**※ 問題生成・知識定着・通知などの機能はすべて別リポジトリ（study-app）の責務であり、このリポジトリでは一切扱わない**

---

## 技術スタック

| 項目 | 技術 |
|---|---|
| ドキュメント生成 | MkDocs + Material テーマ |
| ホスティング | GitHub Pages |
| 自動デプロイ | GitHub Actions |

---

## ディレクトリ構成

```
study-notes/
├── CLAUDE.md
├── mkdocs.yml              # MkDocs設定ファイル
└── docs/                   # ノート置き場
    ├── index.md            # トップページ
    ├── java/               # Javaカテゴリ
    │   └── spring-boot.md
    ├── network/            # ネットワークカテゴリ
    │   └── osi-model.md
    └── ...                 # 分野ごとにディレクトリを追加
```

分野が増えたら `docs/` 配下にディレクトリを追加していく。

---

## MkDocs操作

```bash
# 依存インストール（初回のみ）
pip install mkdocs-material

# ローカルプレビュー
mkdocs serve
# → http://127.0.0.1:8000 で確認

# ビルド
mkdocs build

# GitHub Pagesへ手動デプロイ（通常はGitHub Actionsで自動）
mkdocs gh-deploy
```

---

## mkdocs.yml（基本設定）

```yaml
site_name: Study Notes
theme:
  name: material
  language: ja
  features:
    - navigation.tabs
    - navigation.top
    - search.suggest
nav:
  - Home: index.md
  - Java:
    - Spring Boot: java/spring-boot.md
  - Network:
    - OSI参照モデル: network/osi-model.md
```

分野・ファイルが増えたら `nav` に追記していく。

---

## GitHub Actions（自動デプロイ）

`.github/workflows/deploy.yml` を作成することで、`main` ブランチへのpush時に自動でGitHub Pagesへデプロイされる。

```yaml
name: Deploy MkDocs
on:
  push:
    branches:
      - main
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.x'
      - run: pip install mkdocs-material
      - run: mkdocs gh-deploy --force
```

---

## ノートの書き方ルール

- 1ファイル1トピックを基本とする
- ファイル名はケバブケース（例: `spring-boot.md`, `osi-model.md`）
- 各ファイルの先頭に概要を1〜2行で書く
- 壁打ちで得た理解・気づきも積極的に書き残す

---

## 分野ラベル（カテゴリ）例

必要に応じて `docs/` 配下に追加していく。

| ディレクトリ | 内容 |
|---|---|
| `java/` | Java / Spring Boot |
| `network/` | ネットワーク・プロトコル |
| `database/` | SQL / DB設計 |
| `docker/` | Docker / コンテナ |
| `cs/` | CS基礎（アルゴリズム・OS等） |
| `english/` | 英語 |