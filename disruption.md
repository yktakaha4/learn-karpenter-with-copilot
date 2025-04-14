# Candidate.DisruptionCost のライフサイクル

`Candidate.DisruptionCost` は、Karpenter のノード統合プロセスにおいて、ノードの中断コストを表す重要なプロパティです。この値は、ノードの統合可能性を評価し、最適な統合戦略を決定するために使用されます。以下に、`DisruptionCost` のライフサイクルを説明します。

## 1. 初期化

`DisruptionCost` は、`Candidate` オブジェクトの作成時に初期化されます。この値は、以下の要素に基づいて計算されます：

- **再スケジューリングコスト**: ノード上の Pod を再スケジューリングするためのコスト。
- **ノードの残り寿命**: ノードが終了するまでの推定時間。

これらの要素は、`disruptionutils.ReschedulingCost` と `disruptionutils.LifetimeRemaining` を使用して計算されます。

```go
DisruptionCost: disruptionutils.ReschedulingCost(ctx, pods) * disruptionutils.LifetimeRemaining(clk, nodePool, node.NodeClaim),
```

## 各コストの詳細

### 再スケジューリングコスト

再スケジューリングコストは、ノード上の Pod を再スケジューリングする際のコストを表します。このコストは、以下の要素に基づいて計算されます：

- **Pod 削除コスト**: Pod のアノテーション `corev1.PodDeletionCost` に基づいて計算されます。この値は、Pod の削除がクラスタに与える影響を示します。
- **スケジューリング優先度**: Pod の `Spec.Priority` に基づいて計算されます。この値は、Pod の重要度を示します。

これらの値は、`disruption.EvictionCost` 関数を使用して計算され、最終的に `disruption.ReschedulingCost` 関数で合計されます。

```go
func ReschedulingCost(ctx context.Context, pods []*corev1.Pod) float64 {
    cost := 0.0
    for _, p := range pods {
        cost += EvictionCost(ctx, p)
    }
    return cost
}
```

### PodDeletionCost（Pod 削除コスト）

`PodDeletionCost` は、Kubernetes の Pod に設定できるアノテーションで、Pod の削除順序を制御するために使用されます。この値は、ReplicaSet や他のコントローラーが Pod を削除する際の優先順位を決定するために考慮されます。

- **アノテーションのキー**: `"controller.kubernetes.io/pod-deletion-cost"`
- **値の範囲**: 整数値で指定され、正の値は削除されにくいことを示し、負の値は削除されやすいことを示します。

このアノテーションは、特定の Pod を優先的に保持したい場合や、特定の Pod を優先的に削除したい場合に役立ちます。

#### `EvictionCost` 関数との関係

`EvictionCost` 関数では、`PodDeletionCost` の値を使用して削除コストを計算します。この値は、以下のようにスケールダウンされ、最終的な削除コストに加算されます：

```go
cost += podDeletionCost / math.Pow(2, 27.0)
```

この計算により、`PodDeletionCost` の影響が適切に調整され、削除コスト全体に反映されます。

#### 使用例

以下は、`PodDeletionCost` を設定する例です：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
  annotations:
    controller.kubernetes.io/pod-deletion-cost: "100"
spec:
  containers:
  - name: example-container
    image: nginx
```

この例では、`example-pod` の削除コストが高く設定されており、他の Pod よりも削除されにくくなります。

### Pod Priority（Pod の優先度）

`Pod Priority` は、Kubernetes のスケジューリングにおいて、Pod の重要度を示すために使用されます。優先度の高い Pod は、リソースが不足している場合に優先的にスケジュールされ、必要に応じて低優先度の Pod をプリエンプト（強制終了）することができます。

#### 優先度の設定

Pod の優先度は、`PriorityClass` を使用して設定されます。`PriorityClass` は、クラスター管理者が作成するリソースで、各クラスに関連付けられた整数値（優先度）を定義します。優先度の値が高いほど、Pod の重要度が高くなります。

以下は、`PriorityClass` の例です：

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
preemptionPolicy: PreemptLowerPriority
globalDefault: false
description: "This priority class is for high priority pods."
```

この例では、`high-priority` という名前の優先度クラスが作成され、値は `1000` に設定されています。

#### `EvictionCost` 関数との関係

`EvictionCost` 関数では、Pod の優先度を使用して削除コストを計算します。優先度の値は、以下のようにスケールダウンされ、削除コストに加算されます：

```go
if p.Spec.Priority != nil {
    cost += float64(*p.Spec.Priority) / math.Pow(2, 25)
}
```

この計算により、優先度の高い Pod は削除コストが高くなり、削除されにくくなります。

#### 使用例

以下は、`PriorityClass` を使用して Pod の優先度を設定する例です：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: high-priority-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: example-container
    image: nginx
```

この例では、`high-priority` クラスが適用され、Pod の優先度が高く設定されています。

### EvictionCost（削除コスト）

`EvictionCost` 関数は、特定の Pod を削除する際の中断コストを計算します。このコストは、以下の要素に基づいて計算されます：

- **Pod 削除コスト**: Pod のアノテーション `corev1.PodDeletionCost` に基づいて計算されます。この値は、Pod の削除がクラスタに与える影響を示します。
  - Pod 削除コストは、`strconv.ParseFloat` を使用して文字列から浮動小数点数に変換されます。
  - 値が範囲外または無効な場合、エラーが記録されます。
- **スケジューリング優先度**: Pod の `Spec.Priority` に基づいて計算されます。この値は、Pod の重要度を示します。

これらの値は、以下のように計算されます：

1. Pod 削除コストは、`math.Pow(2, 27.0)` でスケールダウンされます。
2. スケジューリング優先度は、`math.Pow(2, 25)` でスケールダウンされます。
3. 最終的なコストは、`lo.Clamp` を使用して `[-10.0, 10.0]` の範囲に制限されます。

以下は、`EvictionCost` 関数の実装例です：

```go
func EvictionCost(ctx context.Context, p *corev1.Pod) float64 {
    cost := 1.0
    podDeletionCostStr, ok := p.Annotations[corev1.PodDeletionCost]
    if ok {
        podDeletionCost, err := strconv.ParseFloat(podDeletionCostStr, 64)
        if err != nil {
            log.FromContext(ctx).Error(err, fmt.Sprintf("failed parsing %s=%s from pod %s",
                corev1.PodDeletionCost, podDeletionCostStr, client.ObjectKeyFromObject(p)))
        } else {
            cost += podDeletionCost / math.Pow(2, 27.0)
        }
    }

    if p.Spec.Priority != nil {
        cost += float64(*p.Spec.Priority) / math.Pow(2, 25)
    }

    return lo.Clamp(cost, -10.0, 10.0)
}
```

### ノードの残り寿命

ノードの残り寿命は、ノードが終了するまでの推定時間を基に計算されます。この値は、ノードの作成時刻と `ExpireAfter` プロパティに基づいて計算されます。

- **ExpireAfter**: ノードの有効期限を示すプロパティ。この値が設定されている場合、ノードの寿命がスケールダウンされます。

この計算は、`disruption.LifetimeRemaining` 関数を使用して行われます。

```go
func LifetimeRemaining(clock clock.Clock, nodePool *v1.NodePool, nodeClaim *v1.NodeClaim) float64 {
    remaining := 1.0
    if nodeClaim.Spec.ExpireAfter.Duration != nil {
        ageInSeconds := clock.Since(nodeClaim.CreationTimestamp.Time).Seconds()
        totalLifetimeSeconds := nodeClaim.Spec.ExpireAfter.Duration.Seconds()
        lifetimeRemainingSeconds := totalLifetimeSeconds - ageInSeconds
        remaining = lo.Clamp(lifetimeRemainingSeconds/totalLifetimeSeconds, 0.0, 1.0)
    }
    return remaining
}
```

## 2. 利用

`DisruptionCost` は、主に以下の場面で利用されます：

### a. 候補ノードのソート

統合候補ノードは、`DisruptionCost` に基づいてソートされます。これにより、中断コストが最も低いノードが優先的に統合されます。

```go
sort.Slice(candidates, func(i int, j int) bool {
    return candidates[i].DisruptionCost < candidates[j].DisruptionCost
})
```

### b. 統合可能性の評価

統合プロセス中に、`DisruptionCost` を使用してノードの統合可能性を評価します。中断コストが高すぎる場合、そのノードは統合対象から除外されることがあります。

## 3. 破棄

`DisruptionCost` は、以下の条件で無効化または破棄されることがあります：

- **ノードの状態が変更された場合**: 例えば、ノードが削除中の状態になった場合。
- **Pod のスケジューリングエラー**: 全ての Pod を再スケジュールできない場合。

これらの条件が発生すると、統合プロセスは中断され、`DisruptionCost` は再計算されるか、破棄されます。

```go
if !results.AllNonPendingPodsScheduled() {
    return Command{}, pscheduling.Results{}, nil
}
```

## まとめ

`Candidate.DisruptionCost` は、Karpenter の統合プロセスにおいて重要な役割を果たします。この値は、ノードの統合可能性を効率的に評価し、クラスタのリソース最適化を実現するために使用されます。

# DisruptionReason

## 概要
`DisruptionReason` は、ノードプールのディスラプション（中断や変更）の理由を表す列挙型です。これにより、特定の理由に基づいてノードのスケジュールや削除が制御されます。

## 主なDisruptionReason
以下は、現在定義されている主な `DisruptionReason` です。

1. **DisruptionReasonEmpty**
   - ノードが空である場合に設定されます。
   - 主にリソースの効率化のために使用されます。

2. **DisruptionReasonUnderutilized**
   - ノードが十分に活用されていない場合に設定されます。
   - リソースの最適化を目的として、ノードの統合や削除が検討されます。
   - **関連ソースコード**:
     - `/pkg/controllers/disruption/singlenodeconsolidation.go` の `Reason` メソッド:
       ```go
       func (s *SingleNodeConsolidation) Reason() v1.DisruptionReason {
           return v1.DisruptionReasonUnderutilized
       }
       ```
     - `/pkg/controllers/disruption/multinodeconsolidation.go` の `Reason` メソッド:
       ```go
       func (m *MultiNodeConsolidation) Reason() v1.DisruptionReason {
           return v1.DisruptionReasonUnderutilized
       }
       ```

3. **DisruptionReasonDrifted**
   - ノードが期待される状態から逸脱している場合に設定されます。
   - 例として、ノードの設定やラベルが変更された場合が含まれます。

## ライフサイクル

### DisruptionReasonEmpty
- **設定タイミング**: ノードが空であることが検出されたとき。
- **利用タイミング**: 空のノードを削除する際に使用されます。

### DisruptionReasonUnderutilized
- **設定タイミング**: ノードのリソース使用率が一定の閾値を下回ったとき。
- **関連ソースコード**:
  - `/pkg/controllers/disruption/singlenodeconsolidation.go` の `Reason` メソッド:
    ```go
    func (s *SingleNodeConsolidation) Reason() v1.DisruptionReason {
        return v1.DisruptionReasonUnderutilized
    }
    ```
  - `/pkg/controllers/disruption/multinodeconsolidation.go` の `Reason` メソッド:
    ```go
    func (m *MultiNodeConsolidation) Reason() v1.DisruptionReason {
        return v1.DisruptionReasonUnderutilized
    }
    ```
- **利用タイミング**: リソースの効率化を目的として、ノードの統合や削除を行う際に使用されます。

### DisruptionReasonDrifted
- **設定タイミング**: ノードが期待される状態から逸脱していることが検出されたとき。
- **利用タイミング**: 逸脱したノードを修正または再作成する際に使用されます。

## まとめ
`DisruptionReason` は、ノードプールの管理において重要な役割を果たします。各理由は特定の条件下で設定され、ノードのライフサイクル管理に利用されます。これにより、リソースの効率化やシステムの安定性が向上します。
