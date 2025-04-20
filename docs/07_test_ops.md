# テスト・運用・トラブルシューティング

## テスト設計と実装例
- Karpenter本体・Providerともに単体テスト・統合テスト・E2Eテストを重視
- [karpenter/test/](../../karpenter/test/) 配下にテストスイートやテストユーティリティを実装
- 主要なテスト例:
  - [test/pkg/](../../karpenter/test/pkg/) : 単体テスト
  - [test/suites/](../../karpenter/test/suites/) : 統合・E2Eテスト
- Provider固有のテストも [karpenter-provider-aws/test/](../../karpenter-provider-aws/test/) に実装

## テスト自動化・CI
- PR作成時や定期的にGitHub Actions等で自動テストを実行
- テスト結果はPRコメントやダッシュボードで可視化
- [designs/integration-testing.md](../../karpenter/designs/integration-testing.md)

## 運用時の監視・メトリクス
- [pkg/metrics/](../../karpenter/pkg/metrics/) でPrometheus等向けのメトリクスを提供
- 主要な監視項目例:
  - プロビジョニング成功/失敗数
  - ノード数・スケールイベント
  - エラー発生回数

## トラブルシューティング
- ログ出力やStatusフィールドで異常検知・原因特定を支援
- 代表的な障害例と対処法:
  - ノードが起動しない → NodeClaim/NodeClassのSpecやProvider設定を確認
  - スケールしすぎる/しない → スケジューリング要件やDisruptionポリシーを見直し
- [公式トラブルシューティングガイド](https://karpenter.sh/docs/troubleshooting/)
