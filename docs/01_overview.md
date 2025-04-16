# Karpenter全体像

## 目的と役割
KarpenterはKubernetesクラスタのノード自動スケーリングを担うカスタムコントローラーです。従来のCluster Autoscalerと異なり、より柔軟かつ高速なノード管理・最適化を実現します。

## システム構成
- Karpenter本体（kubernetes-sigs/karpenter）
  - コントローラーとしてKubernetes APIサーバと連携し、CRD（NodePool, NodeClaim等）を監視・制御
  - スケジューリングやノード最適化のロジックを内包
- Provider（例: karpenter-provider-aws）
  - クラウド固有のリソース管理（EC2インスタンスの起動/停止、LaunchTemplate管理など）
  - Providerインターフェースを通じて本体と連携

## 主要なCRD
- NodePool: ノードグループの定義。スケーリングやラベル付与などのポリシーを記述
- NodeClaim: 実際に起動するノード（VM）単位のリソース要求

## Karpenterの特徴
- Providerプラグインによるクラウド拡張性
- 高速なノード起動・削除
- Bin PackingやConsolidationなどの最適化アルゴリズム
- CRDによる柔軟なユーザー操作

---

次の粒度（アーキテクチャ詳細）に進む前に、内容をご確認ください。
