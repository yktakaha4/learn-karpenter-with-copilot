# Learn Karpenter with Copilot

このディレクトリには、Karpenterの主要な機能や設計について学ぶためのドキュメントが含まれています。それぞれのファイルには、特定のトピックに関する詳細な説明が記載されています。

## ドキュメント一覧

### [consolidation.md](./consolidation.md)
- **説明**: KarpenterのConsolidation機能について解説しています。
  - Consolidationの種類（Single Node, Multi-Node, Spot-to-Spot）
  - 各Consolidationのロジックと処理フロー
  - `computeConsolidation`や`computeSpotToSpotConsolidation`メソッドの詳細

### [singlenodeconsolidation.md](./singlenodeconsolidation.md)
- **説明**: Single Node Consolidationの`ComputeCommand`メソッドについて解説しています。
  - 単一ノードの統合を実行するためのロジック
  - 入力に対する出力のサンプル

### [multinodeconsolidation.md](./multinodeconsolidation.md)
- **説明**: Multi-Node Consolidationの`ComputeCommand`メソッドについて解説しています。
  - 複数ノードの統合を効率的に実行するためのロジック
  - 最大バッチサイズの設定やバイナリサーチの使用方法

### [docs.md](./docs.md)
- **説明**: Karpenterのスケジューリングやメトリクスに関する詳細な解説を提供しています。
  - スケジューリングの試行に関するロジック
  - Podの`taints`や`tolerations`の確認方法
  - 各種メトリクスの記録と管理方法

## 使用方法

これらのドキュメントを参照することで、Karpenterの設計や機能について深く理解することができます。特定のトピックに関心がある場合は、該当するファイルを開いて詳細を確認してください。
