# スケジューリング・最適化ロジック

## スケジューリングの全体像
- Karpenterは、Podのスケジューリング要求に応じて最適なノード（NodeClaim）を決定
- Bin PackingやConsolidationなどのアルゴリズムでリソース効率を最大化
- [pkg/scheduling/](../../karpenter/pkg/scheduling/) 配下に主要ロジックが実装

## Bin Packingアルゴリズム
- [pkg/scheduling/allocator.go](../../karpenter/pkg/scheduling/allocator.go)
  - `Allocate()`関数で、Podのリソース要求を満たす最小限のノードセットを計算
  - 既存ノードへの割り当てと新規ノードの必要性を判定
- [pkg/scheduling/cluster.go](../../karpenter/pkg/scheduling/cluster.go)
  - クラスタ全体のリソース状況を管理

## Consolidation（統合・最適化）
- [pkg/scheduling/consolidation/](../../karpenter/pkg/scheduling/consolidation/)
  - 不要なノードの削除や、より効率的なノード構成への再配置を実施
  - 例: 空ノードや低利用ノードの自動削除

## スケジューリング戦略のカスタマイズ
- [pkg/apis/karpenter.sh/v1beta1/nodepool_types.go](../../karpenter/pkg/apis/karpenter.sh/v1beta1/nodepool_types.go)
  - NodePoolのSpecで、スケジューリング要件や優先度、ゾーン、インスタンスタイプ等を柔軟に指定可能

## パフォーマンス・スケーラビリティの考慮点
- スケジューリング処理はイベント駆動で非同期に実行
- 大規模クラスタでも効率的に動作するよう、リソース計算やノード最適化のアルゴリズムが工夫されている
- [designs/bin-packing.md](../../karpenter/designs/bin-packing.md)
- [designs/consolidation.md](../../karpenter/designs/consolidation.md)

## 参考
- [Karpenter公式 スケジューリング解説](https://karpenter.sh/docs/concepts/scheduling/)
