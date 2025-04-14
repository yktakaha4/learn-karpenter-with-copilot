# Karpenter Documentation

## エントリーポイント

Karpenterのエントリーポイントは、`kwok/main.go` ファイルにあります。このファイルでは、以下のようにKarpenterの主要なコンポーネントが初期化され、実行されます。

```go
func main() {
	ctx, op := operator.NewOperator()
	instanceTypes, err := kwok.ConstructInstanceTypes(ctx)
	if err != nil {
		log.FromContext(ctx).Error(err, "failed constructing instance types")
	}

	cloudProvider := kwok.NewCloudProvider(ctx, op.GetClient(), instanceTypes)
	clusterState := state.NewCluster(op.Clock, op.GetClient(), cloudProvider)
	op.
		WithControllers(ctx, controllers.NewControllers(
			ctx,
			op.Manager,
			op.Clock,
			op.GetClient(),
			op.EventRecorder,
			cloudProvider,
			clusterState,
		)...).Start(ctx)
}
```

このコードでは、`operator.NewOperator()` を使用してオペレーターが初期化され、`kwok.NewCloudProvider` を通じてクラウドプロバイダーが設定されます。その後、`controllers.NewControllers` を使用してコントローラーが初期化され、`Start` メソッドで実行が開始されます。

## 処理のメインループ

Karpenterのメインループは、`operator` パッケージ内で管理されています。特に、`operator.go` ファイル内の `Start` メソッドがメインループのエントリーポイントです。このメソッドは、以下のように動作します。

1. 各コントローラーが初期化され、`WithControllers` メソッドを通じて登録されます。
2. `Start` メソッドが呼び出され、Karpenterの全体的なイベントループが開始されます。
3. 各コントローラーは、Kubernetesリソースの変更を監視し、必要に応じてノードのプロビジョニングや削除などのアクションを実行します。

以下は、`operator.go` の関連部分の抜粋です。

```go
func (o *Operator) Start(ctx context.Context) {
	// ...existing code...
	o.manager.Start(ctx)
}
```

`o.manager.Start(ctx)` は、コントローラーランタイムのマネージャーを起動し、すべてのコントローラーがKubernetes APIサーバーと連携して動作するようにします。

## スケジュール済みでまだ起動していないPodの発見とNodeへのスケジューリング

Karpenterは、スケジュール済みでまだ起動していないPodを発見するために、KubernetesのAPIサーバーからPodの状態を監視します。Podが`Pending`状態であり、かつ`NodeName`が空である場合、そのPodはスケジュール可能と判断されます。以下の手順で処理が進行します：

1. **Podのキャッシュ更新**: `Scheduler`は、`updateCachedPodData`メソッドを使用してPodの情報をキャッシュに保存します。このメソッドは、`pkg/controllers/provisioning/scheduling/scheduler.go` ファイル内で定義されています。

    ```go
    func (s *Scheduler) updateCachedPodData(p *corev1.Pod) {
        // ...メソッドの実装...
    }
    ```

2. **スケジューリングキューの作成**: `NewQueue`関数を使用して、スケジュール対象のPodをキューに追加します。この関数は、`pkg/controllers/provisioning/scheduling/queue.go` ファイル内で定義されています。

    ```go
    func NewQueue(pods []*v1.Pod, podData map[types.UID]*PodData) *Queue {
        // ...関数の実装...
    }
    ```

3. **スケジューリングの試行**: キューからPodを取り出し、既存のNodeまたは新規に作成するNodeにスケジュールを試みます。`add`メソッドを使用して、PodがNodeのリソース要件やトポロジー要件を満たすかを確認します。

4. **Nodeの作成**: 既存のNodeにスケジュールできない場合、新しいNodeを作成し、Podを割り当てます。このプロセスは、`pkg/controllers/provisioning/scheduling/scheduler.go`内の`Solve`メソッドで実装されています。

    ```go
    func (s *Scheduler) Solve(ctx context.Context, pods []*corev1.Pod) (Results, error) {
        // ...メソッドの実装...
    }
    ```

### スケジューリングの試行に関する実装

スケジューリングの試行に関するロジックは、以下のファイルとメソッドに実装されています：

1. **`pkg/controllers/provisioning/scheduling/scheduler.go`**
   - `Solve`メソッド: Podをスケジュールするための主要なロジックが含まれています。
   - `add`メソッド: Podを既存のノードまたは新しいNodeClaimに追加する処理を担当します。

#### `Solve`メソッドの概要
- **目的**: Podをスケジュール可能なノードに割り当てる。
- **主な処理**:
  1. Podの要件をキャッシュに保存。
  2. Podをキューに追加し、順次スケジューリングを試行。
  3. スケジューリングに失敗した場合、Podの要件を緩和して再試行。
  4. スケジューリング結果を記録し、エラーが発生した場合はログに出力。

#### `add`メソッドの概要
- **目的**: Podを既存のノードまたは新しいNodeClaimに追加。
- **主な処理**:
  1. 既存のノードにPodを追加できるか確認。
  2. 新しいNodeClaimを作成し、Podを追加。
  3. NodePoolのリソース制限を考慮し、インスタンスタイプをフィルタリング。

#### ソースコードへのリンク
- [scheduler.go - GitHub](https://github.com/kubernetes-sigs/karpenter/blob/main/pkg/controllers/provisioning/scheduling/scheduler.go)

---

### Podの`taints`や`tolerations`の確認について

Karpenterでは、Podの`taints`や`tolerations`を確認するコードが多く見られます。これには以下のような意図があります：

1. **スケジューリングの前提条件を満たすため**:
   - Karpenterは、Podをスケジュールする前に、NodeがPodの要件（例: `tolerations`）を満たしているかを確認します。
   - これにより、スケジューリング後にKubernetesのコントロールプレーンでエラーが発生するのを防ぎます。

2. **新しいNodeのプロビジョニング**:
   - Karpenterは、必要に応じて新しいNodeをプロビジョニングします。この際、Nodeに適用する`taints`や`labels`を決定するために、Podの要件を事前に確認します。

3. **効率的なスケジューリング**:
   - Kubernetesのスケジューラーに任せるのではなく、Karpenter自身で`taints`や`tolerations`を確認することで、スケジューリングの効率を向上させます。

#### Kubernetesコントロールプレーンとの関係
Karpenterが`taints`や`tolerations`を確認するのは、Kubernetesコントロールプレーンの検証を置き換えるものではありません。むしろ、Karpenterはスケジューリングの前段階でこれらを確認し、スケジューリングの成功率を高める役割を果たしています。

#### ソースコード例
以下は、`pkg/controllers/provisioning/scheduling/scheduler.go`内の関連コードの抜粋です：

```go
if err := scheduling.Taints(nodeClaimTemplate.Spec.Taints).ToleratesPod(pod); err != nil {
    return false
}
```

このコードは、Nodeの`taints`がPodの`tolerations`に適合するかを確認しています。これにより、スケジューリングの前提条件を満たしているかを事前にチェックしています。

#### 参考
- [scheduler.go - GitHub](https://github.com/kubernetes-sigs/karpenter/blob/main/pkg/controllers/provisioning/scheduling/scheduler.go)

---

## 不要となったNodeの発見と削除または統合

Karpenterは、不要となったNodeを発見し、削除または統合するために以下の手順を実行します：

1. **Nodeの状態監視**: `Cluster`オブジェクトがNodeの状態をキャッシュし、リソース使用率やPodの状態を追跡します。
2. **統合候補の評価**: `pkg/controllers/disruption/consolidation.go`内で、Nodeのリソース使用率が低い場合や、Podが他のNodeに移動可能な場合に統合候補としてマークされます。
3. **Podの移動**: 統合候補のNode上のPodを他のNodeに移動します。この際、`IsDrainable`メソッドを使用して、Podが安全に移動可能かを確認します。
4. **Nodeの削除**: Podがすべて移動した後、Nodeは削除されます。削除は、`pkg/controllers/node/termination/controller.go`内の`finalize`メソッドで処理されます。

これにより、クラスタのリソース効率を最大化し、不要なコストを削減します。

## karpenter-provider-awsリポジトリとの連携

Karpenterは、`karpenter-provider-aws` リポジトリと密接に連携しています。この連携は、主に以下の方法で実現されています。

1. **クラウドプロバイダーインターフェース**: `pkg/cloudprovider/types.go` ファイルで定義されている `CloudProvider` インターフェースを通じて、AWSプロバイダーが実装されています。このインターフェースは、インスタンスタイプの取得やノードのプロビジョニングなど、クラウドプロバイダー固有の操作を抽象化します。

2. **AWSプロバイダーの実装**: `karpenter-provider-aws` リポジトリは、上記のインターフェースを実装し、AWS特有のロジックを提供します。たとえば、AWSのインスタンスタイプや価格情報を取得するためのAPI呼び出しが含まれています。

3. **設定と依存関係**: `go.mod` ファイルには、`karpenter-provider-aws` リポジトリへの依存関係が記載されています。これにより、KarpenterはAWSプロバイダーのコードを直接利用できます。

以下は、`pkg/cloudprovider/types.go` の関連部分の抜粋です。

```go
type CloudProvider interface {
	Create(ctx context.Context, nodeRequest *NodeRequest) (*Node, error)
	Delete(ctx context.Context, node *Node) error
	GetInstanceTypes(ctx context.Context) ([]*InstanceType, error)
}
```

このインターフェースを実装することで、AWSプロバイダーはKarpenterのコアロジックと統合されます。

詳細については、以下のリンクを参照してください。
- [karpenter-provider-aws GitHubリポジトリ](https://github.com/aws/karpenter-provider-aws)
- [KarpenterのCloudProviderインターフェース定義](https://github.com/kubernetes-sigs/karpenter/blob/main/pkg/cloudprovider/types.go)

---

## Releaseビルドのツールチェインとバイナリ作成方法

Karpenterリポジトリでは、GitHub Actionsを使用してリリースビルドを作成しています。以下にそのプロセスを説明します。

### 使用されるツールチェイン
1. **GitHub Actions**: リリースプロセス全体を自動化するために使用されます。
2. **vexctl**: VEX (Vulnerability Exploitability eXchange) ファイルを生成するために使用されます。
3. **tejolote**: ビルドの証明書 (attestation) を生成し、署名するために使用されます。
4. **gh CLI**: GitHubリリースにファイルをアップロードするために使用されます。

### バイナリの作成方法
リリースビルドは、以下の手順で作成されます。

1. **コードのチェックアウト**:
   - `actions/checkout` アクションを使用して、リポジトリのコードを取得します。
   - `fetch-depth: 0` を指定して、完全な履歴を取得します。

2. **VEXファイルの生成**:
   - `openvex/generate-vex` アクションを使用して、`karpenter.vex.json` ファイルを生成します。
   - このファイルには、リリースに関連する脆弱性情報が含まれます。

3. **GitHubリリースの作成**:
   - `marvinpinto/action-automatic-releases` アクションを使用して、リリースを作成します。
   - `karpenter.vex.json` ファイルがリリースに添付されます。

4. **Tejoloteのインストールと実行**:
   - `kubernetes-sigs/release-actions/setup-tejolote` アクションを使用して、`tejolote` をインストールします。
   - `tejolote` を実行して、ビルドの証明書 (attestation) を生成し、署名します。
   - 生成された `karpenter.intoto.json` ファイルは、リリースに追加されます。

5. **リリースへのファイルアップロード**:
   - `gh release upload` コマンドを使用して、生成されたファイルをリリースにアップロードします。

### 注意事項
- リリースプロセスは、`v*.*.*` 形式のタグがプッシュされたときにトリガーされます。
- `GITHUB_TOKEN` が必要であり、リポジトリのシークレットとして設定されている必要があります。
- `id-token` パーミッションが必要であり、`tejolote` による署名に使用されます。

---

# Kwok について

## 導入の目的・解決される課題
Kwok (Kubernetes WithOut Kubelet) は、Kubernetes クラスターのシミュレーション環境を提供するためのツールです。主に以下の目的で使用されます：

- **コスト削減**: 実際のクラウドプロバイダーを使用せずに、Kubernetes クラスターの動作をテスト可能。
- **貢献のハードルを下げる**: 開発者がクラウドプロバイダーに依存せずにコードをテストできる環境を提供。
- **迅速なテスト**: クラウドプロバイダーのリソースを待つことなく、迅速にテストを実行可能。

Kwok は、仮想的なインスタンスタイプや価格設定を使用して、クラウドプロバイダーに依存しないシミュレーションを実現します。

## 使い方

### 必要条件
- Docker イメージをビルド、プッシュ、プルできるリポジトリ。
- Karpenter をインストール可能な Kubernetes クラスター。
  - Kind クラスターを使用する場合は、以下の環境変数を設定してください：
    ```bash
    export KWOK_REPO=kind.local
    export KIND_CLUSTER_NAME=<kind cluster name>
    ```

### インストール手順
1. Kwok をインストールします：
    ```bash
    make install-kwok
    ```
2. Karpenter を適用します：
    ```bash
    make apply
    ```

### NodePool の作成
Kwok をインストールし、Karpenter がクラスターに適用された後、以下のコマンドで NodePool を作成できます：

```bash
cat <<EOF | envsubst | kubectl apply -f -
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
      nodeClassRef:
        name: default
        kind: KWOKNodeClass
        group: karpenter.kwok.sh
      expireAfter: 720h
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
EOF
```

### テスト
以下のコマンドでテストを実行できます：
```bash
make e2etests
```
特定のテストケースを実行する場合：
```bash
export FOCUS="<e2e case name>"
make e2etests
```
