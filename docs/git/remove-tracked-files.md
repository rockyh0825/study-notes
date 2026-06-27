# push 済みファイルを PR から除外する

すでに `git add` → `commit` → `push` してしまったファイルを、追跡対象から外してリモートからも消す方法。

---

## やりたいこと

- **ローカルには残したい**（削除したくない）
- **git の追跡からは外したい**（今後の差分に出てこないようにしたい）
- **リモート（GitHub）からも消したい**（PR の diff に映らないようにしたい）

典型例：IDE の設定フォルダ（`.idea/`）を間違えてコミット＆push してしまったケース。

---

## 手順

### 1. `.gitignore` に追記する

```
.idea/
```

これだけでは「今後 git add しない」だけ。**すでに tracking されているファイルには効かない**。

### 2. `git rm --cached` で tracking を外す

```bash
# 単一ファイルの場合
git rm --cached path/to/file

# ディレクトリごと外す場合（-r = recursive）
git rm -r --cached .idea/
```

| オプション | 意味 |
|-----------|------|
| `--cached` | インデックス（ステージング）からだけ削除。**ローカルファイルはそのまま** |
| `-r` | ディレクトリを再帰的に処理 |

`--cached` を付けないと **ローカルのファイルごと消える**ので注意。

### 3. commit して push する

```bash
git add .gitignore
git commit -m "chore: .idea/ を gitignore に追加し tracking から除外"
git push
```

push すると、リモートブランチの当該ファイルが **削除コミットとして反映**される。PR にも「削除」として表示されて diff から消える。

---

## なぜこれで動くのか

git には「作業ツリー（ローカルファイル）」と「インデックス（次のコミット候補）」という 2 つの概念がある。

```
作業ツリー   →(git add)→   インデックス   →(git commit)→   リポジトリ
```

`git rm --cached` は**インデックスからだけ**エントリを削除する操作。  
その状態でコミットすると「このファイルは無い」というスナップショットが積まれ、リモートに push されることでリモート側でも消える。

`.gitignore` はあくまで「未追跡ファイルを無視する」設定であり、**一度でも追跡されたファイルは `.gitignore` に書いても無視されない**。だから `git rm --cached` とセットで使う必要がある。

---

## よくある間違い

| 操作 | 結果 |
|------|------|
| `.gitignore` だけ追記 | すでに tracked なファイルには効かない。次の `git add .` でまた staging される |
| `git rm`（`--cached` なし） | ローカルファイルごと削除される |
| `git rm --cached` だけして push しない | リモートには残ったまま |

---

## 今回の実例（cleaning-app）

`.idea/` フォルダを含むコミットを push した後、以下を実行して PR の diff から除外した。

```bash
# .gitignore に追加
echo ".idea/" >> .gitignore

# tracking 解除（ローカルには残る）
git rm -r --cached .idea/

# commit & push
git add .gitignore
git commit -m "chore: .idea/ を gitignore に追加し tracking から除外"
git push
```

→ リモートの `.idea/` が削除コミットとして反映され、PR の diff から消えた。
