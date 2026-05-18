# cherry-pick したコミットによる conflict を rebase で解決する

別ブランチで開発中に「cherry-pick → PR マージ → conflict」という状況に遭遇したので、何が起きていたか・どう解消したかをまとめる。

---

## 状況

2 つのブランチを並行して作業していた。

| ブランチ | 内容 |
|---|---|
| `feature/few-change` | UI の文言修正・期日クリアボタン追加 |
| `feat/playwright-setup` | Playwright E2E テスト環境の構築 |

`feat/playwright-setup` でテストを書くには `feature/few-change` の UI 変更（クリアボタンなど）がこのブランチにも必要だった。そこで cherry-pick で取り込んだ。

```bash
git cherry-pick e63c3ed  # feature/few-change のコミットを持ってくる
```

この時点では問題なし。テストも通った。

---

## 何が起きたか

その後 `feature/few-change` が PR #3 としてマージされ、**main に取り込まれた**。

```
main:                  A --- B --- [e63c3ed がマージされた状態]
                              \
feat/playwright-setup:  B --- [ef35027(cherry-pick)] --- [Playwright コミット]
```

`feat/playwright-setup` には同じ変更が 2 つ存在する状態になった。

- `e63c3ed`（main 側） — PR #3 でマージされたオリジナル
- `ef35027`（ブランチ側） — cherry-pick で持ってきたコピー

この状態で `feat/playwright-setup` を main にマージしようとすると、**同じファイルへの同じ変更が 2 度適用されることになり conflict が発生する**。

---

## 解消方法：`git rebase`

`origin/main` にリベースすることで解消した。

```bash
git fetch origin
git rebase origin/main
```

出力：

```
warning: skipped previously applied commit ef35027
Successfully rebased and updated refs/heads/feat/playwright-setup.
```

Git が「cherry-pick したコミット（ef35027）は main にすでに含まれている」と検知し、**自動でスキップ**してくれた。結果、Playwright の作業コミット 1 件だけが残ったきれいな状態になった。

```
main:                  A --- B --- C(マージ済み)
                                    \
feat/playwright-setup:               [Playwright コミット]
```

---

## リベース後の push

`rebase` でコミット履歴が書き換わったため、通常の `git push` はリモートに拒否される。`--force` が必要だが、代わりに **`--force-with-lease`** を使う。

```bash
git push --force-with-lease origin feat/playwright-setup
```

### `--force` vs `--force-with-lease`

| コマンド | 挙動 |
|---|---|
| `--force` | リモートの状態を問わず強制上書き |
| `--force-with-lease` | **自分が最後に fetch した状態からリモートが変わっていなければ**上書き。他の人が push していた場合は失敗する |

チームで作業するときは `--force-with-lease` の方が安全。

---

## rebase が正解だったのか？merge でもよかった説

今回は `rebase` で解消したが、`git merge origin/main` でも同じ問題を解決できた。それぞれの挙動を比べる。

### merge した場合

```bash
git merge origin/main
```

git の 3-way merge は「共通の祖先・main・ブランチ」の 3 点を比較する。EditModal.tsx などの対象ファイルは **main 側もブランチ側も同じ内容に変わっている**ため、conflict とは判定されず**そのままマージが成功する**。

```
main:                  A --- B --- C
                              \       \
feat/playwright-setup:  B --- [ef35027] --- [Playwright] --- M(merge commit)
```

ただし cherry-pick コミット（`ef35027`）はブランチ履歴に**残ったまま**になる。force push も不要。

### rebase した場合（今回）

```bash
git rebase origin/main
```

cherry-pick 済みと検知して `ef35027` を自動スキップするため、Playwright コミットだけが残るきれいな線形履歴になる。一方で履歴の書き換えが発生するため `--force-with-lease` での push が必要。

### どちらを選ぶか

| | merge | rebase |
|---|---|---|
| conflict | 起きない | 起きない（自動スキップ） |
| 履歴 | cherry-pick コミットが残る | cherry-pick コミットが消える |
| PR の diff | UI 変更も混入する | Playwright の変更だけになる |
| force push | 不要 | 必要 |
| リスク | 低い | 履歴書き換えに注意 |

**今回のケース（個人 feature ブランチ）では rebase が適切**だった。PR の diff が Playwright の変更だけになり、レビューしやすい。

merge が向いているのは「とにかく安全に済ませたい」「shared branch など force push が難しい」場面。どちらが正解というより、**PR diff の見やすさと安全性のトレードオフ**で選ぶ。

---

## まとめ

```
cherry-pick → 元ブランチがマージ → 同じ変更が 2 箇所に → conflict
```

- `git rebase origin/main` で解消できる。Git が cherry-pick 済みのコミットを自動スキップしてくれる
- `git merge origin/main` でも conflict は起きない。履歴にコミットが残るが force push 不要で安全
- 個人 feature ブランチなら rebase、shared branch や安全優先なら merge
- リベース後の push は `--force-with-lease` で他の人の変更を上書きするリスクを防ぐ
