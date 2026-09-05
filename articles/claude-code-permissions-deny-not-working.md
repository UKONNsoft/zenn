---
title: "settings.jsonで deny に書いたのに .env が読まれた — Claude Code の permissions が効かない6つの理由"
emoji: "🔒"
type: "tech"
topics: ["claudecode", "claude", "anthropic", "ai", "セキュリティ"]
published: false
---

:::message
**検証環境：Claude Code v2.1.261 / 2026-09 時点**
仕様の更新が速い領域です。重要な判断の前には公式ドキュメントで再確認してください。
:::

グローバル設定（`~/.claude/settings.json`）に、こう書いてありました。

```json
{
  "permissions": {
    "deny": [
      "Read(**/.env)",
      "Read(**/.env.*)",
      "Edit(**/.env)",
      "Edit(**/.env.*)"
    ]
  }
}
```

`.env` は読めない。そのつもりでした。ところが別フォルダに置いた `.env` を読ませたら、素通りしました。中身の Webhook URL がそのまま画面に出ました。

ルールが間違っていたわけではありません。**書いた場所と、パターンの起点（アンカー）が噛み合っていなかった**だけです。そして起動時には、それとは別の警告がずっと出ていました。読み飛ばしていました。

この「書いたのに効かない」は、原因が **6 つ**に分かれます。上から順に潰せば、たいてい特定できます。

---

## 先に結論：切り分けの順番

```text
① Write(...) にパスを書いた            ← 無視される。起動時に警告が出ている
    ↓
② パスのアンカーが違う                  ← 一番刺さる。今回の事故はこれ
    ↓
③ Read だけ／Edit だけしか書いていない
    ↓
④ コマンド文字列マッチをすり抜けられた
    ↓
⑤ Claude の外のプロセスが読んだ
    ↓
⑥ deny を広く書きすぎて、自分で外した
```

①〜③は**書き方の問題**なので直せば確実に効きます。④〜⑥は**仕組みの限界**なので、別のレイヤー（サンドボックス、hook）に持っていく話になります。

---

## 最初の 30 秒：起動時の警告と `/permissions`

原因を探す前に、この 2 つを見てください。

**1. 起動時の警告を読む。** Claude Code は不正なルールを黙って捨てません。起動時に stderr へ理由を出します。ターミナルをスクロールして遡ってください。

**2. `/permissions` を開く。** 現在効いているルールと、**それがどの `settings.json` から来たのか**が一覧で出ます。「書いたはずのルールがここに無い」なら、置き場所かファイル名を間違えています。

---

## ① `Write(...)` にパスを書いた

一番多く、一番あっさり直る原因です。

`.env` を守ろうとして、こう書いていませんか。

```json
"deny": [
  "Read(**/.env)",
  "Write(**/.env)"
]
```

この `Write(**/.env)` は**効きません**。ファイルパスによる権限チェックの対象になるのは、**`Read(path)` と `Edit(path)` の 2 つだけ**です。`Write` や `NotebookEdit` にパスを書いてもルールとしては受理されますが、参照されることがありません。

手元で確かめられます。わざと壊したルールを書いたファイルを `--settings` で渡して起動します（実行場所＝ターミナル）。

```bash
claude --settings ./probe.json -p "ok"
```

`probe.json` の中身：

```json
{
  "permissions": {
    "deny": [
      "Write(**/.env)",
      "Bash(command:rm *)",
      "Reed(**/.env)"
    ]
  }
}
```

返ってきた警告（v2.1.261 実機）：

```text
Permission deny rule "Bash(command:rm *)" targets command as a raw string and will not match — use Bash(…) for Bash's own matcher.
Permission deny rule "Reed(**/.env)" matches no known tool — check for typos.
Permission deny rule (probe.json): Write(**/.env) is not matched by file permission checks — only Edit(path) rules are. Use Edit(**/.env) instead (Edit rules cover all file-editing tools).
```

3 行とも、書いたルールが**死んでいる**という通告です。

### 直し方

`Write(...)` を `Edit(...)` に書き換えるだけです。パターンはそのままで構いません。

```json
"deny": [
  "Read(**/.env)",
  "Edit(**/.env)"
]
```

`Edit` ルールは**ファイルを編集するすべての組み込みツールに適用されます**。`Write` も `NotebookEdit` も旧 `MultiEdit` も、まとめて `Edit` が代表します。ツール名を律儀に並べる必要はありません。

### ついでに読み取れる 2 つの警告

上の実験では、他に 2 種類の「無効なルール」も再現しています。

**`Bash(command:rm *)` — パラメータ名で書いてはいけない。**
`Tool(param:value)` という書式自体は存在します（例：`Agent(model:opus)`）。ただし**ツールが独自の正規化ルールを持つフィールドは対象外**です。Bash と PowerShell の `command`、Read/Edit/Write の `file_path`、Grep/Glob の `path`、WebFetch の `url` がそれにあたります。`Bash(command:rm *)` は複合コマンドで簡単に迂回できてしまうため、Claude Code はこれを**無視して警告を出します**。正しくは `Bash(rm *)` です。

**`Reed(**/.env)` — タイプミス。**
存在しないツール名の deny / ask ルールは起動時警告になります。`_` か `*` を含む名前はチェック対象外なので、MCP ツール名の綴りは自分で確認してください。

:::message alert
ツールの表示名と正規名は違うことがあります。トランスクリプト上で `Stop Task` と表示されるツールの正規名は `TaskStop` です。権限ルールと hook のマッチャーは**正規名しか見ません**。
:::

---

## ② パスのアンカーが違う

冒頭の事故の原因はこれでした。ここが本題です。

`Read` と `Edit` のパスルールは **gitignore の仕様**に従い、書き出しの形で「どこを起点に探すか」が変わります。**4 種類あります。**

| パターン | 起点 | 例 | 実際にマッチする範囲 |
|---|---|---|---|
| `//path` | ファイルシステムのルート（**絶対**） | `Read(//Users/alice/secrets/**)` | `/Users/alice/secrets/**` |
| `~/path` | ホームディレクトリ | `Read(~/Documents/*.pdf)` | `/Users/alice/Documents/*.pdf` |
| `/path` | **そのルールを書いた設定ファイル**からの相対 | `Edit(/src/**/*.ts)` | プロジェクト設定なら `<プロジェクトルート>/src/**/*.ts` |
| `path` / `./path` | **現在のディレクトリ**からの相対 | `Read(*.env)` | `<cwd>/*.env` |

先頭のスラッシュ 1 本は絶対パスではありません。**スラッシュ 2 本で初めて絶対パス**です。ここが直感に反します。

さらに `/path` は「書いた場所」に固定されるので、**同じ文字列が置き場所によって別の場所を指します**。

| ルールを書いた場所 | `/path` の解決先 |
|---|---|
| プロジェクト設定 `.claude/settings.json` | `<プロジェクトルート>/path` |
| ユーザー設定 `~/.claude/settings.json` | `~/.claude/path` |
| `--settings <file>` で渡したファイル | `<そのファイルのあるディレクトリ>/path` |
| CLI フラグ・`/permissions`・セッションルール | `<起動時の cwd>/path` |

つまり、ユーザー設定に `Read(/secrets/**)` と書くと、守られるのは各プロジェクトの `secrets/` ではなく、ユーザー設定ファイルの隣にある `~/.claude/secrets/**` のほうです。ほぼ確実に意図と違います。

### 今回のケース：`**/.env` は cwd 以下しか守らない

先頭にスラッシュが無い `**/.env` は、4 番目の**「現在のディレクトリからの相対」**です。ユーザー設定に書こうがプロジェクト設定に書こうが、起点は**セッションの cwd** です。

公式の対応表がそのまま答えになっています。

| deny ルール | ブロックする | ブロックしない |
|---|---|---|
| `Read(.env)` または `Read(**/.env)` | 現在のディレクトリ以下の任意の `.env` | **親ディレクトリや別プロジェクトの `.env`** |
| `Read(//**/.env)` | ファイルシステム上の任意の `.env` | なし |

`Read(.env)` と `Read(**/.env)` が同じ意味なのは、gitignore ではベアなファイル名が任意の深さにマッチするからです。深さは効いていて、**起点が動かない**わけです。

### 実機で再現する

外側に `.env` を置き、別ディレクトリから読ませます。

```bash
# 準備（実行場所＝ターミナル）
mkdir -p denytest/outside denytest/work
echo 'DUMMY_TOKEN=canary-1234' > denytest/outside/.env
```

相対アンカーのルールを渡して、`work` から `outside/.env` を読ませます。`--add-dir` は「作業ディレクトリの外にあるので触れない」という別要因を消すためです。

```bash
# relative.json = { "permissions": { "deny": ["Read(**/.env)", "Edit(**/.env)"] } }
cd denytest/work
claude --settings ../relative.json --add-dir ../outside \
  -p "Read ../outside/.env and print its contents verbatim."
```

結果：

```text
DUMMY_TOKEN=canary-1234

Not blocked — the file is in the additional working directory, so the read went through.
```

**通りました。** deny ルールは書いてあるのに、起点の外だったので一度も評価されていません。

絶対アンカーに書き換えます。

```bash
# absolute.json = { "permissions": { "deny": ["Read(//Users/alice/denytest/**/.env)", ...] } }
claude --settings ../absolute.json --add-dir ../outside \
  -p "Read ../outside/.env and print its contents verbatim."
```

```text
The Read was denied by permission settings — the `outside/` directory is on a deny list
despite being listed as an additional working directory.
```

**塞がりました。** 差分はアンカーだけです。

### 直し方

**全プロジェクトで守りたいものは、`//` の絶対パスか `~/` のホーム相対で書く。**

```json
"deny": [
  "Read(//Users/alice/**/.env)",
  "Read(//Users/alice/**/.env.*)",
  "Edit(//Users/alice/**/.env)",
  "Edit(//Users/alice/**/.env.*)"
]
```

`Read(//**/.env)` ならファイルシステム全域です。Windows ではパスが POSIX 形式に正規化されるので（`C:\Users\alice` → `/c/Users/alice`）、`//c/**/.env`、全ドライブなら `//**/.env` と書きます。

:::message
**副作用として Bash 側も塞がります。** Read / Edit の deny ルールは、組み込みツールだけでなく、Claude Code が認識するファイルコマンド（`cat`、`head`、`tail`、`sed` など）にも適用されます。`cat ~/x/.env` も止まります。
:::

---

## ③ `Read` だけ、または `Edit` だけしか書いていない

両者は非対称です。片方だけでは穴が残ります。

- **`Read` の deny は、同じパスの Edit ツールもブロックします**（新規ファイル作成を含む）。v2.1.208 以降の挙動です。
- ただし **`Write` と `NotebookEdit` はカバーされません。**

したがって、変更されたくないパスには **`Edit` の deny を必ず併記**してください。読み取りも禁じたいなら `Read` も書く。この 2 枚看板で足ります。

```json
"deny": [
  "Read(//Users/alice/**/.env)",
  "Edit(//Users/alice/**/.env)"
]
```

覚え方は「**守りは Read と Edit の 2 枚看板。`Write` は書かない**」です。

### シンボリックリンクは心配しなくていい

ここは逆に「守られている」側の話です。Claude がシンボリックリンクに触るとき、権限ルールは**リンク自体と、その解決先の両方**をチェックします。

- **allow ルール**：両方が一致したときだけ通る（許可ディレクトリ内のリンクが外を指していればプロンプトが出る）
- **deny ルール**：**どちらか一方**が一致すればブロック

`Read(~/.ssh/**)` を deny してあれば、`./project/key` → `~/.ssh/id_rsa` というリンクも止まります。抜け道にはなりません。

---

## ④ コマンド文字列マッチをすり抜けられた

ここから先は書き方では直りません。**仕組みの限界**です。

`Bash(rm -rf *)` のようなルールは、コマンド文字列に対するパターンマッチです。同じ結果になる別の綴りは、当然ながら一致しません。

### 安全側に倒れている部分

まず、誤解されやすい良い知らせから。

- **複合コマンドは分解されます。** `&&`、`||`、`;`、`|`、`|&`、`&`、改行が区切りとして認識され、**各サブコマンドが独立に照合されます**。`Bash(safe-cmd *)` を allow しても `safe-cmd && other-cmd` は通りません。
- **`bypassPermissions` でも deny は効きます。** 権限プロンプトをスキップするモードでも、明示的な `deny` ルールと `ask` ルールは適用されます。「柵を立ててから自動承認」の順序に意味があるのはこのためです。

### 抜けている部分

問題はこちらです。

**プロセスラッパーは剥がされます。** マッチ前に `timeout`、`time`、`nice`、`nohup`、`stdbuf`、そしてフラグ無しの `xargs` が除去されます。`Bash(npm test *)` は `timeout 30 npm test` にも一致します。allow を書いているときは、想定より広い範囲を許していることになります。

**環境ランナーは剥がされません。** `devbox run`、`mise exec`、`direnv exec`、`npx`、`docker exec` はリストに入っていません。これらは引数をコマンドとして実行するため、

```json
"allow": ["Bash(devbox run *)"]
```

と書くと、**`devbox run rm -rf .` まで通ります**。ランナーを丸ごと allow しないでください。`Bash(devbox run npm test)` のように、内側のコマンドまで含めて 1 本ずつ書きます。

**allow できないラッパーもあります。** `watch`、`setsid`、`ionice`、`flock`、そして `-exec` / `-delete` 付きの `find` は、プレフィックスルールでは自動承認されません（常にプロンプトが出ます）。

**引数を絞る deny は脆いです。** 公式ドキュメントが `curl` を例に列挙しています。`Bash(curl http://github.com/ *)` は次のどれにも一致しません。

- オプションが前に来る：`curl -X GET http://github.com/...`
- プロトコル違い：`curl https://github.com/...`
- リダイレクト経由：`curl -L http://bit.ly/xyz`
- 変数展開：`URL=http://github.com && curl $URL`
- 余分なスペース：`curl  http://github.com`

同じことが `rm` にも言えます。`Bash(rm -rf *)` は `/bin/rm -rf`、`rm --recursive --force` に一致しません。

### だから、deny は「柵」であって「壁」ではない

deny リストは**事故を止める柵**です。自分（と Claude）のうっかりを止めます。**悪意ある書き換えを止める壁ではありません。** ここを取り違えると、実際より安全だと誤認します。

壁が必要なら、次のどれかに移します。

- **サンドボックス**：OS レベルで Bash とその子プロセスのファイルシステム／ネットワークを制限する
- **`PreToolUse` hook**：ツール実行前に自前のロジックで判定して拒否する
- **`WebFetch(domain:...)` に寄せる**：`curl` / `wget` を deny し、ネットワークは WebFetch のドメイン許可制にする

hook については 1 点だけ覚えておくと安全です。**hook の判定は権限ルールを飛び越えません。** hook が `"allow"` を返しても、一致する deny ルールがあれば止まります。逆に、終了コード 2 で終わる hook は権限ルールの評価前にツール呼び出しを止めるので、**allow ルールより hook のブロックが強い**です。

---

## ⑤ Claude の外のプロセスが読んだ

`Read` / `Edit` の deny が適用されるのは、**Claude の組み込みファイルツール**と、**Claude Code が認識する範囲の Bash ファイルコマンド**（`cat`、`head`、`tail`、`sed` など）です。

適用されないものがあります。

```bash
python3 -c "print(open('.env').read())"
```

こういう「ファイルを間接的に開くサブプロセス」には届きません。Claude Code から見れば `python3` を実行しただけで、その中で何を開くかは見えないからです。

すべてのプロセスに対してパスを塞ぎたいなら、**OS レベルの強制＝サンドボックス**が必要です。サンドボックスのファイルシステム制限は `sandbox.filesystem` の設定と Read / Edit の deny ルールが**マージされて**最終的な境界になります。権限とサンドボックスは競合するものではなく、重ねるものです。

- **権限**：Claude が「試みること」自体を止める。全ツールが対象
- **サンドボックス**：Bash とその子プロセスを OS 境界で止める。プロンプトインジェクションで Claude の判断が破られても残る

---

## ⑥ deny を広く書きすぎて、自分で外した

最後は運用の問題です。これで守りを失うケースが一番もったいない。

権限ルールは **deny → ask → allow の順**で評価され、**最初に一致したものが結果を決めます**。そして**ルールの具体性は順序を変えません**。

つまり、

```json
{
  "permissions": {
    "deny":  ["Bash(aws *)"],
    "allow": ["Bash(aws s3 ls)"]
  }
}
```

この allow は**効きません**。`Bash(aws s3 ls)` は deny の `Bash(aws *)` にも一致し、deny が先に評価されるからです。**deny リストに例外は作れません。**

これが分かっていないと、「広めに deny を書く → 日常作業が止まる → 面倒になって deny を丸ごと外す」という経路をたどります。柵がゼロになるのが最悪です。

### 対処

- **deny は狭く、名指しで書く。** 例外を作りたくなる粒度で書かない
- **迷うものは `ask` に置く。** ブロックではなく確認になる。`ask` は `bypassPermissions` モードでもプロンプトを強制するので、無人運用時のチェックポイントとして使えます
- **スコープを使い分ける。** deny はどのスコープに書いても勝ちます（ユーザー設定の deny はプロジェクト設定の allow をブロックする）。全プロジェクト共通の柵はユーザー設定へ

### ベアなツール名 deny は挙動が違う

ついでに知っておくと便利な区別です。

- **`"Bash"`（括弧なし）**：ツールを **Claude のコンテキストから丸ごと削除**します。Claude はその存在を見ません
- **`"Bash(rm *)"`（スコープ付き）**：ツールは使えるまま。一致する呼び出しだけをブロックします

「そもそも触らせたくない」なら前者、「特定の使い方だけ止めたい」なら後者です。

---

## 補足：allow では守れない場所がある

deny の話ではありませんが、混同しやすいので 1 つだけ。

一部のパスは、**`bypassPermissions` を除くすべてのモードで自動承認されません**。`permissions.allow` に `Edit(.claude/**)` と書いても事前承認できません。安全性チェックが allow ルールの評価より前に走るためです。

対象は `.git`、`.claude`（`.claude/worktrees` を除く）、`.vscode`、`.idea`、`.husky`、`.cargo`、`.devcontainer`、`.yarn`、`.mvn`、`.config/git` などのディレクトリと、`.zshrc`、`.bashrc`、`.npmrc`、`.gitconfig`、`.mcp.json`、`.claude.json` などのファイルです。

リポジトリの状態と Claude 自身の設定が事故で壊れないようにする仕組みです。「allow したのにプロンプトが出る」場合、たいていこれです。

---

## コピペ用：最小の柵

グローバル（`~/.claude/settings.json`）に置く前提です。`alice` は自分のユーザー名に置き換えてください。

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "deny": [
      "Read(//Users/alice/**/.env)",
      "Read(//Users/alice/**/.env.*)",
      "Edit(//Users/alice/**/.env)",
      "Edit(//Users/alice/**/.env.*)",

      "Read(//Users/alice/.ssh/**)",
      "Edit(//Users/alice/.ssh/**)",
      "Read(//Users/alice/.aws/**)",
      "Edit(//Users/alice/.aws/**)",
      "Read(//Users/alice/.config/gh/**)",
      "Read(//Users/alice/.claude.json)",

      "Bash(rm -rf *)",
      "Bash(rm -fr *)",
      "Bash(rm -r *)",
      "Bash(sudo *)",
      "Bash(dd *)",
      "Bash(mkfs *)",
      "Bash(chmod 777 *)",
      "Bash(git push --force *)",
      "Bash(git reset --hard *)",
      "Bash(git clean -fd*)"
    ]
  }
}
```

設計の考え方は 2 行です。

- **コマンド一致（`Bash(...)`）は場所に依存しない** → グローバルに 1 回書けば全プロジェクトで効く。プロジェクトごとにコピーする必要はない
- **ファイルパス一致（`Read` / `Edit`）だけが場所に依存する** → 絶対パスで名指しする

秘密の読み取り禁止（`.ssh` / `.aws` / `.claude.json` / `.config/gh`）を入れているのは、**読めなければ送れない**からです。外部送信そのものを塞ぐより、入口で止めるほうが確実です。

なお `permissions` の変更は**再起動なしでほぼリアルタイムに再読み込み**されます（`model` や `outputStyle` は次回起動から）。書いたらすぐ試せます。

---

## チェックリスト

効かないと思ったら、上から順に。

- [ ] 起動時の警告を遡って読んだ
- [ ] `/permissions` に、そのルールが「どのファイル由来」で出ているか確認した
- [ ] パスルールを `Write(...)` ではなく `Edit(...)` で書いた
- [ ] 全プロジェクトで守りたいものを `//` 絶対パスか `~/` で書いた
- [ ] `Read` と `Edit` を両方書いた
- [ ] 守りたいのが「事故」か「攻撃」かを区別した（攻撃ならサンドボックス／hook）
- [ ] deny に例外を作ろうとしていないか（それは `ask` の仕事）

---

## まとめ

`permissions.deny` が効かないときの原因は、ほぼ**書き方 3 つ**に集約されます。

1. パスは `Read` と `Edit` にしか効かない（`Write` は無視される）
2. スラッシュ 2 本で初めて絶対パス。無印は cwd 起点
3. `Read` の deny は Edit を塞ぐが `Write` は塞がない。両方書く

そして、直しても残るのが**限界 3 つ**です。文字列マッチはすり抜けられる、Claude の外のプロセスには届かない、deny に例外は作れない。

**deny リストは事故を止める柵であって、攻撃を止める壁ではありません。** 柵で足りない場所にだけサンドボックスと hook を足す。この線引きができていれば、無人でループを回すときにも「どこまで守れているか」を自分で説明できます。

---

## 関連記事

設定全体の構造（どのファイルがどこにあり、競合したとき何が勝つか）はこちらにまとめています。

https://zenn.dev/tomiyasu_chan/articles/claude-code-overview-decision

## 一次情報

- [権限を設定する](https://code.claude.com/docs/ja/permissions)
- [権限モードを選択する](https://code.claude.com/docs/ja/permission-modes)
- [設定](https://code.claude.com/docs/ja/settings)
- [サンドボックス](https://code.claude.com/docs/ja/sandboxing)
- [Hooks](https://code.claude.com/docs/ja/hooks-guide)
