# CRDとAPI設計

## 主要なCRDの構造
Karpenterは主に以下のCRD（Custom Resource Definition）を用いてノード管理を行います。

### NodePool
- ノードグループの定義。スケーリングやラベル付与などのポリシーを記述
- [pkg/apis/karpenter.sh/v1beta1/nodepool_types.go](../../karpenter/pkg/apis/karpenter.sh/v1beta1/nodepool_types.go)
- 主要フィールド例:
  - `spec.template.spec.requirements`: ノード要件（インスタンスタイプ、ゾーン等）
  - `spec.disruption`: ノードの置き換えや削除に関するポリシー

### NodeClaim
- 実際に起動するノード（VM）単位のリソース要求
- [pkg/apis/karpenter.sh/v1beta1/nodeclaim_types.go](../../karpenter/pkg/apis/karpenter.sh/v1beta1/nodeclaim_types.go)
- 主要フィールド例:
  - `spec.nodeClassRef`: 利用するNodeClass（Provider固有設定）への参照
  - `status.providerID`: 実際に割り当てられたクラウドリソースID

### NodeClass（Provider固有）
- Provider（例: AWS）ごとのノード仕様を記述
- [karpenter-provider-aws/pkg/apis/v1beta1/nodeclass_types.go](../../karpenter-provider-aws/pkg/apis/v1beta1/nodeclass_types.go)
- 主要フィールド例:
  - `spec.subnetSelectorTerms`, `spec.securityGroupSelectorTerms` など

## CRDのバリデーション・デフォルト化
- [pkg/apis/karpenter.sh/v1beta1/nodepool_webhook.go](../../karpenter/pkg/apis/karpenter.sh/v1beta1/nodepool_webhook.go)
- [pkg/apis/karpenter.sh/v1beta1/nodeclaim_webhook.go](../../karpenter/pkg/apis/karpenter.sh/v1beta1/nodeclaim_webhook.go)
- Webhookによるバリデーション・デフォルト値の自動付与

## APIバージョン管理
- [designs/v1-api.md](../../karpenter/designs/v1-api.md)
- [designs/v1-roadmap.md](../../karpenter/designs/v1-roadmap.md)
- v1alpha, v1beta, v1など段階的なAPI進化の設計思想

## ユーザー操作例
```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: example-pool
spec:
  template:
    spec:
      requirements:
        - key: "node.kubernetes.io/instance-type"
          operator: In
          values: ["m5.large", "c5.large"]
  disruption:
    consolidationPolicy: WhenEmpty
```

## 参考
- [Karpenter公式ドキュメント CRDリファレンス](https://karpenter.sh/docs/reference/api/)

---

この内容で「CRDとAPI設計」の章を作成しました。ご確認の上、修正や追加要望があればご指示ください。問題なければ次（Provider実装）に進みます。
