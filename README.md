# TensorFlowAdventCalendar2018
# 課題意識
- どうやってTensorflowを含む機械学習フレームワークを使ったモデルをシステムにデプロイし、継続的に使い続けるか
- Tensorflow Data Validationだけの評価でしてもしょうがないのでどの程度MLworkflowが展開しやすいのか調べる
- Kubeflow pipeline上で実現したい。KubeflowはTFXを包含。そのワークフローの中で活用している。
- KubeflowはGKE上にデプロイする。TFDVはApache Beamが必要になるから、Dataflowとか繋げられるの便利。
- On-preでやりたいよってところはCiscoのUCSでkubeflowサポートしてるし、ハイブリッドでやるならGKEとUSCで、管理ツールとしてGKE-onpreかな

# 用語の整理
- kubeflow pipelines(https://github.com/kubeflow/pipelines)
  - Kubeflow Pipelines SDK ってなんや
  − Documents are here -> https://github.com/kubeflow/pipelines/wiki
  - パイプラインをpythonDSLで記述できる
- TFJob CRD(Cu7stom Resource Definitions library): Kube上でTFの分散学習できるようにする
  - https://github.com/kubeflow/tf-operator/blob/master/tf_job_design_doc.md
  - 基本的なリソースはすでにkubernetesに定義されているが、それ以外のものをCustomeで定義できる。(https://thinkit.co.jp/article/13610)
  - TFjob はkubeflowで定義したCustomResourceDefinitions？]
    - aaa
    - bbb


- https://medium.com/tensorflow/introducing-tensorflow-data-validation-data-understanding-validation-and-monitoring-at-scale-d38e3952c2f0
- https://github.com/tensorflow/data-validation


- https://chinagdg.org/2018/11/getting-started-with-kubeflow-pipelines/


## Tensorflow Data Validation
- 統計量のサマリを出してくれる
- それぞれのFeatureのデータの分布や統計量、TrainとTestの差を見せてくれる
- データスキーマを勝手に作ってくれる、どんな値をとるか、範囲、ボキャブラリー(?)
- 異常値を探してくれる。

##
Kubeflow piplinesでやるよ
- https://chinagdg.org/2018/11/getting-started-with-kubeflow-pipelines/



### Examples
- https://github.com/amygdala/code-snippets/tree/master/ml/kubeflow-pipelines

