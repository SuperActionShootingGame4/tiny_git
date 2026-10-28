# TinyGit (tg) CLI 詳細設計書 — Go実装版 v5

- 版: 5.0（17機能限定・Git名称／オプション／用語統一版）
- 作成日: 2026-10-09
- 基準文書: `TinyGit_CLI_Design_Go_v4_2.md`。本書をv5の正式仕様とし、v4.2は旧版として保存する。
- 状態: Go版の実装前設計。既存C版の実績はGo版の品質根拠にしない。
- 実装: Go、標準Git CLI、シェル非経由、独自Gitオブジェクト／保存形式なし。
- 対象: Windows 11 x64、Linux x86-64（Debian 13）。Git 2.28以降。ARM64は対象外。
- 運用資料: `/home/subcon/AI_WORKING/旧ソフトウェア開発プロセス_ver1.04.doc`（第3、6～9章）。ブランチ運用を踏襲し、Gitリポジトリを新規作成する。hg履歴の変換は対象外。
- リモート: GitHub、AWS CodeCommit、その他Gitリモート。複数ref送信にはatomic push対応が必須。サービス別の対応は実接続試験で確認する。

## 1. 方針とv4.2からの変更

Mercurialで必要とした17種類の機能だけを提供し、CLIの名称、オプション、表示用語はGitへ寄せる。Gitで共通コマンドにまとまる機能を別名として増やさないため、公開サブコマンドは15個になる。Git全オプションの透過ラッパーではなく、本書の許可リストだけを受け付ける。

公開名は `init`、`clone`、`status`、`diff`、`add`、`rm`、`commit`、`fetch`、`switch`、`branch`、`merge`、`push`、`log`、`show-ref`、`tag` の15個に固定する。

`pull`、`update`、`remove`、`forget`、`branches`、`heads`、`resolve`、`tags`、`id`、`backout`、`doctor`、`help`、`version`というサブコマンドや旧名aliasは提供しない。説明と版表示は共通の`--help`／`--version`で行い、追加の業務機能には数えない。履歴改変、stash、ブランチ／タグ削除・改名、任意Gitコマンド実行は提供しない。

### 1.1 17機能との対応

| No. | hgで必要とした機能 | v5の呼出し | 対応範囲 |
|---:|---|---|---|
| 1 | `hg init` | `tg init [-b main] [DIR]` | Gitリポジトリ新規作成 |
| 2 | `hg clone` | `tg clone URL [DIR]` | originを設定しmainをcheckout |
| 3 | `hg status` | `tg status` | worktree／index／現在のbranchを確認 |
| 4 | `hg diff` | `tg diff [HEAD] [-- PATH...]` | Gitの比較対象を明示 |
| 5 | `hg add` | `tg add -- PATH...` | ファイルをindexへstage |
| 6 | `hg remove` | `tg rm [-r] -- PATH...` | indexとworktreeから削除 |
| 7 | `hg commit` | `tg commit [-a] -m MESSAGE [-- PATH...]` | indexまたは指定ファイルをcommit |
| 8 | `hg pull` | `tg fetch [origin] [--tags]` | 履歴を取得。worktreeは変えない |
| 9 | `hg update` | `tg switch BRANCH`／`tg merge --ff-only origin/BRANCH` | branch切替とfast-forward更新を分離 |
| 10 | `hg branch` | `tg branch NAME`／`tg switch -c NAME` | 前者は作成のみ、後者は作成して切替 |
| 11 | `hg branches` | `tg branch --list`／`tg branch -a` | ローカル／remote-tracking branch一覧 |
| 12 | `hg merge` | `tg merge SOURCE` | 統合準備。commitで確定 |
| 13 | `hg resolve` | `tg diff --name-only --diff-filter=U`＋`tg add`／`tg rm` | 競合一覧と解決済み登録 |
| 14 | `hg push` | `tg push [origin] [BRANCH]`／`tg push origin REV:BRANCH` | 全履歴または指定commitまで送信 |
| 15 | `hg log` | `tg log [-n N] [--graph] [--oneline] [--all]` | 履歴確認 |
| 16 | `hg heads` | `tg show-ref --heads`＋`tg branch -avv` | branch先端とローカル／remoteの分岐確認 |
| 17 | `hg tag` | `tg tag NAME [REV]`／`tg tag --list` | Gitタグ作成／一覧 |

### 1.2 用語と互換範囲

用語は「commit」「commit ID」「branch」「tag」「HEAD」「worktree」「index（staging area）」「remote」「remote-tracking branch」「fast-forward」「merge conflict」を用いる。Mercurialのchangeset番号、draft/public phase、named branchは実装しない。

Gitのbranchはcommitへのrefで、1つのbranch refに複数headを保存しない。`show-ref --heads`はローカルbranchの先端を列挙するもので、hgの複数headの完全再現ではない。並行作業による分岐は`branch -vv`のahead/behind表示とmerge前の祖先判定で確認する。

Gitに合わせ、引数なし`diff`はindex対worktree、`diff --cached`はHEAD対index、`diff HEAD`はHEAD対worktree。引数なし`commit`はstage済みだけを記録し、追跡済みの全変更を含める場合は`commit -a`を明示する。Gitタグは`.hgtags`を作らず、タグ作成自体でcommitを増やさない。

## 2. アーキテクチャ・実行環境

```text
cmd/tg → internal/cli → internal/ops
                         ├─ internal/policy
                         ├─ internal/state
                         └─ internal/git（Runner）→ 標準Git CLI
```

Go標準`flag`と手書きサブコマンドで解析する。各サブコマンドのオプションは位置引数より前に置く。共通`--help`、`--version`、`--timeout DURATION`を持ち、`--`以降をパスとして解釈するコマンドではその区切りを保持する。未知コマンド／オプションは2。Gitへの任意フラグ転送を禁止する。

`os/exec`は`internal/git`だけが利用し、`exec.CommandContext`へ引数配列で渡す。Gitは起動時に一度だけ絶対パスへ解決し、cwd探索をしない。Goの最低版／toolchainを`go.mod`に固定する。CIは最低版と最新版のGit、WindowsとLinuxのネイティブ環境で検証する。

サブモジュール、Git LFS、sparse-checkout、shallow clone、複数worktree、署名の独自制御は対象外。通常のGit設定／認証方式を用いる。hooks、fsmonitor、credential helper等による外部実行はGitの信頼境界として利用者へ説明する。

## 3. CLI仕様とGit呼出し

以下は許可する構文の全範囲。`PATH`はリポジトリ内のリテラルパス、`BRANCH`は管理対象名、`REV`は単一commitへ解決できるGit revision、`SOURCE`は§4.2の許可対象。略記`-a`／`-r`／`-m`／`-n`／`-c`／`-b`／`-v`はGitと同じ意味で用いる。長形式は表に記したものだけを許可する。

| サブコマンド | 許可する構文 | Git呼出し・動作 |
|---|---|---|
| init | `[-b main] [DIR]` | 事前検査後`git init -b main -- DIR`。`--initial-branch main`も可 |
| clone | `URL [DIR]` | `git clone --origin origin --no-checkout -- URL [DIR]`後、mainを確定してcheckout（§3.1） |
| status | 引数なし | 元cwdで`git -c core.quotepath=false -c status.relativePaths=true status`。upstream設定がある場合はahead/behindも表示。設定やrefを変更しない |
| diff | `[--cached] [HEAD] [-- PATH...]` | `git diff --no-ext-diff`に検証済み引数を付加。`--cached`と`HEAD`の併用は拒否 |
| diff | `--name-only --diff-filter=U [-- PATH...]` | `git diff --name-only --diff-filter=U -- {P}`。内部検査は`git ls-files -u -z` |
| add | `-- PATH...` | `git add -- {P}`。stageと競合解決済み登録。削除済みパスのstageもGitに従う |
| rm | `[-r] -- PATH...` | `git rm [-r] -- {P}`。作業ファイル維持の`--cached`や強制`-f`は提供しない |
| commit | `[-a] -m MESSAGE [-- PATH...]` | 通常は`git commit -m M`、`-a`なら`git commit -a -m M`、パス指定なら`git commit --only -m M -- {P}` |
| fetch | `[--tags] [origin]` | branch取得とtag取得を分離して安全に取得（§3.3）。remoteはorigin固定 |
| switch | `BRANCH` | ローカルがあれば`git switch --no-guess BRANCH`、originだけにあれば`git switch -c BRANCH --track origin/BRANCH` |
| switch | `-c NAME` | 作成規則検査後`git switch -c NAME`。`--create`も可。作成して切替 |
| branch | `[--list] [-v|-vv] [-r|-a]` | `git branch`の対応する一覧。`--remotes`／`--all`も可。`-r`と`-a`は排他 |
| branch | `NAME` | 作成規則検査後`git branch -- NAME`。現在のbranchは変えない |
| merge | `[--no-ff] [--no-commit] SOURCE` | sourceをcommit IDへ固定後`git -c merge.autoStash=false merge --no-autostash --no-ff --no-commit --no-edit SHA`。通常mergeはこのモード固定 |
| merge | `--ff-only origin/BRANCH` | 祖先関係検査後`git -c merge.autoStash=false merge --no-autostash --ff-only SHA` |
| merge | `--abort` | 警告を標準エラーへ出し`git merge --abort`。Gitと同じ構文で、`--yes`は不要 |
| push | `[origin] [BRANCH | REV:BRANCH]` | §5の単一送信手順。refspecの送信先は現在のbranchに限定 |
| log | `[-n N] [--graph] [--oneline] [--all]` | 対応する`git log`。`N`は1以上。`--max-count=N`も可 |
| show-ref | `--heads` | `git show-ref --heads`。全ローカルbranchのcommit IDとrefを表示 |
| tag | `NAME [REV]` | `git check-ref-format refs/tags/NAME`、REV解決後`git tag -- NAME SHA`。軽量タグ |
| tag | `[--list]` | `git tag --list`。引数なしも一覧 |

`commit -a`とパス指定は排他。`-m`／`--message`は必須、空文字は2。パス指定は通常commitだけで許可し、merge中は4。stageしていない新規ファイルはパス指定でも取り込まない。対象外のstage済み変更は指定ファイルcommitに含めず保持する。stage済み削除のパス指定を含む動作は実Git E2Eで検証する。merge確定は必ず全indexをcommitし、`-a`指定も4で拒否する。

`rm`はworktreeのファイルを削除するため、実行前に対象を表示する。変更が存在してGitが削除を拒否したら5。パス引数のある全操作は、まず全パスを構文／ルート検査してからGitを呼ぶ。Gitが途中失敗した場合のindex／worktree更新はロールバックせず、変更の可能性を案内して`status`確認を促す。

### 3.1 init・clone・初回設定

`init`はリポジトリ外から実行できる。対象自身が既存worktree／bare repositoryなら4、別worktree配下への入れ子作成も4。非空通常DIRは許可し既存ファイルを削除しない。通常ファイル、権限不足、作成失敗は11。競合生成後は再検査し、途中生成物を自動削除しない。

`clone --no-checkout`後、HEADが既にローカルmainを指しmain refが存在すればそのmainをcheckoutする。origin/mainが存在しローカルmainが無ければ`git switch -c main --track origin/main`を行う。空リモートは`git symbolic-ref HEAD refs/heads/main`でunborn mainを作る。空でなくmainが無い場合は12で終了し、生成済みDIRは残す。通常cloneが作るmainに再度`switch -c main`を実行する誤りを避ける。

サブコマンドを増やさないため、初回のユーザー情報とリモート登録は標準Gitによるセットアップとする。認証、サーバー作成も別工程。clone済みの場合はorigin登録不要。

```bash
tg init project
cd project
git config user.name "Developer"
git config user.email "developer@example.com"
git remote add origin <GitリモートURL>
tg add -- README.md
tg commit -m "Initial commit"
tg push origin main
```

origin未設定でfetch／push／remoteを必要とするmergeを呼んだ場合は12。ローカルcommit、branch、tagはoriginなしでも可能。

### 3.2 unborn HEAD・一覧表示

unborn mainではstage済み内容があれば初回commit可能、空indexなら10。`diff`と`diff --cached`はそのままGitを使う。`diff HEAD`だけはHEADが無いので、stage済み差分と未stage差分をそれぞれ見出し付きで順に表示する。各部分が空なら見出しも出さない。HEADを起点とする`log`は空表示0、`log --all`は既存refの履歴を表示する。`show-ref --heads`はrefが無ければ空表示0へ変換する。既存ref／tagがある場合はunbornでも一覧を省略しない。

`branch`はローカル管理対象外も表示する。remote一覧は管理対象とorigin/HEADのみを表示し、origin/HEADは疑似refと分かるGit表示に従う。管理対象外remoteを一覧から除くフィルタ以外はGit形式を維持する。`show-ref --heads`は管理対象外を含む全ローカルrefを表示する。upstreamがoriginの同名branchなら`-vv`でahead/behindを表示する。情報は最後にfetchした時点のもので、自動fetchしない。

### 3.3 fetchとタグ取得

1. `git fetch --no-tags origin`でbranch履歴を取得する。worktree／ローカルbranchは更新しない。
2. 通常fetchでは取得済みorigin branchから到達可能なcommitを指すタグを対象とする。remoteのtag／peeled IDを使い、対象commitがローカルに存在してorigin branchの祖先であることをplumbingで確認する。未取得objectを指すtagは通常fetchでは対象外。`--tags`指定時は全remote tagを対象とする。
3. `git ls-remote --tags origin`のタグref自身のobject IDをローカルと比較する。annotated tagのpeeled行`^{}`は比較対象外。同名異内容は9、ローカルタグを上書きしない。
4. 新規tagだけを明示refspecで`git fetch --no-tags origin refs/tags/T:refs/tags/T ...`により取得する。先頭`+`は禁止。途中競合は再度refを機械検査して9、分類不能な失敗は8。

fetchは複数段階のため、タグ競合で9になってもbranchのremote-tracking refは更新済みの場合がある。診断へ更新済み段階を表示する。fetchの再実行は同名同内容tagを変更せず、作業内容を破棄しない。明示tag refspecの大量送信はOS引数上限を考慮して分割する。

## 4. 状態とブランチ運用

### 4.1 状態と許可判定

状態を優先順でX（外部Gitのrebase／revert／cherry-pick／bisect／sequencer等）、MC（merge中・未解決）、M（merge中・解決済み）、H（detached／管理対象外）、U（管理対象unborn）、D（追跡済みworktreeまたはindex変更あり）、N（clean）と判定する。独立属性として現在のbranch名、HEAD有無、未追跡ファイルも保持する。単一状態だけで管理対象判定を省略してはならない。

| 操作 | N | D | M | MC | X | H | U |
|---|---|---|---|---|---|---|---|
| status／diff／log／show-ref／branch一覧／tag一覧 | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| add／rm | ○ | ○ | ○ | ○ | 4 | 4 | ○ |
| commit | 10 | ○※ | ○ | 7 | 4 | 4 | ○※ |
| branch NAME／switch -c NAME | ○ | ○ | 4 | 4 | 4 | 4 | 4 |
| tag作成 | ○ | ○ | 4 | 4 | 4 | 4 | 4 |
| fetch | ○ | ○ | ○ | ○ | 4 | ○ | ○ |
| switch既存branch | ○ | 5 | 4 | 4 | 4 | ○※※ | 4 |
| merge／merge --ff-only | ○ | 5 | 4 | 4 | 4 | 4 | 4 |
| merge --abort | 4 | 4 | ○ | ○ | 4 | 4 | 4 |
| push | ○ | ○ | 4 | 4 | 4 | 4 | 4 |

※ stage内容／選択内容が空なら10。Dで未stage変更だけがある場合、引数なしcommitは10だが`commit -a`は可能。※※ Hからは追跡変更が無い場合だけ管理対象branchへ戻れる。全ローカル書込みとpushは管理対象branchが必須。外部gitが作った管理対象外branch上のmergeもcommit／abortを4で拒否する。

進行中検出は`git rev-parse --git-path`でMERGE_HEAD、REVERT_HEAD、CHERRY_PICK_HEAD、rebase-merge、rebase-apply、sequencer、BISECT_STARTを解決して存在確認し、`git ls-files -u -z`と併用する。MERGE_HEADなしの未解決indexはX扱いとして書込みを拒否する。

未追跡ファイルはDに含めない。Gitが未追跡ファイルの上書きを拒否した場合は11で案内する。pushはDでもコミット済み履歴だけを送信し、未commit変更は送らない旨を警告する。

### 4.2 分岐・統合規則

| 役割 | 運用資料の名前 | Git／tgの名前 | 分岐元／統合方向 |
|---|---|---|---|
| リリース | default | main | devel→main |
| 開発 | devel | devel | mainから作成 |
| チケット | issXXX | iss[0-9]+ | develから作成、issNNN→devel |

名前は`^(main|devel|iss[0-9]+)$`に全体一致、数字は半角、先頭0可。iss1とiss0001は別branch。mainはinit／cloneで用意し、`branch main`／`switch -c main`で再作成しない。既存ローカルまたはorigin同名branchがあれば作成は4、不正名は2。作成元は現在のbranchとHEADに限定し、任意start-pointは許可しない。`branch NAME`は切り替えないので次に`switch NAME`が必要。変更を持ち越して作成・切替したい場合は`switch -c NAME`を使う。

通常mergeのSOURCEは、方向規則に合うローカルbranch（issNNN→devel、devel→main）、または同名の`origin/<現在のbranch>`のみ。revision一般やtagの直接mergeは提供しない。ローカルbranch統合は事前に`fetch --no-tags`相当のbranch取得を行い、originに統合元があればローカルと完全一致を要求する。不一致は6、リモート先行なら統合元へswitchしてff-only merge、ローカル先行ならpush、分岐なら同期mergeを案内する。originに無いローカル統合元は許可する。検査したSOURCEはcommit IDに固定してGitへ渡す。

`merge --ff-only origin/B`は現在のBと同名だけ許可する。両refが同一またはremoteがHEADの祖先なら通知して0、HEADがremoteの祖先ならfast-forward、分岐なら6。origin/Bが無い場合は4。通常mergeでsourceがHEAD祖先なら0、同名remoteが単純先行ならff-only更新を案内して4。異なるbranchはfast-forward可能でもmerge commitを準備する。

mainにはテスト済みdevelを統合し、必要な成果物を同じmerge commitまたは後続commitへ含める。ファイル内容が成果物かどうかはCLIでは保証できないため、直接修正の禁止と品質判定はレビュー／CIで担保する。

`tinygit.commitMessagePattern`は任意のGit設定キーとして継承する。GoのRE2でメッセージ文字列に対する一致を検査し、不一致は2、不正な式または設定取得失敗は12。全体一致が必要なら利用者が`^`と`$`を指定する。設定コマンドは公開せず`git config`で設定する。

## 5. pushの安全仕様

`tg push`の既定送信先はorigin、送信branchは現在のB。`tg push origin B`も同じ。選択送信は`tg push origin REV:B`とし、Mercurial式の`-r`や`--new-branch`を設けない。新規branchは通常pushで作成する。

1. 状態／branch／設定を検査する。originのURLは1つ、pushurlは未設定またはfetch URLと同一の1つだけ許可する。異なるpushurl、複数URL、mirror設定、tagを含む独自fetch refspecは12で拒否する。明示refspec、`--no-follow-tags`、`-c remote.origin.mirror=false`で暗黙の追加送信を防ぐ。
2. 送信元Sを固定する。既定はHEAD、選択時は`git rev-parse --verify --end-of-options REV^{commit}`で解決する。SがHEADの祖先でなければ4。不正REVは2。送信先はBだけ許可し、完全ref、空送信元、先頭`+`、複数refspecはユーザー入力として拒否する。
3. `git fetch --no-tags origin`でremote branchを取得し、`git ls-remote --heads origin refs/heads/B`で実在と先端を確認する。remoteが存在する場合、remote先端RがSの祖先でなければ6。Rのobjectが不足する場合は対象branchを追加fetchしてから判定する。remoteが無ければ新規branch作成へ進む。
4. Sから到達可能なcommitを指すローカルtagだけを送信候補にする。`git tag --merged S`と`ls-remote --tags origin`を比較し、同名異objectは9で送信しない。tag自体のobject IDで比較し、peeled行は無視する。Sより先のcommitを指すtagは送らない。
5. branch更新と新規tagを単一の`git -c remote.origin.mirror=false push --atomic --porcelain --no-follow-tags origin S:refs/heads/B refs/tags/T:refs/tags/T ...`で送る。更新不要のbranch／tagは省略する。全て不要なら0。refspec分割送信はせず、OS引数上限を超える場合は2で送信前拒否する。
6. 明確なnon-fast-forward／tag競合は9、一般の拒否／通信／認証失敗／atomic非対応は8。失敗時に非atomicへfallbackしない。
7. 成功時はbranch、送信commit ID、送信tag、送信不要refを表示する。選択pushでローカルBがSより先行していても、ローカルbranch／worktreeは移動しない。origin/Bは実際に送ったSへ更新されたことを確認する。upstreamは内部の`git branch --set-upstream-to=origin/B B`で設定する。remote成功後のローカル設定失敗は11とし、「remote送信は成功済み」と明記する。再実行してもbranch／tagを強制更新しない。

atomicはサーバーのref更新を全件成功か全件拒否にする保証であり、応答を受け取る前の通信断では「全件成功したが応答だけ失われた」可能性がある。通信断／中断後は成否不明と明記し、再接続時に`ls-remote`で各refを照合する。同一内容の再送は安全で、tagを強制移動しない。事前取得したremote先端とpushまでの並行更新はサーバーのnon-fast-forward判定へ委ねる。

## 6. 実行規約・Runner

- Gitを絶対パスで起動し、共通で`--no-pager`、`GIT_LITERAL_PATHSPECS=1`を設定する。機械解析は`LC_ALL=C`、NUL区切り出力を用いる。表示と解析を分離する。
- PATHは元cwd相対からルート相対へ変換し、ルート外は2。内部Gitはルートで実行し、表示用statusは元cwdで実行する。Windowsの予約名、CRLF、case、長いパス、symlinkはGitに委ねる。
- `GIT_EDITOR=:`、`GIT_MERGE_AUTOEDIT=no`を設定し、commitにはメッセージを要求する。ネットワーク操作以外は`GIT_TERMINAL_PROMPT=0`。認証だけstdinを接続し、出力はマスキングを通す。
- 継承した`GIT_DIR`、`GIT_WORK_TREE`、`GIT_INDEX_FILE`等のrepository転送環境は除去し、検査対象と操作対象の食い違いを防ぐ。`GIT_TRACE*`、`GIT_CURL_VERBOSE`を除去する。
- 禁止構文はサブコマンド別に検査する。`--force`、`--force-with-lease`、`+`refspec、削除refspec、`branch -d/-D/-m/-M/-f`、`tag -d/-f`、`switch -C/--discard-changes`、`commit --amend`、`reset --hard`は構築できない。メッセージやファイル名の文字列を誤って禁止フラグと判定しない。
- Ctrl+Cはcontextキャンセル、子プロセス終了、WaitDelay、必要時killで扱い130。index.lockは削除せず11を案内する。tg独自の永続ロックは作らずGitロックに従う。
- `merge --abort`はGit構文に合わせ確認オプションを要求しない。マージ中に追加した変更や解決内容を破棄し得るとhelpに明記する。autoStashは無効化し、開始前cleanを必須にする。

### 6.1 出力とマスキング

解析用stdoutは改変せず取得し、表示・ログ・デバッグへ渡す出口でマスクする。マスクでcommit IDやパスが変わったデータを機械解析してはならない。stderrは表示前にマスクする。URL userinfo、Authorizationヘッダ、AWS認証パラメータ、識別可能なGitHub tokenを対象とする。任意の未知tokenを完全検出する保証はしない。credential helperをtgが直接起動して秘密値を収集しない。

表示フィルタは1行を完結まで保留し、16KiBを超える未完結行は行全体を`[redacted: oversized line]`として出力し、次の改行まで破棄する。chunkをまたぐ秘密文字列と終端flushを検査する。機械可読の解析出力とは別経路にする。TTYへ直接書く認証補助プログラムの出力は保証範囲外としてマニュアルに記載する。環境値をログへ出さない。

表示出力は上限到達を明示できるが、status、refs、ancestor検査等の解析用出力は上限超過を11として失敗させ、切り詰めたデータで安全判定しない。結果と一覧はstdout、警告・診断はstderr。各失敗には次の操作を1行示す。

### 6.2 インターフェース

```go
type Options struct {
    Dir         string
    Env         map[string]string
    Stdin       io.Reader
    Stdout      io.Writer // 表示時はマスキングフィルタ経由
    Stderr      io.Writer // マスキングフィルタ経由
    MaxOutput   int64
    Interactive bool
}
type Result struct {
    ExitCode  int
    Stdout    []byte // 内部解析用rawデータ。直接ログ出力は禁止
    Stderr    []byte // マスク済み
    Truncated bool
}
type Runner interface {
    Run(context.Context, Options, ...string) (Result, error)
}
```

Gitが終了したら非0でもerrorはnil、起動不能／中断／タイムアウトはerror。共通安全環境の上書きはRunnerが拒否する。テストはフェイクRunnerと実Gitを使い、許可した構文だけが生成されることを検査する。

## 7. エラーコード

| コード | 分類 |
|---:|---|
| 0 | 成功、更新不要、空一覧 |
| 2 | 未知コマンド／オプション、不正名／REV／PATH、空メッセージ、引数上限 |
| 3 | リポジトリ外（init／clone／共通help／versionは除外） |
| 4 | 管理対象外、進行中操作、不適合な作成／統合／push方向 |
| 5 | clean必須違反、変更によるrm拒否 |
| 6 | 履歴分岐、送信元がremoteを含まない、統合元の鮮度不一致 |
| 7 | 未解決merge conflict |
| 8 | remote通信／認証／一般拒否／atomic非対応、push経路のhook失敗 |
| 9 | non-fast-forward、tag競合 |
| 10 | commit対象の変更なし |
| 11 | ローカルGit失敗、ローカルcommit hook失敗、lock、解析出力上限 |
| 12 | user.name/email、origin、URL／refspec、正規表現等の設定不備 |
| 13 | 予約。v5では認証失敗も8に分類 |
| 14 | Git未検出／最低版不足 |
| 130 | Ctrl+C |

stderr文面に依存せず、plumbing結果とpush porcelainで分類する。`rejected`だけで9と判断せず、non-fast-forwardやtag競合が明確な場合だけ9へ分類する。不明なremote失敗は8。タイムアウトはネットワークなら8、ローカル操作なら11とし、Ctrl+Cの130と分ける。

## 8. 運用シーケンス（各コマンドの成功を確認して次へ進む）

```bash
# 初回参加
tg clone <GitリモートURL> project
cd project
tg fetch origin
tg switch devel
tg merge --ff-only origin/devel

# チケット作業
tg switch -c iss109
# 編集・レビュー
tg add -- src/new_file.c src/changed_file.c
tg diff --cached
tg commit -m "【ソフト開発】FP02294G_ファーストフォーカス追加 #109 1.0h"
tg tag VER_5311_DEVEL
tg push origin iss109

# テスト合格後に開発branchへ統合
tg switch devel
tg fetch origin
tg merge --ff-only origin/devel
tg merge iss109
tg commit -m "【ソフト開発】Merge iss109"
tg push origin devel

# リリース（Jenkins成果物を配置し、新規／変更ファイルをstageする）
tg switch main
tg fetch origin
tg merge --ff-only origin/main
tg merge devel
tg add -- dist/app.exe dist/library.dll
tg commit -m "【ソフト開発】Release 5.3.11"
tg tag ORCA_5311_2018_0130_1614
tg push origin main
```

develが無い場合はmain上で`tg branch devel`、`tg switch devel`、`tg push origin devel`を行う。同じcommitを指す新規branchのpushも許可する。

```bash
# 同じbranchで並行commitし、pushが拒否された場合
tg fetch origin
tg branch -avv
tg show-ref --heads
tg merge origin/iss109
tg diff --name-only --diff-filter=U
# 競合内容を修正。保持するファイルはadd、削除するファイルはrm
tg add -- src/conflicted_file.c
tg commit -m "【ソフト開発】Merge origin/iss109"
tg push origin iss109

# 中止する場合（解決内容等を破棄し得る）
# tg merge --abort

# 選択したcommitまでの送信。現在iss109上で実行
tg push origin <commit-ID>:iss109
```

ファイル指定commitは`tg commit -m "..." -- src/a.c src/b.c`。merge確定ではファイルを指定しない。複数チケットで同じ開発バージョン名のtagを再利用しないようチームで命名を調整する。テスト合格の判定、Redmine工数／チケット更新、CI成果物の取得は別工程。

## 9. ディレクトリ・ビルド・テスト

構成は`cmd/tg`、`internal/{cli,ops,policy,state,git}`、`tests/`、`docs/`、`.github/workflows/`とする。外部依存なしのCLI、CGOなしの配布バイナリを基本とする。

```bash
go test -race ./...
go vet ./...
govulncheck ./...
go test -tags=e2e ./...
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath -o dist/linux/tg ./cmd/tg
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build -trimpath -o dist/windows/tg.exe ./cmd/tg
```

| 分野 | 必須検証 |
|---|---|
| 公開範囲 | 15サブコマンド／17機能の対応、未知コマンド2、旧名alias拒否、共通help／version |
| 初期化 | init外部実行、既存repository／入れ子4、非空DIR、main既定clone、main以外が既定のclone、main欠落12、空clone初回commit／push |
| Git構文 | branch作成だけでは切替しない、switch -c、Gitの-m/-a/-n/-rの意味、禁止フラグがメッセージ／パスにある場合の誤判定防止 |
| commit | stage済みのみ、-a、パス限定、追加／変更／削除、他のstage内容保持、空10、unborn、mergeでパス／-a拒否、user未設定12 |
| 削除 | rmでworktreeとindexから削除、-r、変更あり5、--cached／-f拒否、失敗後status |
| branch | 作成元規則、方向規則、既存名4、管理対象外、dirtyからswitch -c、originのみへのswitch、一覧／ahead・behind／show-ref |
| merge | ff-only同一／先行／分岐6、origin refなし4、統合元remote先行／ローカル先行／分岐6、未公開統合元、競合7、diff U→add／rm→commit、abort |
| 並行開発 | 2cloneで同じbranchへcommit、fetchで分岐確認、同期mergeしてpush、異なるチケットを順にdevelへ統合 |
| fetch | worktree不変、remote refs更新、到達可能tag／--tags、同名異object9、peeled行、tag失敗後もbranch取得済みの案内 |
| push | 新規branch、選択commit、非祖先4、remoteより古い送信6、先のtag除外、branch＋複数tagの1ref拒否時に全送信ref不変、atomic非対応8 |
| 障害 | push応答前の通信断で成否不明表示、全ref照合と再送、fetch各段階失敗、lock11、commit hook11、push hook8、中断130 |
| 設定 | push.default／followTags／mirror／pushurl／複数URL／独自fetch refspec、merge.autoStash、editor／pager、正規表現、Git転送環境 |
| パス／OS | cwd相対、ルート外2、空白／日本語／改行／先頭-／literal pathspec、CRLF、case rename、長いパス、symlink |
| 出力 | raw解析とマスク表示の分離、1バイトchunk、終端flush、16KiB超行、解析上限11、UTF-8 |
| 品質 | Git最低版／最新版、両OSネイティブCI、race、vet、govulncheck、status／パス／CLI fuzz、多数ファイル・巨大履歴 |

リリース基準は全上記E2Eの両OS成功、静的検査成功、CodeCommit／GitHubの実接続試験（`remote`タグ）とatomic対応確認。未確認リモートのatomic保証を製品説明で断定しない。

## 10. 保証範囲と残作業

履歴の追記とrefの通常更新を原則とし、強制push／履歴改変の入口は公開しない。サーバー管理者や通常Gitによる操作は制限しない。Git branchの削除禁止はtgのAPI制限であって永続named branch保証ではない。mainへの成果物以外の変更禁止も内容検査では保証しない。

今回の更新対象は設計書。Go実装、CI、実接続検証、利用者マニュアルは未完了。実装時に17機能の対応表とテスト計画を受入基準にする。v4.2のbackout／doctor／id等を互換機能として残さない。

## 11. 参照

- 運用資料: `/home/subcon/AI_WORKING/旧ソフトウェア開発プロセス_ver1.04.doc`
- [Git commit（stageとファイル指定）](https://git-scm.com/docs/git-commit)
- [Git switch（切替と作成）](https://git-scm.com/docs/git-switch)
- [Git branch（作成と一覧）](https://git-scm.com/docs/git-branch)
- [Git diff（比較対象と未解決一覧）](https://git-scm.com/docs/git-diff)
- [Git push（refspecとatomic）](https://git-scm.com/docs/git-push)
- [Git show-ref（branch ref先端）](https://git-scm.com/docs/git-show-ref)
