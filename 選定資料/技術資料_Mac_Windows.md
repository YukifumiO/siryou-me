# 生成AI CLI比較・導入と安全なWebアプリ自動生成基盤 技術選定書（macOS / Windows / WSL2）

- 対象：生成AI CLIを導入・比較し、安全なWebアプリ自動生成基盤を検討する開発者
- 更新日：2026年11月18日
- 対象環境：macOS、Windows 11（PowerShell）、Windows 11 + WSL2（Ubuntu）
- 目的：CLIの能力を比較し、Web開発技術を選定したうえで、Mac/Windowsへの導入とAI自動生成基盤の開発方針をまとめる

> **注意**：プラン料金・利用枠・インストール方法は変更される可能性があります。導入時には各社の公式ドキュメントも確認してください。月額料金は目安（米ドル、税別）です。AIサービスの月額プランと、APIキーを使った従量課金は別物です。

## 目次・構成

- **第I部（1〜10章）**：Claude Code / Codex CLI / Gemini CLIの契約と導入（Mac・Windows・WSL2）
- **第II部（11〜18章）**：主要6 CLIの性能比較（新規開発・解析・一括修正・レビュー）、評価方法と役割分担
- **第III部（19〜30章）**：安全なWebアプリ生成の技術比較（Laravel・Rust/Axum・Go・ASP.NET・Django、Vue・React・Svelte・Livewire）、DSL、検証、ロードマップ

---

# 第I部：生成AI CLI導入手順

## 1. 導入対象と料金

| CLI | アカウント | 想定する利用プラン | 料金目安 | コマンド |
|---|---|---|---:|---|
| Claude Code | Anthropic / Claude | Claude Pro | $20/月 | `claude` |
| Codex CLI | OpenAI / ChatGPT | ChatGPT Plus | $20/月 | `codex` |
| Gemini CLI | Google | 個人向け無料枠 | $0（無料枠内） | `gemini` |

3つを上記の構成で使用すると、合計 **$40/月**（税・追加料金を除く）です。既存契約があれば重複契約は不要です。Codexの無料枠・利用条件は時期によって異なるため、契約前に利用可能か確認してください。各プランには利用回数やレート制限があります。

公式サイト：
- Claude: https://claude.ai/ / Claude Code: https://code.claude.com/docs/en/setup
- ChatGPT: https://chatgpt.com/ / Codex CLI: https://github.com/openai/codex
- Gemini CLI: https://github.com/google-gemini/gemini-cli

## 2. 共通：アカウントを作成・契約する

### 2.1 Claude Code

1. https://claude.ai/ を開き、アカウントを登録してログインする。
2. プランのアップグレード画面から **Claude Pro**（月額の目安 $20）を契約する。
3. 契約済みアカウントをClaude Codeのブラウザ認証に使用する。
4. APIキーの登録は不要。`ANTHROPIC_API_KEY`を使うと別のAPI課金経路になる場合があるため注意する。

### 2.2 Codex CLI

1. https://chatgpt.com/ を開き、アカウントを登録してログインする。
2. **ChatGPT Plus**（月額の目安 $20）を利用する場合はアップグレードする。すでに契約済みなら再契約しない。
3. Codex CLIでは **Sign in with ChatGPT** で同じアカウントにログインする。
4. OpenAI APIキーを使う認証は、ChatGPT Plusとは別課金なので今回の手順では利用しない。

### 2.3 Gemini CLI

1. 既存の個人用Googleアカウントを用意する。なければ https://accounts.google.com/signup で作成する。
2. Gemini CLIの初回起動時に **Sign in with Google** を選ぶ。
3. 個人用無料枠では原則としてAPIキーや有料契約は不要。組織アカウントでは追加設定が必要な場合がある。

## 3. macOSにインストールする

### 3.1 環境確認

ターミナルを起動し、以下を確認する。

```bash
sw_vers
git --version
node -v
npm -v
brew --version
```

Git、Node.js、npmが入っていれば再導入は不要。Node.jsがなく、Homebrewが導入済みの場合：

```bash
brew install node
node -v
npm -v
```

Homebrew公式：https://brew.sh/ja/

### 3.2 Claude Code

```bash
curl -fsSL https://claude.ai/install.sh | bash
claude --version
claude
```

初回起動時にClaudeアカウントを使ってブラウザでログインする。CLI内でログインし直す場合は `/login` を利用する。

`claude: command not found` の場合はターミナルを再起動し、必要に応じてPATHを確認する。ネイティブインストールで `~/.local/bin` が利用されている場合：

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
claude --version
```

### 3.3 Codex CLI

```bash
npm install -g @openai/codex
codex --version
codex login
```

ブラウザでChatGPTアカウントによる認証を行う。起動コマンド：

```bash
codex
```

### 3.4 Gemini CLI

```bash
npm install -g @google/gemini-cli
gemini --version
gemini
```

初回画面で「Sign in with Google」を選択し、Googleアカウントで認証する。

## 4. Windows 11（PowerShell）にインストールする

### 4.1 事前準備

PowerShellまたはWindows Terminalを起動し、確認する。

```powershell
git --version
node --version
npm --version
```

未導入なら、`winget`が使用できる環境で次を実行する。

```powershell
winget install --id Git.Git -e
winget install --id OpenJS.NodeJS.LTS -e
```

インストール後はPowerShellを開き直してバージョンを再確認する。GUIでの導入先：
- Git: https://git-scm.com/downloads/win
- Node.js: https://nodejs.org/en/download

### 4.2 Claude Code

```powershell
irm https://claude.ai/install.ps1 | iex
claude --version
claude
```

Claude Proのアカウントでブラウザ認証する。Windows向けの実行要件・Git Bash関連の注意は公式セットアップガイドも参照する。

### 4.3 Codex CLI

Windows用の公式インストール方法は更新される場合がある。以下は前回案内したPowerShellのインストール方法：

```powershell
irm https://chatgpt.com/codex/install.ps1 | iex
codex --version
codex
```

スクリプトの利用可否・最新版の手順は https://github.com/openai/codex で確認する。代替として、Node.jsが導入済みなら次のnpm方式も利用できる。

```powershell
npm.cmd install -g @openai/codex
codex --version
codex
```

初回起動時に **Sign in with ChatGPT** を選び、ChatGPTアカウントでログインする。2つのインストール方式を重複して実行する必要はない。

### 4.4 Gemini CLI

```powershell
npm.cmd install -g @google/gemini-cli
gemini --version
gemini
```

Googleアカウントで認証する。`npm.ps1 cannot be loaded`と表示される場合、上記のように`npm.cmd`を使うとPowerShellの実行ポリシーを変更せずに実行できる。

## 5. Windows 11 + WSL2（Ubuntu）にインストールする

WSL2はLinuxサーバー向け開発やLaravel、Dockerを扱う場合に便利。Windowsネイティブとは**別の実行環境**なので、各CLIをUbuntu内にもインストール・認証する。

### 5.1 WSL2とUbuntuの導入

管理者権限のPowerShellを開く。

```powershell
wsl --install -d Ubuntu
```

必要に応じてWindowsを再起動し、Ubuntuを起動。初回起動時にLinuxユーザー名とパスワードを設定する。

Ubuntuのターミナルで：

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y git curl ca-certificates
```

Node.jsは`nvm`や`fnm`などで対応するLTS版を導入する。導入後：

```bash
node -v
npm -v
git --version
```

### 5.2 Claude Code（Ubuntu内）

```bash
curl -fsSL https://claude.ai/install.sh | bash
claude --version
claude
```

### 5.3 Codex CLI（Ubuntu内）

```bash
npm install -g @openai/codex
codex --version
codex
```

ChatGPTアカウントで認証する。CLIの起動後、画面の案内に従う。

### 5.4 Gemini CLI（Ubuntu内）

```bash
npm install -g @google/gemini-cli
gemini --version
gemini
```

Googleアカウントで認証する。

### 5.5 WSL2の推奨作業場所

ソースコードはLinuxのホームディレクトリ内に置く。

```bash
mkdir -p ~/development/ai-app
cd ~/development/ai-app
git init
```

Linux側で動くツールのソースを`/mnt/c/`配下ではなくLinuxファイルシステムに置くと、ファイル操作・性能・権限の問題を避けやすい。

## 6. 動作確認（各CLI共通）

**重要：** 初回は別々のテストディレクトリで実行し、AIのファイル作成要求を確認してから許可する。

### macOS・WSL2（Bash / zsh）

```bash
mkdir -p ~/ai-test/claude-test ~/ai-test/codex-test ~/ai-test/gemini-test
cd ~/ai-test/claude-test
claude
```

CLI画面で「このフォルダにREADME.mdを作り、『Claude Codeの動作確認』と記載してください」と入力。終了後：

```bash
cat README.md
```

同じ方法で`~/ai-test/codex-test`に移動して`codex`、`~/ai-test/gemini-test`に移動して`gemini`を起動する。

### Windows（PowerShell）

```powershell
New-Item -ItemType Directory -Force "$HOME\ai-test\claude-test"
New-Item -ItemType Directory -Force "$HOME\ai-test\codex-test"
New-Item -ItemType Directory -Force "$HOME\ai-test\gemini-test"
Set-Location "$HOME\ai-test\claude-test"
claude
```

README.mdを生成するよう指示し、終了後：

```powershell
Get-Content .\README.md
```

同様にCodexとGeminiのフォルダに移動し、各CLIを起動して確認する。

### バージョンの一括確認

```text
claude --version
codex --version
gemini --version
```

上記はmacOS/WSL2のシェルでも、Windows PowerShellでも同じように実行できる。

## 7. よくあるトラブル

| 症状 | 対処 |
|---|---|
| `command not found` / `not recognized` | 新しいターミナルを開き、PATHとインストール先を確認 |
| `node`や`npm`がない | Node.jsの対応版を導入 |
| PowerShellで`npm.ps1 cannot be loaded` | `npm.cmd`を使う |
| npmの`EACCES`・アクセス拒否 | 権限とNode.jsのインストール方法を確認。安易に`sudo npm -g`しない |
| ブラウザ認証が失敗 | 使用アカウント、ブラウザ、プロキシ・ファイアウォールを確認し再ログイン |
| CLIが契約プランを認識しない | 月額プランの契約アカウントとCLIの認証アカウントが同じか確認 |
| API利用料金が発生する | APIキー認証や環境変数を使っていないか確認 |
| WSL2でCLIが見つからない | Windows側とは別にUbuntu内へインストールする |

## 8. 3つを使った開発に進む場合の推奨構成

まずは**順番に使用**し、同時に同じファイルを編集させない。

1. Claude Code：要件分析・設計
2. Codex CLI：コード実装・テスト
3. Gemini CLI：独立レビュー・仕様確認
4. 人間：差分確認・承認・Gitコミット

例：

```text
ai-app/
├── docs/
│   ├── requirements.md
│   ├── architecture.md
│   └── security.md
├── CLAUDE.md
├── AGENTS.md
├── GEMINI.md
└── src/
```

※各CLIの指示ファイルの読み込み仕様は異なる。共通ルールは`docs/`に置き、各CLIに明示して共有する。並列実行する場合はGit worktreeで作業ディレクトリを分離する。

### セキュリティ上の注意

- 初回は独立したテストディレクトリで実行する。
- AIのシェルコマンド実行とファイル変更は、内容を確認してから承認する。
- 本番DB、クラウド秘密鍵、顧客データをCLIに無制限に渡さない。
- 会社のポリシー・契約上のデータ取り扱い条件を確認する。
- 本番デプロイや破壊的変更は人間の承認を必須にする。

## 9. 導入チェックリスト

### 共通
- [ ] 利用するOS・ターミナルを決めた
- [ ] Gitを確認した
- [ ] npm方式を使う場合、Node.js/npmを確認した

### Claude Code
- [ ] Claudeアカウントを用意した
- [ ] 必要に応じてClaude Proを契約した
- [ ] `claude --version`が成功した
- [ ] Claudeアカウントでログインした
- [ ] README.mdの生成に成功した

### Codex CLI
- [ ] ChatGPTアカウントを用意した
- [ ] 必要に応じてChatGPT Plusを契約した
- [ ] `codex --version`が成功した
- [ ] ChatGPTアカウントでログインした
- [ ] README.mdの生成に成功した

### Gemini CLI
- [ ] Googleアカウントを用意した
- [ ] `gemini --version`が成功した
- [ ] Googleアカウントでログインした
- [ ] README.mdの生成に成功した

## 10. 導入順序のおすすめ

**Gemini CLI → Codex CLI → Claude Code** の順が試しやすい。まず無料利用枠でCLIの操作に慣れ、必要に応じて有料プランを契約する。

---

### 公式情報（確認用リンク）

- Claude Code Setup: https://code.claude.com/docs/en/setup
- Claude Plans: https://claude.ai/
- Codex CLI GitHub: https://github.com/openai/codex
- ChatGPT: https://chatgpt.com/
- Gemini CLI GitHub: https://github.com/google-gemini/gemini-cli
- Git for Windows: https://git-scm.com/downloads/win
- Node.js: https://nodejs.org/en/download
- Microsoft WSL: https://learn.microsoft.com/windows/wsl/install

---

# 第II部：生成AI CLIの比較・選定

> **評価の読み方**：以下の「◎・○・△」は、公開されている機能・設計思想・開発運用への適合性から判断した**定性的な評価**であり、同じ課題を実行して得たベンチマークの実測順位ではない。AIモデル、推論設定、権限、ツールバージョン、対象プロジェクトによって結果は変わる。特にOpenCodeとAiderは接続モデルの影響が大きい。

## 11. 比較するCLIの範囲

| ツール | 種別 | 主な特徴 | 導入判断 |
|---|---|---|---|
| **Claude Code** | Anthropic公式AI開発エージェント | リポジトリ解析、複数ファイル編集、コマンド・テスト実行、設計・実装 | 設計・大きな機能の実装を試す |
| **Codex CLI** | OpenAI公式AI開発エージェント | ローカルコード編集、コマンド実行、テスト、差分レビュー、非対話実行 | 実装・自動テスト・レビューを試す |
| **Gemini CLI** | Google公式AI開発エージェント | ファイルの検索・読取、広範なコード解析、編集、Googleログインの無料枠 | 調査・独立レビューを試す |
| **GitHub Copilot CLI** | GitHub中心のAI開発CLI | GitHubとの連携、調査・レビュー等のエージェント機能 | GitHub中心の運用を重視する場合 |
| **OpenCode** | OSSのマルチモデルエージェント | AIモデルやエージェントの使い分け、Plan/Build等のワークフロー | 複数モデルの統一操作をしたい場合 |
| **Aider** | OSSのGit連携型ペアプログラミングツール | Repository Map、コード編集、Git、lint・テスト連携 | 既存リポジトリの段階的な修正 |

**注意**：OpenCodeやAiderはそれ自体が独自の高性能モデルではない。Claudeなどの同一モデルを接続しても、コンテキストの選び方・ツール実行・編集方法によって公式CLIと結果が異なり得る。

## 12. 作業別の概要比較

| 作業 | Claude Code | Codex CLI | Gemini CLI | Copilot CLI | OpenCode | Aider |
|---|---|---|---|---|---|---|
| 新規Webアプリの設計・実装 | ◎ | ◎ | ○ | ○〜◎ | モデル依存 | モデル依存 |
| 既存コードの構造解析 | ◎ | ◎ | ◎ | ◎ | モデル依存 | ○ |
| 複数ファイルの一括修正 | ◎ | ◎ | ○〜◎ | ○〜◎ | モデル依存 | ○〜◎ |
| コードレビュー | ◎ | ◎ | ○〜◎ | ◎ | モデル依存 | ○ |
| Git・差分管理 | ◎ | ◎ | ○ | ◎ | ◎ | ◎ |
| モデルを柔軟に切り替える | 限定的 | 限定的 | 限定的 | 設定次第 | ◎ | ◎ |
| 初期費用を抑えた試用 | △ | ○ | ◎ | △ | 利用モデル次第 | 利用モデル次第 |

**推奨**：最初はClaude Code・Codex CLI・Gemini CLIの3つを、同じ評価課題で試す。GitHub中心のチームならCopilot CLI、モデル切り替えを統一したいならOpenCode、Git連携の細かな編集を重視するならAiderを追加候補にする。

## 13. 新規アプリ開発性能の詳細

新規アプリの評価は「コードを書けるか」だけでは不十分。仕様理解→DB・API・画面設計→実装→テスト→実行の一貫性を測る。

| 評価軸 | チェック内容 | 主な失敗例 |
|---|---|---|
| 要件理解 | 必須機能・例外条件・権限を漏らさないか | 認可条件・入力制限の欠落 |
| アーキテクチャ | Controller/Service/Repository/画面の役割が適切か | 業務ロジックが画面に混在 |
| DB・API・UI整合 | スキーマ、API、フロントの型が一致するか | `customer_id` と `customerId` の食い違い |
| 規約遵守 | Laravel/Rust等の慣例と既存規約に従うか | 例外処理や認証方式の独自実装 |
| テスト生成 | 正常・異常・権限・境界値を確認するか | 正常系だけ通過 |
| 自律修復 | ビルド/テストエラーを解析して修正できるか | 無関係なファイルまで変更 |
| 完成度 | 起動でき、実際の利用操作が通るか | 見た目のみ完成・保存できない |

- **Claude Code**：仕様から複数ファイルにわたる機能の設計・実装をまとめて進める候補。広範な変更や独自判断をさせる際は、計画と変更範囲の承認を挟む。
- **Codex CLI**：実装→コマンド実行→テスト→修正の反復を任せる候補。テストを通しただけでは要件適合を保証しない。
- **Gemini CLI**：仕様整理、コードとドキュメントの照合、別視点での実装案に活用する。長い入力が処理できることと高い正解率は同義でない。
- **Copilot CLI**：GitHub中心のIssue、PR、コード調査との連携を重視する際に検討する。
- **OpenCode**：選択したモデルとPlan/Buildの設定で結果が大きく変わる。
- **Aider**：範囲を限定した対話型編集に適する。アプリ全体の自律開発は個別検証が必要。

**測定案**：ログイン・顧客CRUD・一覧検索・ロール認可・監査ログを含む同一仕様を渡し、「受け入れテスト通過率」「ビルド成功率」「回帰バグ」「所要時間」「費用」を複数回測定する。

## 14. 既存コード解析性能の詳細

| 能力 | 何ができれば合格か | 確認方法 |
|---|---|---|
| 構造把握 | 主要モジュールと責務を説明できる | 架空の説明でなく実ファイルを引用 |
| 依存追跡 | 画面→API→業務処理→DBを辿れる | 関連関数と呼び出し箇所を列挙 |
| 障害原因特定 | 再現手順と根本原因を分離できる | 故意に入れた不具合を調査 |
| 影響分析 | 変更対象と変更してはいけない箇所を特定 | 修正前後のテスト比較 |
| 根拠の提示 | ファイル・行・ログに基づいて説明 | 不存在の関数や行番号を指摘していないか |
| 調査効率 | 不要な全文読取や無関係な探索が少ない | ツール呼び出し数・時間・費用 |

具体例：React/Vue→Nginx→Laravel→MySQL/PostgreSQLのログインで401が発生したとき、APIリクエスト、Cookie、Nginx、ミドルウェア、Policy、ログを辿って原因を判定する。

- **Claude Code/Codex CLI**：検索とツール実行、テスト・ログ照合を含めた調査に適する。
- **Gemini CLI**：`glob`、ファイル読取、検索などを用いた大量コードの調査に適する。関連性を絞ることが重要。
- **Copilot CLI**：GitHub上の履歴・PRと連動するワークフローが適する。
- **OpenCode**：Explore等で読み取り専用調査と編集を分離できる。
- **Aider**：Repository Mapによるファイル・シンボルの構造要約が特徴。

**測定案**：既知の原因を持つ障害を10件以上用意し、根本原因の正解率、関連ファイルの再現率、誤指摘率、根拠の正確性を比較する。

## 15. 複数ファイルの一括修正性能の詳細

例：APIの`user_name`を`userName`に変更する際、DB・API・型・Vue/Reactコンポーネント・テスト・外部互換性の全てを点検する。

| 能力 | 成功条件 | 典型的な失敗 |
|---|---|---|
| 対象探索 | 修正が必要な場所を発見 | 呼び出し側の直し忘れ |
| 変更完全性 | 全必須箇所を正しく修正 | 特定画面だけ旧名称が残る |
| 最小差分 | 不要な変更をしない | 無関係な命名・整形・設計変更 |
| 既存互換 | 外部利用者・移行時の互換性を維持 | API破壊・DB移行失敗 |
| テスト復旧 | 型・ビルド・テストが正常 | 型だけ通るが実行時に破綻 |
| 復元性 | Gitで差分と戻し方を管理 | 生成変更の上書きや混合 |

**評価指標**：変更完全率、過剰変更率、回帰テスト通過率、未修正箇所数、最終差分の品質。単純な変更ファイル数は性能を示さない。

**運用策**：別々のGit worktreeでAIごとに作業する。API仕様を破壊する場合は明示的な承認を必須にする。AiderはGit連携・lint/テスト反復が特徴。Codex、Claude、Gemini、OpenCodeでもテストを必須にして比較する。

## 16. コードレビュー性能の詳細

| レビュー分類 | 具体例 | 必要な検証 |
|---|---|---|
| セキュリティ | SQLインジェクション、XSS、CSRF | 攻撃条件・対策コードの再確認 |
| 認証・認可 | 別テナントの顧客を読める | 403/404やテナント境界のテスト |
| ロジック | 条件分岐、境界値、NULL | 単体テスト・性質ベーステスト |
| パフォーマンス | N+1、無駄な通信 | クエリ数・負荷テスト |
| 競合 | 二重更新、排他漏れ | 並行操作の再現テスト |
| 設計・保守 | 重複、循環依存、責務逸脱 | 実際の変更コストとの照合 |
| 回帰 | 修正前に動いていた機能が壊れる | 既存テスト・E2E |

**最重要の指標**：
- **Recall（再現率）**＝正しく発見したバグ数／実際のバグ数
- **Precision（適合率）**＝正しいバグ指摘数／全指摘数
- 重大バグの検出率、誤指摘率、修正案の有効率、レビュー時間・費用

Claude Code・Codex CLI・Copilot CLIはコードレビュー候補。Gemini CLIを独立した視点で追加するのもよい。OpenCodeのレビュー力は接続モデルに依存する。どのAIの指摘も**正しいことを自動的に保証しない**。

## 17. 公開ベンチマークの読み方

| 名称 | 主に評価する内容 | 主な限界 |
|---|---|---|
| SWE-bench Verified / Pro | 既存リポジトリのIssue修正 | 新規アプリ設計全体、長期運用を直接示さない |
| SWE-bench Multilingual | 多言語の既存コード修正 | フレームワークごとの安全な生成率とは異なる |
| Aider Polyglot / editing系 | 複数言語の編集課題 | CLI全体・最新モデルの総合性能ではない |
| 自前の受入テスト | 自社の技術・仕様に対する実務成功率 | 他社への一般化は限定的 |

**注意**：モデル単体ベンチマークの順位を、Claude Code / Codex CLI / Gemini CLI等のCLI製品順位に置き換えない。同一時期、同じリポジトリ・モデルバージョン・実行条件で比較する。

## 18. 推奨するCLIの役割分担と運用

```text
人間：要件・権限・最終承認
  ↓
Claude Code：要件分析・設計案・DSL案
  ↓
Gemini CLI：仕様漏れ・認可漏れの独立レビュー
  ↓
安全なコード生成器：テンプレートから決定的に生成
  ↓
Codex CLI：限定的な業務処理実装・テスト・修正案
  ↓
静的解析・依存監査・統合テスト・セキュリティテスト
  ↓
人間の承認 → Gitマージ → デプロイ
```

3つのCLIを同じワーキングツリーに同時編集させない。モデル評価時はClaudeとCodexの役割を入れ替え、「担当が有利だから良かった」という偏りも検証する。

**開発方式は3種類**：①役割分担型（最初に推奨）、②並列競争型（同じタスクを別worktreeで生成し比較）、③自動オーケストレーション型（成熟後にCLIの非対話実行とジョブ管理を統合）。

---

# 第III部：安全なWebアプリ自動生成基盤の技術選定

## 19. 方針：安全なフレームワークとは何か

「脆弱性が一切ない言語・フレームワーク」は存在しない。PHPランタイム、Webフレームワーク、Composer/npm/Cargoの依存ライブラリ、DB・OS・インフラ、そして業務アプリ固有の認可・ロジックを分けて管理する。

**目的**：脆弱性の発生可能性を減らすことに加え、発見後に**影響するアプリを特定し、更新・再テスト・再生成できること**を基盤の中核機能とする。

| 脆弱性の場所 | 例 | 共通基盤による対処 |
|---|---|---|
| 言語・ランタイム | PHP、Rust依存先、Goランタイム | サポート期限管理、パッチ適用 |
| フレームワーク・依存 | Laravel、Axum周辺、npm | SBOM、依存監査、更新試験 |
| アプリ実装 | SQLi、XSS、認可漏れ | 安全な生成器、lint、テスト |
| 設定・インフラ | Cookie設定、公開DB、秘密情報 | 安全な初期設定、最小権限、監査 |
| 業務仕様 | テナント越境、二重処理 | DSLでポリシー宣言、受入テスト |

## 20. バックエンド技術の比較

| 技術 | セキュリティ・型の特性 | AI生成・アプリ開発の適性 | 主な注意点 | 推奨場面 |
|---|---|---|---|---|
| **Laravel + PHP** | 認証、Policy、FormRequest、CSRF、Eloquent等が整備 | CRUD・管理画面の生成に強い | PHP/Composerの更新、認可漏れに注意 | 業務アプリ自動生成の第一候補 |
| **Rust + Axum** | Safe Rustのメモリ安全、強い型制約 | 安全な型・DSL・生成器と相性がよい | 認証・管理UIの統合部品は自前選定、初期開発負担 | 安全性の制約を厳密化した基盤、API |
| **Go + net/http** | メモリ安全、標準HTTPの構成がシンプル | 少数依存の共通基盤を作りやすい | 標準の認証・管理機能が不足しがち | コンパクトなAPI・共通基盤 |
| **ASP.NET Core + C#** | 型、認証・認可・データ保護等の公式機能 | 統合的な業務アプリ開発に強い | .NETの更新とエコシステム選定 | 企業向け基幹・業務システム |
| **Django + Python** | 認証、ORM、CSRF、管理画面が整備 | 管理・CRUDの迅速な実装 | Pythonと依存・動的型に注意 | 管理画面・運用ツール |
| **NestJS + TypeScript** | 型付けされたアプリ構造 | フロントとTypeScriptを共有しやすい | Node/npmの依存・ランタイム管理 | TypeScript統一志向 |

「Rustだから認可漏れがない」「Laravelだから必ず危険」とは言えない。RustでもSQLi、XSS、CSRF、認可やロジックの欠陥は起こり得る。

## 21. Rust + Axum案：AIで安全なコードを生成する基盤

### 21.1 長所と弱点

**長所**：所有権・借用・型チェックによるエラー検出、Safe Rustでのメモリ安全性、`#![forbid(unsafe_code)]`で自分のクレート内のunsafe利用を禁止できること、Axumの型付きリクエスト処理。

**弱点**：Safe Rustでも業務権限の漏れは防げない。依存クレート内部の`unsafe`を一律に禁止できるわけではない。開発初期に、認証・セッション・UIの選定と共通実装が必要。

### 21.2 推奨構成

| 分野 | 候補 | 注意 |
|---|---|---|
| Web API | Axum + Tokio + Tower | HTTP/ミドルウェア基盤として利用 |
| DB | PostgreSQL + SQLx | 動的SQLまで自動で安全になるわけではない |
| シリアライズ | Serde | 入力制約・業務妥当性は別途検証 |
| 認証 | 信頼できるOIDC IdP | 暗号・認証プロトコルを自作しない |
| セキュリティ検査 | `cargo audit`、`cargo deny`、Clippy | 既知の脆弱性・規約チェックが中心 |
| UI | サーバー側HTML、Vue、React等 | UI方式によって安全な描画規則が変わる |

### 21.3 生成するコードの安全性を高める方法

1. AIにコードを全面的に自由生成させず、画面・データ・権限を表す**DSL**を生成させる。
2. DSLをJSON Schema等と独自のルールで検証し、既定は`deny`とする。
3. 認証・認可・DB接続・監査処理は**レビュー済みテンプレート**から決定的に生成する。
4. AIが追加する自由な業務コードはインターフェースと権限を制限する。
5. コンパイル・単体・結合・認可・脆弱性テストを通す。
6. 生成器・テンプレート本体の変更は人間が承認する。

## 22. Laravel + Vue案：AIによる業務アプリ自動生成

### 22.1 特徴

- Laravelの**Model / Controller / FormRequest / Policy / Migration / Test**という標準的な構造をテンプレート化しやすい。
- Vue 3 + TypeScript + Inertiaで、Laravelのルーティング・認証を使いながら高機能な画面を生成できる。
- DB・API・UIまで生成する初期速度はRust + Axumより有利になりやすい（設計上の見込みであり実測値ではない）。
- PHPランタイム、Laravel、Composer、Node.js、npmの**更新と依存監査が継続的に必要**。

### 22.2 共通ルール

| リスク | 安全な標準実装・生成ルール |
|---|---|
| SQLインジェクション | Eloquent/Query Builderのパラメータ化を優先し、危険な文字列連結SQLを禁止 |
| 入力検証 | FormRequest等を全書き込みAPIに適用 |
| 認可漏れ | Policy/Gateを必須化、レコード単位とテナント境界を検証 |
| XSS | Vueのエスケープ、`v-html`制限、CSP |
| CSRF | 認証方式に応じたCSRF対策を維持 |
| マスアサインメント | 更新可能属性を明示 |
| Cookie/セッション | Secure / HttpOnly / SameSite、セッション失効と監査 |
| 既知の脆弱性 | `composer audit`、`npm audit`、更新・回帰テスト |

**とくに重要**：認証済み（ログインしている）と認可済み（その顧客情報を閲覧してよい）は違う。生成ルートが認可処理を必ず通る設計と、他テナントデータにアクセスできないテストが必要。

## 23. フロントエンド比較：Vueは最適か

| 技術 | 強み | AI自動生成での適性 | セキュリティ・保守の注意 | 向くアプリ |
|---|---|---|---|---|
| **Vue 3 + TypeScript** | コンポーネント構造が明瞭、Laravel/Inertiaと連携 | ◎：標準化したCRUD部品を生成しやすい | `v-html`、npm依存、状態管理 | 一般的な業務アプリ、EC、管理画面 |
| **React + TypeScript** | 高機能UIと大きなライブラリエコシステム | ◎：複雑なUIに対応 | `dangerouslySetInnerHTML`、採用ライブラリの統一 | 高機能SaaS、複雑なダッシュボード |
| **Svelte** | 比較的簡潔なコンポーネント、コンパイル方式 | ○〜◎：コード量削減の可能性 | `{@html}`、チーム・部品選定 | 軽量なWeb UI |
| **Livewire + Blade** | Laravel/PHP中心で状態と画面を記述 | ◎：CRUD中心の管理画面に適合 | 公開プロパティ改ざん・毎操作の認可確認 | 社内業務、標準的な管理画面 |

**注意**：HTML描画時のエスケープはどの方式でも重要。JavaScriptを減らすことだけで脆弱性がなくなるわけではない。Vue/React/SvelteでのTypeScript静的検査が全ての実行時バグを検出するわけでもない。

### 23.1 選定の考え方

- **汎用的な業務Webアプリを多数生成**：Laravel + Vue 3 + TypeScript + Inertiaを第一候補。
- **検索・一覧・登録・編集など管理画面が中心**：Laravel + Livewire + Bladeも同条件で比較する。
- **非常に複雑なUI、豊富なUI部品**：React + TypeScriptを候補にする。
- **軽量で比較的単純なUI**：Svelteを検討する。

安全性の順位をフレームワーク名だけで断定せず、「生成コードの規約遵守率」「依存数」「回帰テスト」「脆弱性・認可の検査結果」を評価する。

## 24. 推奨する最初のシステム構成

```text
自然言語の要件
     |
Claude Code / Codex CLI / Gemini CLI
  （要件整理・仕様生成・独立レビュー）
     |
共通DSL（バージョン付き）
  - entities / fields / validation
  - screens / workflows
  - roles / permissions / record policies
  - API / events / tests
     |
DSLバリデータ（不正/不足なら生成停止）
     |
決定的コードジェネレーター
     |----------------------------|
 Laravel + Vue             将来: Rust + Axum
 (最初の実装)              (必要なら別生成器)
     |
自動検査：型・静的解析・テスト・依存監査・E2E
     |
Git PR・人間の承認
     |
リリース・継続的な脆弱性監視
```

**設計方針**：生成器は安全な共通処理を含むレビュー済みテンプレートを使用し、AIは主にDSL作成・仕様レビュー・限定的な業務ロジック実装を担う。生成器本体をAIが無承認で更新することは禁止する。

## 25. DSL（アプリ仕様）の例

```yaml
app:
  name: customer-management
  spec_version: 1

entities:
  - name: Customer
    fields:
      - { name: id, type: uuid, required: true }
      - { name: tenant_id, type: uuid, required: true }
      - { name: name, type: string, required: true, max_length: 100 }
      - { name: email, type: email, required: true }

permissions:
  default: deny
  roles:
    admin:
      Customer: [create, read, update, delete]
    staff:
      Customer: [read, update]
  resource_scope:
    Customer: same_tenant_only

ui:
  - resource: Customer
    screens: [list, detail, create, edit]
```

このDSLは**設計例**であり、そのまま動作する既製の仕様ではない。`same_tenant_only`の意味はコード生成器で厳密に実装・検証する。staffの`read`権限を持っていても、他テナントの顧客データは読めないようにする。

## 26. セキュリティゲート

| 工程 | 代表的な検査 | 失敗したら |
|---|---|---|
| DSL検証 | 定義不足、権限不明、未許可操作 | コード生成停止 |
| コード生成 | 生成ファイルの再現性・テンプレート署名 | 差分のレビュー |
| 型・lint | PHPStan、`vue-tsc`、Clippy等 | 修正後に再実行 |
| 単体/結合 | 正常・異常・境界・権限 | マージ禁止 |
| E2E | ログイン・CRUD・検索・ロール操作 | マージ禁止 |
| 依存監査 | Composer/npm/Cargoの既知脆弱性 | 影響・更新を評価 |
| セキュリティ | 認可越境、CSRF/XSS、秘密情報 | リリース停止・修正 |
| デプロイ | ステージングでの動作・ロールバック | 承認前に本番適用禁止 |

Laravel側の例（適切なツール導入後）：

```bash
composer audit
npm audit
vendor/bin/phpstan analyse
php artisan test
npx vue-tsc --noEmit
npm run build
```

Rust側の例（各ツール導入後）：

```bash
cargo check
cargo clippy --all-targets --all-features -- -D warnings
cargo test
cargo audit
cargo deny check
```

※これらで**未知の脆弱性、業務仕様の誤り、全ての認可漏れ**を自動検出できるわけではない。AIレビューとは独立したテストと必要に応じた専門的セキュリティレビューを行う。

## 27. 段階的な開発ロードマップ

1. **PoC-1：手動の基準アプリ** — Laravel + Vue + Inertia + PostgreSQLで、ログイン、顧客CRUD、ロール・レコード認可、監査ログを実装する。
2. **PoC-2：比較アプリ** — 同じ仕様をLivewire版でも構築し、生成コード量・テスト・脆弱性・保守性を比較する。
3. **PoC-3：DSLと生成器** — Model、FormRequest、Policy、Controller、Migration、Vue UI、テストを生成する。生成物の再現性を確認する。
4. **PoC-4：AI CLI連携** — Claude/Codex/Geminiで要件→DSL→レビュー→限定実装を構築。権限と作業領域を分離する。
5. **PoC-5：セキュリティゲート** — 認可越境、静的解析、脆弱性検査、E2E、CI/CD、Git承認を必須化する。
6. **運用段階：更新管理** — ランタイム・フレームワーク・依存パッケージのサポート期限、SBOM、影響アプリ一覧、更新・再生成・回帰テストを管理する。
7. **将来の展開** — DSLが安定したら、Rust + Axum等の別バックエンド生成器を比較検討する。

## 28. 比較検証に使うベンチマーク仕様案

**同一の顧客管理アプリ要件**と**既知の不具合セット**を使用し、CLI・技術構成を比較する。

| 項目 | 提案比重 | 実測すること |
|---|---:|---|
| 新規アプリ開発 | 30% | 受入テスト成功率、仕様充足率、ビルド・E2E |
| 既存コード解析 | 25% | 根本原因特定率、証拠の正確性 |
| 複数ファイル修正 | 25% | 変更完全率、最小差分、回帰率 |
| コードレビュー | 20% | Recall、Precision、重大欠陥検出率 |

- 各CLIで同じベースコミット、同じ要件、同じテスト、同じ権限を使う。
- モデル名・バージョン・推論設定・CLI版・開始時刻・トークン/利用枠・費用を記録する。
- 同一課題を複数回実行する。非決定的な結果や試行間のぶれを見る。
- 開発者によるヒント・修正、追加プロンプト回数も記録する。
- 評価スコアの根拠となる差分、ログ、テスト結果を保管する。
- OpenCode・Aiderは、まず接続モデルを固定してCLI差を測る。

## 29. 最終的な選定方針

**短期の推奨**：
- CLI：**Claude Code + Codex CLI + Gemini CLI**。最初は順次使用し、その後にGit worktreeを使って並列化する。
- バックエンド：**Laravel**。認証・認可・DB・バリデーション等の既存機能を活用し、業務Webアプリ生成のPoCを早く作る。
- フロントエンド：**Vue 3 + TypeScript + Inertia**を第一候補。CRUD中心なら**Livewire + Blade**も比較する。
- DB：**PostgreSQL**。
- 設計：**AI生成DSL + 安全なテンプレートによる決定的コード生成 + 自動検査 + 人間の承認**。

**中長期の選択肢**：メモリ安全性・強い型制約・独立API基盤に重点を置くなら、Rust + AxumをDSLの第2生成先として評価する。Go + net/http、ASP.NET Coreも用途に応じて候補に残す。

**最重要**：「最初に脆弱性を減らす」だけでなく、脆弱性発見後の**影響検知→修正→再生成→再テスト→承認→配布**まで設計する。

## 30. 比較・技術選定の参考リンク

- Claude Code: https://code.claude.com/docs/en/overview
- Codex CLI: https://github.com/openai/codex
- Gemini CLI: https://github.com/google-gemini/gemini-cli
- GitHub Copilot CLI: https://docs.github.com/en/copilot
- OpenCode: https://opencode.ai/docs/
- Aider / Repository Map: https://aider.chat/docs/repomap.html
- SWE-bench: https://www.swebench.com/
- Laravel: https://laravel.com/docs
- Inertia: https://inertiajs.com/
- Vue Security: https://vuejs.org/guide/best-practices/security
- React: https://react.dev/
- Svelte: https://svelte.dev/
- Livewire: https://livewire.laravel.com/docs
- Rust: https://www.rust-lang.org/
- Axum: https://docs.rs/axum/latest/axum/
- SQLx: https://github.com/launchbadge/sqlx
- OWASP Top 10: https://owasp.org/www-project-top-ten/
- RustSec: https://rustsec.org/
