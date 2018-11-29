# TensorFlowAdventCalendar2018
# 課題意識
- どうやってTensorflowを含む機械学習フレームワークを使ったモデルをシステムにデプロイし、継続的に使い続けるか
- Tensorflow Data Validationだけの評価でしてもしょうがないのでどの程度MLworkflowが展開しやすいのか調べる
- Kubeflow pipeline上で実現したい。KubeflowはTFXを包含。そのワークフローの中で活用している。
- KubeflowはGKE上にデプロイする。TFDVはApache Beamが必要になるから、Dataflowとか繋げられるの便利。
- On-preでやりたいよってところはCiscoのUCSでkubeflowサポートしてるし、ハイブリッドでやるならGKEとUSCで、管理ツールとしてGKE-onpreかな

# 用語の整理
- kubeflow pipelines(https://github.com/kubeflow/pipelines)
  - Kubeflow Pipelines SDK ってなんやhttps://github.com/kubeflow/pipelines/wiki
  − Documents are here -> https://github.com/kubeflow/pipelines/wiki
  - パイプラインをpythonDSLで記述できる
- TFJob CRD(Cu7stom Resource Definitions library): Kube上でTFの分散学習できるようにする
  - https://github.com/kubeflow/tf-operator/blob/master/tf_job_design_doc.md
  - 基本的なリソースはすでにkubernetesに定義されているが、それ以外のものをCustomeで定義できる。(https://thinkit.co.jp/article/13610)
  - TFjob はkubeflowで定義したCustomResourceDefinitions？。Driverとかのマウントが必要無くなる
    - TFJobSpecs:The actual specification of our TensorFlow job, defined below.
    - TFReplicaSpec: Specification for a set of TensorFlow processes, defined below.
  - これを動かすと、tf-job-operator-configというConfigMap、Deployment, "tf-job-operator"というPodができる
  - TFJob is a Kubernetes custom resource that makes it easy to run TensorFlow training jobs on Kubernetes.
  
  


## Introducting Kubeflow Pipelines
- Kubeflow Pipelines はML workflowsをE2Eでmanagementするツール
- https://github.com/amygdala/code-snippets/tree/master/ml/kubeflow-pipelines
- https://github.com/tensorflow/model-analysis/tree/master/examples/chicago_taxiこれと同じ



## TFX building blocks
- Kubeflowは TFXのコンポーネントをブロックとして使ってます。
- TensorFlow Transform, TEMA, TF serving


## TFT
- tf.transform はtraning-serving skewを解決する、それはtrainとtestで前処理の仕方が違う。チームが異なってたり、計算資源が異なると起こり得る。
- TFTのアウトプットはTF graphとして出力される
- TFT はApache Beamを使ってTFTするよ、Google Cloud DataflowはManagedのApache Beamだよ
- ExamplesはlocalのBeamを使っているけど。dataflowを使うこともできるよ
− 

## TFMA
- TFMA は様々な状況下や特徴、subsetにおけるモデルのパフォーマンスをビジュアライズしてくれる
− TFT と同様にBeamが必要。
- ExampleはTFDVを直接は使ってないけど、schemaはTFDVによって作られたものだよ

## Cloud ML Engine Online Prediction
- 今回のexampleじゃ使ってないよ
- モデルの管理ができていいよ

## Building workflows using kubeflow pipelines 

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

↓コマンドラインで
https://github.com/kubeflow/pipelines/wiki/Deploy-the-Kubeflow-Pipelines-Service#deploy-kubeflow-pipelines

## はじめに
- gcloud をインストールする
- APIsをEnableにしておく
  - Dataflow
  - BigQuery
  - Cloud ML Engine
  - Kubefnetes Engine
  
 ## Set up a Kubernetes Engine (GKE) cluster
 - GKEクラスタを立ち上げる
  - Machine type 8 vCPUs、Allow full accessto all cloud APIs
  - 詳細はスクショ
 - 'kubectl' コンテクストをセットします。
 
```
 gcloud beta container --project "mlops-215604" clusters create "kubeflow-pipelines" --zone "us-central1-a" --username "admin" --cluster-version "1.9.7-gke.11" --machine-type "custom-8-40960" --image-type "COS" --disk-type "pd-standard" --disk-size "100" --scopes "https://www.googleapis.com/auth/cloud-platform" --num-nodes "4" --enable-cloud-logging --enable-cloud-monitoring --no-enable-ip-alias --network "projects/mlops-215604/global/networks/default" --subnetwork "projects/mlops-215604/regions/us-central1/subnetworks/default" --addons HorizontalPodAutoscaling,HttpLoadBalancing,KubernetesDashboard --enable-autoupgrade --enable-autorepair
```
 
 
 ###  cloud shellの中で
```
gcloud container clusters get-credentials kubeflow-pipelines --zone us-central1-a --project mlops-215604
 
# kubectl create clusterrolebinding default-admin --clusterrole=cluster-admin --user=taketoshi.kazusa@brainpad.cp.jp
> kubectl create clusterrolebinding ml-pipeline-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)

clusterrolebinding.rbac.authorization.k8s.io "ml-pipeline-admin-binding" created

> kubectl create clusterrolebinding sa-admin --clusterrole=cluster-admin --serviceaccount=kubeflow:pipeline-runner

clusterrolebinding.rbac.authorization.k8s.io "sa-admin" created
```


```
PIPELINE_VERSION=0.1.2
kubectl create -f https://storage.googleapis.com/ml-pipeline/release/$PIPELINE_VERSION/bootstrapper.yaml
```

```
kubectl get job
NAME                       DESIRED   SUCCESSFUL   AGE
deploy-ml-pipeline-rt8zf   1         1            14h

kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.7.240.1   <none>        443/TCP   14h
```
あれ？kubernetes本体以外のサービスたってなくね？ポートフォワード意味ある？

ここ以降は下記を読みなさい。
https://github.com/amygdala/code-snippets/tree/master/ml/kubeflow-pipelines/samples/kubeflow-tf

## Install python SDK (python 3.5 above) 
https://github.com/kubeflow/pipelines/releases
```
pip3 install https://storage.googleapis.com/ml-pipeline/release/0.1.2/kfp.tar.gz --upgrade
```

## run the example pipelines
```
python3 workflow1.py
```


Kubeflow pipelines UIにローカルのブラウザからGKE上のpod内のコンテナにアクセスできるようにポートフォワードの設定をしておく。
```
export NAMESPACE=kubeflow
kubectl port-forward -n ${NAMESPACE} $(kubectl get pods -n ${NAMESPACE} --selector=service=ambassador -o jsonpath='{.items[0].metadata.name}') 8080:80
```

Kubeflowは出てくるけど、kubeflowpipelinesのUIは出てこない

https://github.com/kubeflow/pipelines/wiki/Build-a-Pipeline


Cloudshellからは”web preview”末尾に"pipeline"を付けてアクセス
https://8080-dot-3326024-dot-devshell.appspot.com/pipeline/#/pipelines

入れた！

UIからgz.tarごとアップロード。

ExperimentsとRunを定義する画面に映るので設定してみる。

project: mlops-215604
working-dir: gs://bp-kubeflow-pipelines/

結果が走る。pipelineの今どこにいるかも可視化してくれている。

kubeflow pipelines　UI -> notebook.
ID:admin
pass:admin



Jupyternotebookに入れる。



# Draft
## 手順
これに従う
- https://github.com/amygdala/code-snippets/tree/master/ml/kubeflow-pipelines

GKEクラスタ立ち上げる

```
> gcloud beta container --project "mlops-215604" clusters create "kubeflow-pipelines" --zone "us-central1-a" --username "admin" --cluster-version "1.9.7-gke.11" --machine-type "custom-8-40960" --image-type "COS" --disk-type "pd-standard" --disk-size "100" --scopes "https://www.googleapis.com/auth/cloud-platform" --num-nodes "4" --enable-cloud-logging --enable-cloud-monitoring --no-enable-ip-alias --network "projects/mlops-215604/global/networks/default" --subnetwork "projects/mlops-215604/regions/us-central1/subnetworks/default" --addons HorizontalPodAutoscaling,HttpLoadBalancing,KubernetesDashboard --enable-autoupgrade --enable-autorepair
 ```

gcloudコマンドと紐付ける
```
> gcloud container clusters get-credentials kubeflow-pipelines --zone us-central1-a --project mlops-215604
Fetching cluster endpoint and auth data.
kubeconfig entry generated for kubeflow-pipelines.
> kubectl create clusterrolebinding ml-pipeline-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)
clusterrolebinding.rbac.authorization.k8s.io "ml-pipeline-admin-binding" created
> kubectl create clusterrolebinding sa-admin --clusterrole=cluster-admin --serviceaccount=kubeflow:pipeline-runner
clusterrolebinding.rbac.authorization.k8s.io "sa-admin" created
```


https://www.kubeflow.org/docs/guides/pipelines/deploy-pipelines-service/ に従ってkubeflow pipelineをデプロイする。

```
> PIPELINE_VERSION=0.1.2
# job を作る
> kubectl create -f https://storage.googleapis.com/ml-pipeline/release/$PIPELINE_VERSION/bootstrapper.yaml
clusterrole.rbac.authorization.k8s.io "mlpipeline-deploy-admin" created
clusterrolebinding.rbac.authorization.k8s.io "mlpipeline-admin-role-binding" created
job.batch "deploy-ml-pipeline-qqk9j" created
# 
> kubectl get job
NAME                       DESIRED   SUCCESSFUL   AGE
deploy-ml-pipeline-qqk9j   1         1            7m
> kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.7.240.1   <none>        443/TCP   18m
```

Kubeflow Pipelines のpythonSDKをインストールする 
https://github.com/kubeflow/pipelines/releases
```
> pip3 install https://storage.googles.com/ml-pipeline/release/0.1.2/kfp.tar.gz --upgrade
Installing collected packages: urllib3, six, certifi, python-dateutil, PyYAML, google-resumable-media, pytz, setuptools, chardet, idna, requests, protobuf, cachetools, pyasn1, rsa, pyasn1-modules, google-auth, googleapis-common-protos, google-api-core, google-cloud-core, google-cloud-storage, pycparser, cffi, asn1crypto, cryptography, PyJWT, adal, oauthlib, requests-oauthlib, websocket-client, kubernetes, kfp
  Running setup.py install for kfp ... done
Successfully installed PyJWT-1.6.4 PyYAML-3.13 adal-1.2.0 asn1crypto-0.24.0 cachetools-3.0.0 certifi-2018.10.15 cffi-1.11.5 chardet-3.0.4 cryptography-2.4.2 google-api-core-1.5.2 google-auth-1.6.1 google-cloud-core-0.28.1 google-cloud-storage-1.13.0 google-resumable-media-0.3.1 googleapis-common-protos-1.5.5 idna-2.7 kfp-0.1 kubernetes-8.0.0 oauthlib-2.1.0 protobuf-3.6.1 pyasn1-0.4.4 pyasn1-modules-0.2.2 pycparser-2.19 python-dateutil-2.7.5 pytz-2018.7 requests-2.20.1 requests-oauthlib-1.0.0 rsa-4.0 setuptools-40.6.2 six-1.11.0 urllib3-1.24.1 websocket-client-0.54.0
```

`~/code-snippets/ml/kubeflow-pipelines/samples/kubeflow-tf`へ移動する。
README.mdにある手順に従ってpipelinesのexampleを試してみる。

```
python3 workflow1.py
```
Kubeflow pipelines UIにローカルのブラウザからGKE上のpod内のコンテナにアクセスできるようにポートフォワードの設定をしておく。
```
export NAMESPACE=kubeflow
kubectl port-forward -n ${NAMESPACE} $(kubectl get pods -n ${NAMESPACE} --selector=service=ambassador -o jsonpath='{.items[0].metadata.name}') 8080:80
```

Cloudshellからは”web preview”末尾に"pipeline"を付けるとkubeflow pipeliensのUIへ飛ぶことができる。
https://8080-dot-3326024-dot-devshell.appspot.com/pipeline/#/pipelines


projectとworkind-dirをそれぞれ下記に設定
project: mlops-215604
working-dir: gs://bp-kubeflow-pipelines/


## workflow1をやってみる
- preprocess-modeとXX-modeをlocalで。
- 動いた: スクリーンショット


## Juptyter notebookでTFMAを可視化
- https://www.kubeflow.org/docs/guides/components/jupyter/
browser previewでkubeflowにアクセス、jupyterhubへ。
- cloudshellから~/~/code-snippets/ml/kubeflow-pipelines/components/dataflow/tfma/tfma_expers.ipynbをダウンロード
- jupyter上へ



















# サンプルがいくつかある
BasicとあるものはこのRepoにものがpipelineとしてpython dslとして記述されており、compileされてkubeflow piplinesにアップロードされた状態で置いてある。

スクショ：pipeliensトップ画面スクショ

このBuild a Pipeline(https://www.kubeflow.org/docs/guides/pipelines/build-pipeline/#compile-the-samples)にもあるようにcompileされたのだろう

https://github.com/kubeflow/pipelines/tree/master/samples/basic
- Condition:  https://github.com/kubeflow/pipelines/blob/master/samples/basic/condition.py
- Exit handler: https://github.com/kubeflow/pipelines/blob/master/samples/basic/exit_handler.py
- Immediate Value: https://github.com/kubeflow/pipelines/blob/master/samples/basic/immediate_value.py
- parallel_join: https://github.com/kubeflow/pipelines/blob/master/samples/basic/parallel_join.py
- sequential.py: https://github.com/kubeflow/pipelines/blob/master/samples/basic/sequential.py
- ML - TFX: Example pipeline that does classification with model analysis based on a public tax cab BigQuery dataset.
  - https://github.com/kubeflow/pipelines/tree/master/samples/tfx
  ->TFTとTFMAが一緒に楽しめる？しかもdataflow上で？
- ML - XGboost: https://github.com/kubeflow/pipelines/tree/master/samples/xgboost-spark
  - Google DataProc clusterを作る (Hadoop, spark)
  - TFあんまり関係なさそう

## TFX　Experimentsをやってみたいがエラーが出る。。。
- Experimentsのところにvalidation-modeとpreprocess-modeがありましてDefaultはlocalになっている->CloudにするとDataflowが動く。API許可しておこう
- predict-modeとanalyse-modeは？

とりあえず、localで走らせてみた。
- 

TODO:
- https://github.com/tensorflow/model-analysis/tree/master/examples/chicago_taxi を読む
- 4つのパートがある
  - TFDV
  - TFT
  - Train
  - TFMA
  -
   
   





