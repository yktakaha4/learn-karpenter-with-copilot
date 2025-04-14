# ProvisionerのSchedule関数について

`Schedule`関数は、Karpenterのプロビジョニングロジックの中心的な役割を果たします。この関数は、以下の手順で動作します。

## 主な処理の流れ

1. **クラスタ状態の取得**:
   クラスタ内のノードとその使用状況を収集します。この情報は、スケジューリングの基盤となります。
   - 実装箇所: `pkg/controllers/provisioning/provisioner.go` の `Schedule` 関数内 `p.cluster.Nodes()` 呼び出し。

2. **ペンディングポッドの取得**:
   スケジュール待ちのポッドを取得します。これには、削除予定のノード上のポッドも含まれます。
   - 実装箇所: `pkg/controllers/provisioning/provisioner.go` の `GetPendingPods` 関数。

3. **スケジューリングの準備**:
   スケジューリングに必要なオプションを設定し、新しいスケジューラを初期化します。
   - 実装箇所: `pkg/controllers/provisioning/provisioner.go` の `NewScheduler` 関数。

4. **スケジューリングの実行**:
   `Solve`関数を用いてポッドをノードに割り当てます。この処理はタイムアウトを設定して効率的に行われます。
   - 実装箇所: `pkg/controllers/provisioning/scheduling/scheduler.go` の `Solve` 関数。

   ```go
   func (p *Provisioner) Schedule(ctx context.Context) (scheduler.Results, error) {
       defer metrics.Measure(scheduler.DurationSeconds, map[string]string{scheduler.ControllerLabel: injection.GetControllerName(ctx)})()
       start := time.Now()

       // クラスタ内のノード情報を収集
       nodes := p.cluster.Nodes()

       // ペンディングポッドを取得
       pendingPods, err := p.GetPendingPods(ctx)
       if err != nil {
           return scheduler.Results{}, err
       }

       // 削除予定のノード上のポッドを取得
       deletingNodePods, err := nodes.Deleting().CurrentlyReschedulablePods(ctx, p.kubeClient)
       if err != nil {
           return scheduler.Results{}, err
       }

       pods := append(pendingPods, deletingNodePods...)
       if len(pods) == 0 {
           return scheduler.Results{}, nil
       }

       // スケジューラの初期化
       s, err := p.NewScheduler(ctx, pods, nodes.Active(), scheduler.DisableReservedCapacityFallback)
       if err != nil {
           return scheduler.Results{}, err
       }

       // スケジューリングの実行
       timeoutCtx, cancel := context.WithTimeout(ctx, time.Minute)
       defer cancel()
       results, err := s.Solve(timeoutCtx, pods)
       if err != nil && !errors.Is(err, context.DeadlineExceeded) {
           return scheduler.Results{}, err
       }

       results.Record(ctx, p.recorder, p.cluster)
       return results, nil
   }
   ```

5. **結果の記録**:
   スケジューリング結果を記録し、必要に応じてメトリクスを更新します。
   - 実装箇所: `pkg/controllers/provisioning/scheduling/scheduler.go` の `Results.Record` 関数。

## 関連コード

- `Schedule`関数: スケジューリング全体の流れを管理。
- `NewScheduler`関数: スケジューラの初期化を担当。
- `Solve`関数: 実際のスケジューリングロジックを実行。

これらの関数は、`pkg/controllers/provisioning/provisioner.go` および `pkg/controllers/provisioning/scheduling/scheduler.go` ファイルに実装されています。

## 参考リンク

- [Provisionerの実装コード](../../pkg/controllers/provisioning/provisioner.go)
- [Schedulerの実装コード](../../pkg/controllers/provisioning/scheduling/scheduler.go)
