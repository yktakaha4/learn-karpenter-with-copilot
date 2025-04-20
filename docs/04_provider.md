# Provider実装（AWSを例に）

## Providerインターフェースの設計
- Karpenter本体は [pkg/cloudprovider/](../../karpenter/pkg/cloudprovider/) にProviderインターフェースを定義
- 主要メソッド例:
  - `Create(ctx, nodeClass, nodeClaim)`: ノード（VM）作成要求
  - `Delete(ctx, nodeClaim)`: ノード削除要求
  - `GetInstanceTypes(ctx, nodeClass)`: 利用可能なインスタンスタイプ取得
- [pkg/cloudprovider/cloudprovider.go](../../karpenter/pkg/cloudprovider/cloudprovider.go)

## AWS Providerの主要機能
- [karpenter-provider-aws/pkg/cloudprovider/aws.go](../../karpenter-provider-aws/pkg/cloudprovider/aws.go)
- LaunchTemplate, AMI, UserData, CapacityType などAWS固有のリソース管理
- NodeClass（[karpenter-provider-aws/pkg/apis/v1beta1/nodeclass_types.go](../../karpenter-provider-aws/pkg/apis/v1beta1/nodeclass_types.go)）でサブネットやセキュリティグループ等を指定
- 実装例:
  - [Create()](../../karpenter-provider-aws/pkg/cloudprovider/aws.go#LXX)（NodeClaim/NodeClassからEC2インスタンスを起動）
  - [Delete()](../../karpenter-provider-aws/pkg/cloudprovider/aws.go#LXX)（EC2インスタンス削除）

## Provider固有の拡張ポイント・注意点
- cloudprovider/配下のインターフェースを満たせば他クラウドも実装可能
- AWS ProviderはLaunchTemplateやUserDataのマージ、AMI選択など独自ロジックを持つ
- 参考: [karpenter/designs/aws-launch-templates-options.md](../../karpenter/designs/aws-launch-templates-options.md)

## 他Provider実装との比較観点
- ProviderごとにNodeClassの仕様やリソース管理の流儀が異なる
- cloudprovider/のインターフェースを共通化することで、Karpenter本体のロジックはProvider非依存で保守可能

## 参考
- [Karpenter公式 Provider拡張ガイド](https://karpenter.sh/docs/concepts/providers/)

---

この内容で「Provider実装（AWSを例に）」の章を作成しました。ご確認の上、修正や追加要望があればご指示ください。問題なければ次（コントローラー詳細）に進みます。
