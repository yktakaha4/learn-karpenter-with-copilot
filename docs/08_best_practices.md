# ベストプラクティス・アンチパターン

## 効率的なリソース管理・コスト最適化
- NodePool/NodeClaimの要件を明確にし、必要最小限のインスタンスタイプ・ゾーンを指定
- Consolidation（統合）やDisruptionポリシーを活用し、無駄なノードを自動削除
- [designs/consolidation.md](../../karpenter/designs/consolidation.md)

## よくある落とし穴とその回避策
- NodeClassやProvider設定の不備によるノード起動失敗
  - 例: サブネットやセキュリティグループの指定ミス
- スケジューリング要件の過剰指定によるスケール失敗
  - 例: インスタンスタイプやゾーンを絞りすぎてリソース不足
- CRDバージョン不一致やWebhookバリデーションエラー
  - [pkg/apis/karpenter.sh/v1beta1/nodepool_webhook.go](../../karpenter/pkg/apis/karpenter.sh/v1beta1/nodepool_webhook.go)

## バージョンアップ・マイグレーション時の注意点
- CRD/APIのバージョン進化に合わせてマニフェストやProvider設定を更新
- [designs/v1-roadmap.md](../../karpenter/designs/v1-roadmap.md)
- 公式アップグレードガイド: [https://karpenter.sh/preview/upgrading/upgrade-guide/](https://karpenter.sh/preview/upgrading/upgrade-guide/)

## 参考Tips
- 公式ドキュメントの「Best Practices」: [https://karpenter.sh/docs/best-practices/](https://karpenter.sh/docs/best-practices/)
- 実運用で得られた知見やトラブル事例を随時追記することを推奨
