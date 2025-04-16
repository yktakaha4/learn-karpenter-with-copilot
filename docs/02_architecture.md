# アーキテクチャ詳細

## コントローラーパターンの実装
KarpenterはKubernetesのカスタムコントローラーとして、主にReconcileループを通じてクラスタ状態を監視・制御します。各種CRD（NodePool, NodeClaim等）の変更イベントをトリガーに、必要なノードの作成・削除・最適化を行います。

- コントローラーはKubernetes APIサーバと連携し、リソースの状態変化を検知
- Reconcile関数で「理想状態」と「現実状態」の差分を解消
- ワークキューによるイベント駆動型の処理

**参考ソース**:
- [pkg/controllers/](../../karpenter/pkg/controllers/)
- [pkg/controllers/nodepool/controller.go](../../karpenter/pkg/controllers/nodepool/controller.go)
- [pkg/controllers/nodeclaim/controller.go](../../karpenter/pkg/controllers/nodeclaim/controller.go)

## 主要コンポーネントの責務
- **controllers/**: 各種CRDの監視・Reconcileロジック  
  [pkg/controllers/](../../karpenter/pkg/controllers/)
- **scheduling/**: PodのスケジューリングやBin Packing等の最適化アルゴリズム  
  [pkg/scheduling/](../../karpenter/pkg/scheduling/)
- **cloudprovider/**: Providerインターフェースの定義と実装（AWS等）  
  [pkg/cloudprovider/](../../karpenter/pkg/cloudprovider/)
- **apis/**: CRDやAPI型定義  
  [pkg/apis/](../../karpenter/pkg/apis/)
- **operator/**: Karpenter自体の運用管理  
  [pkg/operator/](../../karpenter/pkg/operator/)

## Provider拡張ポイント
- cloudprovider/配下にProviderインターフェースを定義  
  [pkg/cloudprovider/](../../karpenter/pkg/cloudprovider/)
- 各Provider（例: AWS）はこのインターフェースを実装し、クラウド固有のリソース管理を担う  
  [karpenter-provider-aws/pkg/](../../karpenter-provider-aws/pkg/)
- Providerの切り替えや追加が容易な設計

## 内部データフロー・イベントフロー
1. ユーザーがNodePool等のCRDを作成・変更
2. コントローラーがイベントを検知し、Reconcileループが始動
3. 必要に応じてscheduling/でノード最適化ロジックを実行
4. cloudprovider/経由でクラウドリソース（例: EC2インスタンス）を操作
5. 状態が理想に近づくまでループ

**参考設計ドキュメント**:
- [designs/v1-api.md](../../karpenter/designs/v1-api.md)
- [designs/v1-roadmap.md](../../karpenter/designs/v1-roadmap.md)

## ノードプロビジョニングのライフサイクル
Karpenterによるノードプロビジョニングの一連の流れと、各ステップで参照される主な実装箇所を示します。

### 1. NodePool/NodeClaimの作成・変更
- ユーザーがNodePoolやNodeClaim CRDを作成・変更
- [pkg/apis/karpenter.sh/v1beta1/nodepool_types.go](../../karpenter/pkg/apis/karpenter.sh/v1beta1/nodepool_types.go)
- [pkg/apis/karpenter.sh/v1beta1/nodeclaim_types.go](../../karpenter/pkg/apis/karpenter.sh/v1beta1/nodeclaim_types.go)

### 2. コントローラーによるイベント検知
- [pkg/controllers/nodepool/controller.go](../../karpenter/pkg/controllers/nodepool/controller.go)
  - NodePoolControllerがNodePoolリソースの変更を監視
- [pkg/controllers/nodeclaim/controller.go](../../karpenter/pkg/controllers/nodeclaim/controller.go)
  - NodeClaimControllerがNodeClaimリソースの変更を監視

### 3. Reconcileループの実行
- 例: NodePoolControllerのReconcile関数
  - [Reconcile()](../../karpenter/pkg/controllers/nodepool/controller.go#LXX)（※実際の行番号は要確認）
  - NodePoolのSpecと現状を比較し、必要なNodeClaimを生成
- 例: NodeClaimControllerのReconcile関数
  - [Reconcile()](../../karpenter/pkg/controllers/nodeclaim/controller.go#LXX)
  - NodeClaimのSpecに基づき、クラウドリソース（例: EC2インスタンス）を要求

### 4. スケジューリング・最適化
- [pkg/scheduling/cluster.go](../../karpenter/pkg/scheduling/cluster.go)
- [pkg/scheduling/allocator.go](../../karpenter/pkg/scheduling/allocator.go)
  - Bin PackingやConsolidationなどのアルゴリズムで最適なノード配置を決定

### 5. Provider経由でクラウドリソース作成
- [pkg/cloudprovider/](../../karpenter/pkg/cloudprovider/)
  - Providerインターフェース（Create, Delete等）を呼び出し
- AWSの場合: [karpenter-provider-aws/pkg/cloudprovider/aws.go](../../karpenter-provider-aws/pkg/cloudprovider/aws.go)
  - 実際にEC2インスタンスやLaunchTemplateを作成

### 6. ノードオブジェクトの生成・管理
- Providerがクラウド上にノード（VM）を作成し、KubernetesのNodeリソースとして登録
- [pkg/controllers/nodeclaim/controller.go](../../karpenter/pkg/controllers/nodeclaim/controller.go)
  - NodeClaimのStatus更新や、Nodeリソースのラベル・アノテーション付与

### 7. 結果として生成・変更されるリソース
- NodePool/NodeClaim（CRD）
- Node（Kubernetesリソース）
- クラウド上のVM（例: EC2インスタンス）
- LaunchTemplate等のクラウドリソース

---
