# Bitcoin定量分析・AI自動売買 開発環境・フレームワーク設計書

## 1. 目的

本文書は、Bitcoin定量分析・AIパターン発見・バックテスト・ペーパートレード・自動売貲を段階的に開発するための、推奨開発環境と技術スタックを定義する。

単一の「全部入りフレームワーク」に依存せず、データ収集、時系列処理、研究、モデル学習、バックテスト、API、可視化、監視の各役割に適したツールを組み合わせる。

---

## 2. 結論

本プロジェクトの推奨構成は次のとおりである。

> **Pythonモジュラーモノリス + uv + Docker Compose + Polars / DuckDB + PostgreSQL / TimescaleDB + Prefect + MLflow + FastAPI + Streamlit**

初期からKubernetes、Kafka、Spark、多数のマイクロサービスは導入しない。一方、Notebookだけで全処理を実装せず、確定した処理は再利用可能なPythonパッケージとして管理する。

---

## 3. 設計原則

### 3.1 モジュラーモノリス

ソースコードは1つのリポジトリで管理し、データ取得、特徴量、モデル、バックテスト、リスク管理、発注などの責務をモジュールで分離する。

利点：

- 研究と実運用で同じロジックを使える
- 初期のデプロイ・監視が簡単
- サービス間通信による障害箇所を増やさない
- 必要になったモジュールだけを後からサービス分割できる

### 3.2 研究・バックテスト・実運用のロジック共有

次のロジックは、Notebook、バックテスト、ペーパートレード、実売買で共通化する。

- データ標準化
- 特徴量計算
- 市場レジーム判定
- モデル前処理と推論
- シグナル生成
- リスク判定
- 注文意図の作成

差し替えるのは実行部だけとする。

```text
BacktestExecution
PaperExecution
LiveExchangeExecution
```

### 3.3 再現性

- Pythonバージョンを固定する
- `uv.lock`をGitで管理する
- データスナップショットを保存する
- コードのGit commitを実験に記録する
- 特徴量、ラベル、モデル設定をバージョン管理する
- ランダムシードと実行環境を記録する

### 3.4 安全側の失敗

データの古さ、欠損、モデルエラー、口座照合不一致、注文状態不明の場合は、発注せず停止する。

---

## 4. 推奨技術スタック

| 領域 | 推奨技術 | 用途 |
|---|---|---|
| 言語 | Python 3.12 | 収集、分析、ML、API、自動売買の統一 |
| 依存関係 | uv | 環境作成、パッケージ、ロックファイル管理 |
| コンテナ | Docker Compose | 開発・CI・運用環境の統一 |
| HTTP | HTTPX | REST API取得、非同期I/O |
| ストリーム | websocketsまたは取引所SDK | WebSocket市場データ |
| データ加工 | Polars | 時系列ETL、特徴量作成 |
| 研究用SQL | DuckDB | Parquetの直接分析 |
| 履歴保存 | Parquet | Raw、整形済み、特徴量データ |
| 運用DB | PostgreSQL | 注文、約定、残高、リスク、監査ログ |
| 時系列DB拡張 | TimescaleDB | 大量の時系列と連続集計 |
| ワークフロー | Prefect | 取得、品質検査、学習、レポートの定期実行 |
| API | FastAPI + Pydantic | 内部API、推論、ダッシュボード接続 |
| 可視化 | Streamlit + Plotly | 初期ダッシュボード |
| 統計 | NumPy, SciPy, statsmodels | 基礎統計、検定、時系列分析 |
| 機械学習 | scikit-learn | 前処理、GMM、評価、パイプライン |
| 勾配ブースティング | XGBoost, LightGBM | 将来リターン・方向予測 |
| 深層学習 | PyTorch | LSTM、GRU、Transformer系 |
| 時系列DL | PyTorch Forecasting | TFTと分位点予測 |
| 説明可能性 | SHAP | 特徴量寄与と個別予測の説明 |
| パラメータ探索 | Optuna | 制限付きハイパーパラメータ探索 |
| 実験管理 | MLflow | データ、パラメータ、指標、モデルの紐付け |
| テスト | pytest, Hypothesis | 単体、統合、性質ベーステスト |
| 静的品質 | Ruff, mypy | フォーマット、Lint、型チェック |
| 開発フック | pre-commit | コミット前チェック |
| CI | GitHub Actions | 自動テスト、Lint、ビルド |
| メトリクス | Prometheus | ジョブ、API、注文、リスク監視 |
| 監視画面 | Grafana | 運用メトリクスとアラート |

---

## 5. システム構成

```text
外部市場API
  ↓
HTTPX / WebSocketコネクタ
  ↓
Prefectワークフロー
  ↓
Raw JSON / CSV保存
  ↓
Parquetデータレイク
  ↓
Polars / DuckDBによる品質検査・加工
  ↓
PostgreSQL / TimescaleDB
  ↓
特徴量生成
  ├─ HMM / GMM / Pólya tree
  ├─ XGBoost / LightGBM
  └─ PyTorch / TFT
  ↓
独自バックテスト・リスクエンジン
  ↓
FastAPI
  ├─ Streamlitダッシュボード
  ├─ ペーパートレード
  └─ 取引所アダプタ
  ↓
Prometheus / Grafana / アラート
```

---

## 6. Python環境

### 6.1 Pythonバージョン

初期推奨はPython 3.12とする。統計・機械学習・深層学習パッケージの互換性を優先し、上位バージョンへの移行はCIで互換性を確認してから行う。

### 6.2 uv

`pyproject.toml`でパッケージを宣言し、`uv.lock`で解決済みの正確なバージョンを固定する。

原則：

- `uv.lock`はGitで管理する
- 開発・CI・デプロイで同じロックを使う
- CIでは`uv run --locked`または`--frozen`を使う
- 依存関係の更新は個別パッケージごとに実施する
- 機械学習と深層学習の依存グループを分ける

依存グループ例：

```text
core        設定、共通型、ログ
data        API取得、Polars、DuckDB
research    Jupyter、統計分析
ml          scikit-learn、XGBoost、SHAP
deep        PyTorch、TFT
api         FastAPI
dashboard   Streamlit、Plotly
dev         pytest、Ruff、mypy
```

---

## 7. データフレームとストレージ

### 7.1 Polars

主なETLと特徴量計算にPolars Lazy APIを使用する。

採用理由：

- クエリプランの自動最適化
- ストリーミング実行
- メモリを超えるデータへの対応
- 実行前のスキーマエラー検出
- Parquetとの高い親和性

学習ライブラリがNumPyやpandasを必要とする場合は、最終入力の直前で変換する。

### 7.2 Parquetデータレイク

```text
data/
├── raw/          # APIの未加工応答
├── bronze/       # 型と時刻を標準化
├── silver/       # 欠損・重複・品質フラグ処理済み
├── gold/         # 特徴量・学習用データ
└── snapshots/    # 実験で使用した固定版
```

各層を上書きで変化させず、生成ジョブ、ソース、生成日時を記録する。

### 7.3 DuckDB

DuckDBでParquetを直接SQL検索する。研究段階で大量のデータをサーバーDBへコピーせず、列選択・期間抽出・集計を行う。

### 7.4 PostgreSQL / TimescaleDB

Phase 7以降の運用状態はPostgreSQLで管理する。

- シグナル
- 注文意図
- 取引所注文
- 約定
- ポジション
- 口座残高
- リスク状態
- 自動停止状態
- 監査ログ

高頻度の時系列、保持ポリシー、連続集計が必要になった段階でTimescaleDBを追加する。

---

## 8. ワークフロー管理

### 8.1 Prefectの対象

- 市場データ取得
- リトライ
- 欠損バックフィル
- データ品質レポート
- 特徴量バッチ生成
- 定期学習
- ウォークフォワード実験
- 日次・週次レポート

### 8.2 Prefectに任せない処理

実注文の送信を、一般的なタスク再試行に任せない。発注は独立した実行コンポーネントから行い、次を使って二重発注を防ぐ。

- クライアント注文ID
- 冪等性キー
- 取引所注文ID
- 発注後照会
- 状態不明時の追加発注禁止

---

## 9. APIとダッシュボード

### 9.1 FastAPI

FastAPIは以下の内部APIに使用する。

- 最新市場データ
- 特徴量
- 市場レジーム
- モデル推論
- シグナル
- リスク状態
- ペーパートレード状態
- 注文・約定・ポジション
- 緊急停止

Pydanticで入出力と設定を検証し、OpenAPIで仕様を自動生成する。

### 9.2 Streamlit

初期ダッシュボードはStreamlitで構築する。

表示対象：

- BTC、DXY、Gold、VIX、DVOL、米10年債利回り、BTC ETF Flow
- 相関・先行相関
- 市場レジームと状態確率
- 予測値と予測確率
- SHAP寄与
- バックテスト結果
- データ品質
- ペーパー・実取引状態
- ドローダウンと停止状態

売買ロジックはStreamlitに記述せず、共通ドメイン層またはFastAPIを呼び出す。

---

## 10. 実験管理

### 10.1 MLflowで記録する項目

- 実験IDと実行時刻
- Git commit
- データスナップショットID
- 特徴量バージョン
- ラベル定義
- 学習・検証・テスト期間
- Gap、purging、embargo
- ハイパーパラメータ
- ランダムシード
- 統計的評価指標
- 取引コスト控除後指標
- 学習済みモデル
- SHAPグラフ、混同行列、バックテスト結果

### 10.2 開始時の構成

個人・小規模開発ではローカルMLflowから開始する。チーム共有が必要になったら、メタデータをPostgreSQL、成果物をS3互換オブジェクトストレージへ移行する。

---

## 11. バックテストフレームワーク

### 11.1 独自コアの採用

既製バックテストライブラリだけに依存せず、本プロジェクトの要件に合わせた小さなイベント駆動コアを実装する。

理由：

- ETF Flowやマクロデータの公表時刻を反映する必要がある
- Funding、OI、DVOLなど多種類のデータを扱う
- Maker / Taker手数料とスプレッドを扱う
- スリッページ、遅延、部分約定を再現する
- 日本の取引所固有仕様に対応する
- 自動停止条件をバックテストにも適用する

### 11.2 共通インターフェース

```python
signal = strategy.generate_signal(market_state)
decision = risk_manager.evaluate(signal, portfolio)
result = execution.submit(decision.order_intent)
```

主なドメインオブジェクト：

- `MarketEvent`
- `FeatureSnapshot`
- `RegimeState`
- `Prediction`
- `Signal`
- `RiskDecision`
- `OrderIntent`
- `Order`
- `Fill`
- `Position`
- `PortfolioSnapshot`

---

## 12. 推奨ディレクトリ構成

```text
bitcoin-quant/
├── pyproject.toml
├── uv.lock
├── compose.yaml
├── .env.example
├── README.md
├── configs/
│   ├── data_sources/
│   ├── models/
│   ├── strategies/
│   └── risk/
├── data/
│   ├── raw/
│   ├── bronze/
│   ├── silver/
│   ├── gold/
│   └── snapshots/
├── src/bitcoin_quant/
│   ├── domain/
│   │   ├── market.py
│   │   ├── signal.py
│   │   ├── order.py
│   │   └── portfolio.py
│   ├── connectors/
│   │   ├── market_data/
│   │   └── exchanges/
│   ├── ingestion/
│   ├── quality/
│   ├── features/
│   ├── regimes/
│   │   ├── hmm.py
│   │   ├── gmm.py
│   │   └── polya_tree.py
│   ├── models/
│   ├── backtest/
│   ├── strategies/
│   ├── risk/
│   ├── execution/
│   ├── monitoring/
│   ├── api/
│   └── dashboard/
├── notebooks/
│   └── exploratory/
├── workflows/
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── data_quality/
│   └── regression/
├── migrations/
├── artifacts/
└── docs/
```

### Notebookのルール

- Notebookは探索、グラフ、仮説確認に限定する
- 確定した取得、特徴量、学習、評価処理は`src/`へ移す
- Notebook内で実売買APIを呼び出さない
- 実験に使用したデータと設定を明記する

---

## 13. コンテナ構成

### 13.1 初期Docker Compose

```text
app             Python実行環境
postgres        運用DB
mlflow          実験管理UI
prefect-server  ワークフロー管理
dashboard       Streamlit
```

Phase 1～2は`app`だけで開始してもよい。必要になったサービスをPhaseごとに追加する。

### 13.2 ボリューム

- PostgreSQLデータ
- MLflow成果物
- Prefectメタデータ
- Parquetデータレイク
- ログ

データをコンテナの一時レイヤーだけに保存しない。

---

## 14. テスト戦略

### 14.1 単体テスト

- データ変換
- 時刻・タイムゾーン
- 特徴量計算
- ラベル作成
- 手数料・スリッページ
- リスク上限
- ポジション・損益

### 14.2 性質ベーステスト

Hypothesisで次の不変条件を確認する。

- OHLCの整合性
- 売買前後の資産整合性
- 手数料を増やして純損益が改善しないこと
- 同じ冪等性キーで二重注文が作られないこと
- リスク上限超過時に発注されないこと

### 14.3 統合テスト

- API応答サンプルからDB保存まで
- 特徴量からモデル推論まで
- シグナルから仮想約定まで
- 取引所サンドボックスまたはモックへの発注
- 発注後の照会・照合

### 14.4 回帰テスト

- 固定データセットで特徴量が変化しない
- 固定モデルで予測値が予期せず変化しない
- 固定バックテストの取引数・損益が予期せず変化しない

---

## 15. コード品質とCI

### 15.1 ローカルチェック

```text
Ruff format
Ruff check
mypy
pytest
```

pre-commitで軽量なチェックを実行し、重いバックテストや深層学習は別ジョブに分離する。

### 15.2 CI

Pull Requestごとに以下を実行する。

- `uv.lock`の整合性
- フォーマット・Lint
- 型チェック
- 単体・統合テスト
- DBマイグレーションテスト
- Dockerイメージビルド
- 秘密情報混入の検査
- 必要に応じて小規模な回帰バックテスト

---

## 16. 秘密情報とセキュリティ

- `.env`はローカル開発だけに使用し、Gitへ追加しない
- `.env.example`にはキー名だけを記載する
- 本番はSecret Manager等から注入する
- データ取得用と売買用のAPIキーを分ける
- 売買用キーに出金権限を付与しない
- IPアドレス制限を有効にする
- ログにAPIキー、シグネチャ、シークレットを出力しない
- 権限とキーを定期的に監査・ローテーションする
- キー漏洩時の無効化手順を文書化する

---

## 17. 監視

### 17.1 開発初期

当初はJSON形式の構造化ログ、ジョブ状態、データ鮮度、エラー通知から開始する。

### 17.2 ペーパー・実運用

Prometheus / Grafanaで以下を監視する。

- API成功率と遅延
- データの鮮度、欠損、異常値
- 特徴量・予測値のドリフト
- シグナル数
- 注文・約定・キャンセル
- スプレッドとスリッページ
- ポジションと口座残高
- 損益、ドローダウン、停止状態

---

## 18. Phaseごとの導入

### Phase 1～2：取得元調査・PoC

```text
Python 3.12
uv
Polars
DuckDB
Parquet
HTTPX
pytest
JupyterLab
```

この時点でPostgreSQL、MLflow Server、Prometheus等をすべて必須にしない。

### Phase 3～5：自動収集・統計・機械学習

```text
Prefect
PostgreSQL
MLflow
scikit-learn
statsmodels
XGBoost / LightGBM
SHAP
Optuna
```

### Phase 6：深層学習

```text
PyTorch
PyTorch Forecasting
TFT
必要に応じてPatchTST / iTransformer
```

GPUはモデルとデータ量に必要な場合だけ追加する。

### Phase 7～9：ペーパー・実売買・監視

```text
FastAPI
Streamlit
TimescaleDB（必要に応じて）
取引所WebSocket
Prometheus
Grafana
アラート通知
```

---

## 19. 初期に導入しないもの

### Kubernetes

初期規模ではオーケストレーションの運用負荷が上回る。Docker Composeで容量、可用性、デプロイ頋度の限界が実際に発生してから検討する。

### Kafka

イベント量や消費者数が少ない段階では過剰である。WebSocket、PostgreSQL、プロセス内キューで不足する場合に再検討する。

### Spark

PolarsとDuckDBで処理できない規模に到達してから検討する。

### マイクロサービス分割

チーム、デプロイ、スケーリング、障害分離の要求が明確になったモジュールだけを分離する。

### 独自Reactダッシュボード

初期はStreamlitで研究と運用に必要な画面を確定し、アクセス制御、複数ユーザー、高度なUIが必要になった後で置き換える。

### MT5中心設計

MT5は指標表示、監視、アラートの補助には使えるが、データ基盤、AI学習、実験管理、日本の取引所API接続の中心にはしない。

---

## 20. 開発開始時に作成するファイル

```text
pyproject.toml
uv.lock
.python-version
.gitignore
.env.example
compose.yaml
README.md
Makefileまたは共通コマンドスクリプト
src/bitcoin_quant/__init__.py
tests/
configs/
data/
notebooks/
```

READMEには次を記載する。

- 必要ソフトウェア
- 初回セットアップ
- 環境変数
- テスト実行方法
- データ取得方法
- ダッシュボード起動方法
- バックテスト方法
- 秘密情報の禁止事項

---

## 21. フレームワークの再評価条件

次のいずれかが発生した場合に、技術構成を再評価する。

- 1台でデータ処理が期限内に完了しない
- WebSocketイベントを消費する独立システムが増えた
- 複数チームが個別にデプロイする必要が生じた
- 高可用性や無停止更新が必須になった
- Streamlitでユーザー・権限要件を満たせない
- PostgreSQL / TimescaleDBで性能要件を満たせない
- GPU学習の共有・スケジューリングが必要になった

---

## 22. 最終方針

開発環境の成功条件は、最新の技術を多数導入することではない。

次を満たすことを成功条件とする。

1. 新しい開発者が同じ環境を再現できる
2. 同じデータと設定から同じバックテスト結果を得られる
3. 研究・バックテスト・実運用でロジックを共有できる
4. データとモデルのバージョンを追跡できる
5. 未来データ混入をテストで防止できる
6. 発注・約定・リスク判定を監査できる
7. 異常時に安全側で停止できる
8. 運用上の限界が生じるまで、構成を必要以上に複雑化しない

---

## 23. 参考資料

- [uv: Working on projects](https://docs.astral.sh/uv/guides/projects/)  
  `pyproject.toml`、プロジェクト環境、クロスプラットフォームの`uv.lock`による再現性の公式資料。

- [Polars: Lazy API](https://docs.pola.rs/user-guide/lazy/using/)  
  クエリ最適化、ストリーミング、実行前のスキーマチェックに関する公式資料。

- [DuckDB: Reading and Writing Parquet Files](https://duckdb.org/docs/stable/data/parquet/overview)  
  Parquetの直接読み書きとフィルター・列プッシュダウンに関する公式資料。

- [FastAPI](https://fastapi.tiangolo.com/)  
  Pydanticによるデータ検証、OpenAPIドキュメント、非同期APIに関する公式資料。

- [MLflow Tracking](https://mlflow.org/docs/latest/ml/tracking)  
  実験、パラメータ、指標、成果物、データセット、モデル追跡に関する公式資料。

