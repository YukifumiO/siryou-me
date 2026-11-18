# Bitcoin定量分析・AI研究 段階別作業索引

本ディレクトリは、統合計画書 `bitcoin_quant_ai_research_summary.md` の開発ロードマップを、実行単位ごとに分割したものである。

開発言語、データベース、ワークフロー、実験管理、API、監視などの共通開発環境は、[`development_environment_framework.md`](../development_environment_framework.md) を参照する。

## 進行順序

| Phase | 内容 | 主な完了条件 |
|---|---|---|
| [Phase 1](phase_01_data_source_research/README.md) | データ取得元の調査 | 第一候補と代替ソースが決定 |
| [Phase 2](phase_02_data_acquisition_poc/README.md) | データ取得PoC | 7種類のデータを手動取得・保存可能 |
| [Phase 3](phase_03_automated_data_platform/README.md) | 自動収集基盤 | 毎日自動的にデータが蓄積 |
| [Phase 4](phase_04_visualization_statistics/README.md) | 可視化・統計分析 | 市場環境とデータ品質を1画面で確認 |
| [Phase 5](phase_05_machine_learning_baseline/README.md) | 機械学習ベースライン | 再現可能なHMM・GMM・XGBoost検証 |
| [Phase 6](phase_06_deep_learning_comparison/README.md) | 深層学習比較 | 基準モデルに対する追加価値を判定 |
| [Phase 7](phase_07_paper_trading/README.md) | ペーパートレード | 実時間とバックテストの差を説明 |
| [Phase 8](phase_08_small_live_trading/README.md) | 少額実売買 | 実約定コスト込みで評価可能 |
| [Phase 9](phase_09_automation_monitoring/README.md) | 自動化・監視 | 取得から停止監視まで一貫稼働 |

## 共通ルール

- Phaseは原則として番号順に進める。
- 各Phaseの完了判定は、利益や主観ではなく記載済みの完了条件で行う。
- 実験条件の変更は上書きせず、新しいバージョンとして保存する。
- 実売買はPhase 8まで行わない。
- 長期保有資産と研究資金を常に分離する。
- APIキーはリポジトリに保存しない。
- すべての時刻、特徴量、ラベルについて未来情報の混入を検査する。
- 予測精度だけでなく、取引コスト控除後の経済的有効性を評価する。

## Phase完了時の記録

各Phaseの完了時に次を残す。

- 完了日
- 実行担当者
- 使用したデータ・コード・設定のバージョン
- 成果物の保存場所
- 完了条件の判定結果
- 既知の問題と残存リスク
- 次Phaseへの引き継ぎ事項

