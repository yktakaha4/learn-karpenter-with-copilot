# Consolidation 概要

## Consolidation の種類

Karpenter は、リソースの使用効率を最適化し、コストを削減するために複数の種類の Consolidation をサポートしています。主な種類は以下の通りです：

### 1. Single Node Consolidation
- **説明**: 単一のノードを対象に最適化を行います。ノードがより安価または効率的なインスタンスタイプに置き換え可能かどうかを評価します。
- **主な特徴**:
  - 一度に1つのノードを対象とします。
  - ワークロードの要件を満たすことを確認した上で、置き換えノードを選択します。
  - インスタンスタイプの柔軟性を最小限に保つことで、繰り返しの統合を回避します。
- **実装**:
  - `singlenodeconsolidation.go` に定義されています。
  - 評価時間を制限するためにタイムアウト (`SingleNodeConsolidationTimeoutDuration`) を使用します。
  - 中断コストとノードプールに基づいて候補を優先順位付けします。
  - [ソースコードリンク](https://github.com/kubernetes-sigs/karpenter/blob/main/pkg/controllers/provisioning/singlenodeconsolidation.go)

### 2. Multi-Node Consolidation
- **説明**: 複数のノードを同時に評価し、最適な統合戦略を見つけます。ワークロードをより少ないノードに再分配することで効率を向上させます。
- **主な特徴**:
  - ノードのバッチを対象とします。
  - バイナリサーチを使用して、統合可能な最適なノードセットを見つけます。
  - ユーザー定義のポリシーに準拠するために中断予算を考慮します。
- **実装**:
  - `multinodeconsolidation.go` に定義されています。
  - 評価時間を制限するためにタイムアウト (`MultiNodeConsolidationTimeoutDuration`) を使用します。
  - 中断予算と再スケジュール可能な Pod に基づいて候補をフィルタリングします。
  - [ソースコードリンク](https://github.com/kubernetes-sigs/karpenter/blob/main/pkg/controllers/provisioning/multinodeconsolidation.go)

### 3. Spot-to-Spot Consolidation
- **説明**: スポットインスタンスをより安価なスポットインスタンスに置き換えることに焦点を当てた統合です。置き換えインスタンスがコスト効率が高く、ワークロード要件を満たしていることを確認します。
- **主な特徴**:
  - スポットインスタンスを対象とします。
  - 統合をトリガーするために、少なくとも15の安価なインスタンスタイプオプションが必要です。
  - インスタンスタイプの選択肢を制限することで、繰り返しの統合を回避します。
- **実装**:
  - `consolidation.go` に定義されています。
  - `computeSpotToSpotConsolidation` メソッドによって実施されます。
  - `SpotToSpotConsolidation` フィーチャーフラグによって制御されます。
  - [ソースコードリンク](https://github.com/kubernetes-sigs/karpenter/blob/main/pkg/controllers/provisioning/consolidation.go)

## 共通機能

### Base Consolidation Controller
- **説明**: 異なる統合方法で使用される共通機能を提供します。
- **主な機能**:
  - 冗長な操作を回避するために最後の統合状態を追跡します。
  - 中断ポリシーとノード条件に基づいて候補をフィルタリングします。
  - 統合オプションを評価するためにスケジューリングをシミュレーションします。
- **実装**:
  - `consolidation.go` に定義されています。
  - `IsConsolidated`、`markConsolidated`、`computeConsolidation` などのメソッドを含みます。
  - [ソースコードリンク](https://github.com/kubernetes-sigs/karpenter/blob/main/pkg/controllers/provisioning/consolidation.go)

## 重要な考慮事項
- **中断予算**: 統合は、同時に中断されるノードの数を制限するためにユーザー定義の予算を尊重します。
- **タイムアウト**: 各統合タイプには、タイムリーな意思決定を保証するためのタイムアウトが設定されています。
- **検証**: 統合コマンドは、動的なクラスター環境で有効であることを確認するために実行前に検証されます。

## 参考
- `consolidation.go`: 基本機能とスポット間統合。
- `singlenodeconsolidation.go`: 単一ノード統合ロジック。
- `multinodeconsolidation.go`: 複数ノード統合ロジック。

# Consolidation の computeSpotToSpotConsolidation メソッド

## 目的
`computeSpotToSpotConsolidation` メソッドは、スポットインスタンスをより安価なスポットインスタンスに置き換えるための統合（Spot-to-Spot Consolidation）を実行するコマンドを生成することを目的としています。このメソッドは、コスト削減を目指しつつ、クラスタのリソース効率を向上させる役割を果たします。

## 処理のロジック

### 1. フィーチャーフラグの確認
- `SpotToSpotConsolidation` フィーチャーフラグが有効であるかを確認します。
- 無効な場合、統合を中止し、空のコマンドを返します。

### 2. スポットインスタンスの要件適用
- 統合対象のノードがスポットインスタンスであることを確認し、必要な要件を追加します。

### 3. インスタンスタイプのフィルタリング
- 現在の候補ノードよりも安価なインスタンスタイプをフィルタリングします。
- フィルタリング結果が空の場合、統合を中止します。

### 4. シングルノード統合の特別条件
- 候補ノードが単一の場合、以下を確認します：
  - 少なくとも15の安価なインスタンスタイプが存在すること。
  - 現在の候補ノードがその15のインスタンスタイプに含まれていないこと。
- 条件を満たさない場合、統合を中止します。

### 5. インスタンスタイプの制限
- インスタンスタイプの選択肢を15に制限します。
- これにより、繰り返しの統合を防ぎます。

## 最終的な出力結果
- 統合可能なコマンド（`Command`）とスケジューリング結果（`scheduling.Results`）を返します。
- 統合が不可能な場合、空のコマンドと結果を返します。

## 入力に対する出力のサンプル

### 入力
- 候補ノード：
  - Node A（価格: $0.05, スポットインスタンス）
  - Node B（価格: $0.03, スポットインスタンス）
- フィーチャーフラグ：
  - SpotToSpotConsolidation: 有効

### 出力
- 統合コマンド：
  - Node A を Node B に置き換えるコマンド。
- スケジューリング結果：
  - Node A のPodが Node B に再スケジュールされる情報。

このメソッドは、スポットインスタンスのコスト効率を最大化し、クラスタのリソース利用を最適化する重要な役割を果たします。
