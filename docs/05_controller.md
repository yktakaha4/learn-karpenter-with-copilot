# コントローラーの詳細解説

## NodePoolControllerの役割と実装
- NodePoolリソースの作成・変更を監視し、必要なNodeClaimを生成
- [pkg/controllers/nodepool/controller.go](../../karpenter/pkg/controllers/nodepool/controller.go)
- 主要関数:
  - `Reconcile(ctx, request)`: NodePoolのSpecと現状を比較し、NodeClaimの作成・削除を判断
  - `reconcile(ctx, nodePool)`: 実際の差分解消ロジック
- 参考: [Reconcile関数の実装例](../../karpenter/pkg/controllers/nodepool/controller.go#LXX)

## NodeClaimControllerの役割と実装
- NodeClaimリソースの作成・変更を監視し、クラウドリソース（VM）を管理
- [pkg/controllers/nodeclaim/controller.go](../../karpenter/pkg/controllers/nodeclaim/controller.go)
- 主要関数:
  - `Reconcile(ctx, request)`: NodeClaimのSpecに基づき、Provider経由でノード作成・削除を実行
  - `reconcile(ctx, nodeClaim)`: クラウドリソースの状態監視・Nodeリソースのラベル付与等
- 参考: [Reconcile関数の実装例](../../karpenter/pkg/controllers/nodeclaim/controller.go#LXX)

## Reconcileループの典型的な流れ
1. CRD（NodePool/NodeClaim）の変更イベントを検知
2. Reconcile関数が呼ばれ、現状とSpecの差分を計算
3. 必要なNodeClaimやクラウドリソースの作成・削除を実行
4. 結果をStatusやNodeリソースに反映

## エラー処理・リトライ
- エラー発生時はワークキューに再投入し、リトライを自動化
- [controller-runtimeのReconcileパターン](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/reconcile)

## イベントハンドリング・ワークキュー
- [pkg/controllers/nodepool/controller.go](../../karpenter/pkg/controllers/nodepool/controller.go)
- [pkg/controllers/nodeclaim/controller.go](../../karpenter/pkg/controllers/nodeclaim/controller.go)
- Informer/WorkQueueによるイベント駆動型の設計
