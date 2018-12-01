# TensorFlowAdventCalendar2018
この記事はTensorFlow Advent Calendar 2018年 3日目の記事です。今年のAdvent Calendarは[Kubeflow pipeliens](https://github.com/kubeflow/pipelines/wiki)について書こうと思います。


# 機械学習システムの実環境へのデプロイ&サービング
機械学習が普及した2018年ですが、PoC(Proof of Concept)を超えて実運用まで漕ぎ着けている事例が増えてきたとはいえ、実システムに組み込んで運用する場合のハードルは依然高いように見えます。 その理由としては、2014年にGoogleから出された論文[Machine Learning: The High Interest Credit Card of Technical Debt] (https://ai.google/research/pubs/pub43146) でいくつか課題が挙げられており、それらの一つの解決策として機械学習プラットフォームである[TensorFlow Extended(TFX)](https://www.tensorflow.org/tfx/)が提案されています。

現在、OSSとして公開されているTFXはそれぞのコンポーネントがバラバラであり、機械学習のワークフロー全体としては管理しづらいものでした。そこで機械学習のワークフロー全体をEndToEndで管理できるようにするためのコンポーネントがkubeflow pipelineです。以前から機械学習システム構築するためのツールキットである[Kubeflow](https://www.kubeflow.org/)にTFXはその一部が取り込まれていましたが、今年11月に[発表](https://cloud-ja.googleblog.com/2018/11/introducing-ai-hub-and-kubeflow-pipelines-making-ai-simpler-faster-and-more-useful-for-businesses.html)されたKubeflow pipelinesでワークフローの管理が洗練されより使いやすくなったように感じます。

2017年末にkubeflowが出てきてから一年、kubeflow自体はまだ0.4と発展途上であり、公式のexamplesもまともに動かなかったりします。このkubeflow pipelinesも例に漏れずexampleを動かすのさえ苦行ではありますが、ユーザーが増えて知見が貯まることを願ってご紹介をしようと思います。

# Kubeflow pipelines

kubeflow pipelinesはKubeflowの新しいコンポーネントであり、機械学習システムのワークフローを管理できるツールです。ワークフローの定義はPythonをベースにしたDSLで記述し、その中でTFXのコンポーネントを活用する事ができます。また、ワークフローごとに異なる設定をして実験(Experiment)を実施したログが残せたり、ワークフローがちゃんと動いているかモニタリングができるようになっていたりと、機会学習システムのモデリング以外に必要な機能が整備されています。ワークフローマネジメント自体はKubeflowのCoreComponentである、Argoが動いているらしいですが、UIが整ったことでやっと統一感があるpipeline管理ツールが出てきたなというところです。


# Kubeflow Pipelines examples
今回はkubeflowの[slack](https://kubeflow.slack.com)で紹介されていたこの[examples](https://github.com/amygdala/code-snippets/tree/master/ml/kubeflow-pipelines)を試してみます。GKEを使ってBigQueryのChicago taxiのデータを用いて、TFXのコンポーネントを使って機械学習のワークフローを定義し、サービングまで機械学習モデルをデプロイしていく良いチュートリアルです。

基本的には[README.md](https://github.com/amygdala/code-snippets/blob/master/ml/kubeflow-pipelines/README.md)に書かれている通りに動かしますが、そのままではエラーが出る部分などあるのでworkaroundも示しながら進めたいと思います。

1. GCP環境のセットアップ
2. Kubernetes Engine (GKE) クラスタの準備
3. Kubeflow Pipelinesのインストール
4. Examplesを試す

# GCP環境のセットアップ
まずはGCPの環境を整えます。大まかな手順としては下記です。
- GCPプロジェクトを作る
- 必要なAPIをenableにする
  - Cloud Machine Learning Engine、Kubernetes Engine、オプションでTFTやTFMAをDataflow上で動かしたり、データソースをcsvファイルからBigQueryに変えるなどする場合はそれぞれEnableする必要があります
- gcloud sdkをインストールする、もしくはcloud shellを使う
- GCSのバケットを用意する
  - Backet名はXXXにしてあります。

# Kubernetes Engine (GKE) クラスタの準備
この[README.md](https://github.com/amygdala/code-snippets/blob/master/ml/kubeflow-pipelines/README.md#set-up-a-kubernetes-engine-gke-cluster)にGKEクラスタを作成します。

```
gcloud beta container --project <PROJECT_NAME> clusters create "kubeflow-pipelines" --zone "us-central1-a" --username "admin" --cluster-version "1.9.7-gke.11" --machine-type "custom-8-40960" --image-type "COS" --disk-type "pd-standard" --disk-size "100" --scopes "https://www.googleapis.com/auth/cloud-platform" --num-nodes "4" --enable-cloud-logging --enable-cloud-monitoring --no-enable-ip-alias --network "projects/<PROJECT_NAME>/global/networks/default" --subnetwork "projects/<PROJECT_NAME>/regions/us-central1/subnetworks/default" --addons HorizontalPodAutoscaling,HttpLoadBalancing,KubernetesDashboard --enable-autoupgrade --enable-autorepair
```

<PROJECT_NAME>には使っているGCPのプロジェクト名を入れて下さい。次に作ったクラスタをコンテキストに割り当て、ClusterRoleリソースを作成します。
gcloud container clusters get-credentials kubeflow-pipelines --zone us-central1-a --project mlops-215604

```
> gcloud container clusters get-credentials kubeflow-pipelines --zone us-central1-a --project <PROJECT_NAME>
Fetching cluster endpoint and auth data.
kubeconfig entry generated for kubeflow-pipelines.
> kubectl create clusterrolebinding ml-pipeline-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)
clusterrolebinding.rbac.authorization.k8s.io "ml-pipeline-admin-binding" created
> kubectl create clusterrolebinding sa-admin --clusterrole=cluster-admin --serviceaccount=kubeflow:pipeline-runner
clusterrolebinding.rbac.authorization.k8s.io "sa-admin" created
```


# Kubeflow Pipelinesのインストール
Kubeflowの[このページ](https://www.kubeflow.org/docs/guides/pipelines/deploy-pipelines-service/)の中のDeploy Kubeflow Pipelinesに従います。

Kubeflow PipelinesをGKEクラスタにデプロイします。

```
> PIPELINE_VERSION=0.1.2
> kubectl create -f https://storage.googleapis.com/ml-pipeline/release/$PIPELINE_VERSION/bootstrapper.yaml
clusterrole.rbac.authorization.k8s.io "mlpipeline-deploy-admin" created
clusterrolebinding.rbac.authorization.k8s.io "mlpipeline-admin-role-binding" created
job.batch "deploy-ml-pipeline-qqk9j" created
 > kubectl get job
NAME                       DESIRED   SUCCESSFUL   AGE
deploy-ml-pipeline-qqk9j   1         1            7m
> kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.7.240.1   <none>        443/TCP   18m
```

Kubeflow Pipelines UIにローカルのブラウザからGKE上のpod内のコンテナにアクセスできるようにポートフォワードの設定をしておきます。

```
> export NAMESPACE=kubeflow
> kubectl port-forward -n ${NAMESPACE} $(kubectl get pods -n ${NAMESPACE} --selector=service=ambassador -o jsonpath='{.items[0].metadata.name}') 8080:80

```

この状態でCloud shellから"Web Preview"するとKubeflowのとても簡素なダッシュボードに飛びます。

- スクショ


またそのURLの末尾に/pipelinesを追加することでKubeflow pipelinesのUIに移れます。

- スクショ


以上で[README.md](https://github.com/amygdala/code-snippets/blob/master/ml/kubeflow-pipelines/README.md)に記載されているExamplesのための準備は終わりました。しかし、これ以降のExamplesを動かすためにもう少し準備をします。Examplesを完走するためには、下記2点が必要です。
- Jupyter notebookへ設定を追加する
- 必要なJupyter extensionをインストールする

# Jupyter notebookへ設定を追加する
GKE上にjupyter notebookのサービス(?)は立ち上がるのですが、新しいnotebookを起動できません。しかしこの[issue](https://github.com/kubeflow/pipelines/issues/179)を参考にしてFixすることができました。

まずはjupyter hubからイメージを選択し、Spawnします。
今回は`gcr.io/kubeflow-images-public/tensorflow-1.10.1-notebook-cpu:v0.3.1`を選択しました。

立ち上がったら、Jupyter notebookのPodに入り、jupyterのコンフィグファイルに設定を追記します。

```
# Jupyter pod nameを調べます `jupyter-<USER>` (Here user is 'admin')
> kubectl get pods -n kubeflow

> kubectl exec -it -n kubeflow jupyter-<USER> bash

# 設定ファイルを変更します。
> jovyan@jupyter-admin:~$ vim .jupyter/jupyter_notebook_config.py
```
スクショ

上記のように`c.NotebookApp.allow_origin='*'`を追記します。そしてPodを再起動。
```
jovyan@jupyter-admin:~$ ps -auxw
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
jovyan       1  0.0  0.0   4528   820 ?        Ss   12:44   0:00 tini -- start-singleuser.sh --ip="0.0.0.0" --port=8888 --allow-root
jovyan       5  0.1  0.8 292128 62168 ?        Sl   12:44   0:01 /opt/conda/bin/python /opt/conda/bin/jupyterhub-singleuser --ip="0.0.0.0" --port=8888 --allow-root
jovyan      33  0.0  0.0  20316  4108 ?        Ss   12:52   0:00 bash
jovyan      41  0.0  0.0  36080  3300 ?        R+   12:53   0:00 ps -auxw

jovyan@jupyter-admin:~$ kill 1
jovyan@jupyter-admin:~$ command terminated with exit code 137

<USER>@CloudShell:~$ kubectl get pods  -n kubeflow |grep jupyter
jupyter-admin                                            1/1       Running   2          16m

export NAMESPACE=kubeflow
kubectl port-forward -n ${NAMESPACE} $(kubectl get pods -n ${NAMESPACE} --selector=service=ambassador -o jsonpath='{.items[0].metadata.name}') 8080:80
```

これでnotebookを立ち上げることができるようになりました。

# Juptyter notebookでTFMAを可視化するための extensionのインストール
TFMAはインタラクティブにデータをスライスし、その結果をjupyter notebook上で可視化することができますが、そのためにExtensionをインストールする必要があります。

```
# --system を付けた方が良いかも
> jupyter nbextension enable --py widgetsnbextension
> jupyter nbextension install --py --symlink tensorflow_model_analysis
> jupyter nbextension enable --py tensorflow_model_analysis
```
本来であればExtensionが無事にインストール&Enabledされるはずなのですが、tensorflow_model_analysisのインストール時にエラーが出てしまい、結局TFMAのレンダリングがされないままでした。解決しましたら追記します！

# Running the examples
Kubeflow Pipelineに機械学習のPipelineを定義していきます。Kubeflow pipelinesのUIに入るとすでにいくつかサンプルのPipelineが定義されています。

スクショ


ワークフローは[ここ](https://www.kubeflow.org/docs/guides/pipelines/build-pipeline/#compile-the-samples)にあるように、DSLで書かれた.pyファイルをコンパイルして、Kubeflow PipelinesのUIにアップロードすることでデプロイできます。
現在、サンプルとして挙げられているものはそれぞれ、[ここのSamples](https://github.com/kubeflow/pipelines/tree/master/samples)にあげられているもののようです。特にML-TFXはExample pipeline that does classification with model analysis based on a public tax cab BigQuery dataset.とあるように、workflowと基本的には同じっぽいです。(ちなみに、試しにやってみたら途中でエラー吐くので諦めました)。


## workflow1をやってみる
ここではすでにUI上にあるPipelineではなくて、新しくPipelineをアップロードして実行するようです。まずはDSLで書かれたスクリプトをコンパイルします。手順は[こちら](https://github.com/amygdala/code-snippets/blob/master/ml/kubeflow-pipelines/samples/kubeflow-tf/README.md#example-workflow-1)です。

```
> cd ~/code-snippets/ml/kubeflow-pipelines/samples/kubeflow-tf
> python3 workflow1.py
> ls
README.md  workflow1.py  workflow1.py.tar.gz  workflow2.py
```
Kubeflow pipelines UIにこの`workflow1.py.tar.gz`をアップロードするとpipelineができます。


スクショ 

UIからExperimentsを設定し、Runさせます。
このとき`preprocess-mode`と`tfma-mode`を`local`で実行していますが、ここを'cloud'にするとDataflowで動作します。

スクショ

この後ワークフローが走ります。UI眺めているだけではあまり実感無いですが、CSVにあったデータがTFTで処理され機械学習モデルに学習され、MLEngineにデプロイされてサービングされてます。

## TensorFlow Model Analysisでモデル解析
ワークフローが走り終わるとJupyterNotebook上で学習されたモデルについての解析ができます。JupyterHubにサインインします。IDもパスワードはなんでも入れますが、GCPのアカウントを使うようにしました。解析にはこの[tfma_expers.ipynb](https://github.com/amygdala/code-snippets/blob/master/ml/kubeflow-pipelines/components/dataflow/tfma/tfma_expers.ipynb)を使います。`OUTPUT_PATH_PREFIX = 'gs://<YOUR_BUCKET_PATH>/<WORKFLOW_NAME>/'`の<YOUR_BUCKET_PATH>はクラスタ立てるときに指定したバケット名、<WORKFLOW_NAME>は下記コマンド、もしくはCloud ConsoleでGoogle Strageの当該バケットを見に行くとディレクトリができているのでそれを使います。
```
kubectl -n ${NAMESPACE} describe pods jupyter-taketoshi-2ekazusa-40brainpad-2eco-2ejp
kubectl -n ${NAMESPACE} describe pods jupyter-<USER>
```

現状、レンダリングがうまくいっていないので全く意味をなしてませんが、一応実行できているようです。

JupyterNotebookのスクショ

## the TF-serving endpointsを使ってみる
今回のワークフローでは学習させたモデルがTF servingでサービングされています。
クラウドシェルからでいけるかな？

```
> kubectl get services


> python chicago_taxi_client.py \
>  --num_examples=1 \
>  --examples_file='../taxi_model/data/eval/data.csv' \
>  --server=<EXTERNAL IP>:9000 --model_name=<SERVICE NAME>
```

# まとめ
モデルやデータの管理など、まだまだ欲しい機能は出揃っていない感じですが素晴らしいツールだなと感じています。機械学習モデリングの後、実運用まで持っていくための試行錯誤はまだまだ続きそうですが、KubeflowやTFXなど便利なツールが普及して知見が貯まり、そのハードルがどんどん下がっていけば良いなと思いました。

  
#　以下下書き。




## Building workflows using kubeflow pipelines 

- https://medium.com/tensorflow/introducing-tensorflow-data-validation-data-understanding-validation-and-monitoring-at-scale-d38e3952c2f0
- https://github.com/tensorflow/data-validation


- https://chinagdg.org/2018/11/getting-started-with-kubeflow-pipelines/

## エクステンション問題

```
> jupyter nbextension list
Known nbextensions:
  config dir: /home/jovyan/.jupyter/nbconfig
    notebook section
      tfma_widget_js/extension  enabled 
      - Validating: problems found:
        - require?  X tfma_widget_js/extension
  config dir: /opt/conda/etc/jupyter/nbconfig
    notebook section
      jupyter-js-widgets/extension  enabled 
      - Validating: OK
  config dir: /usr/local/etc/jupyter/nbconfig
    notebook section
      jupyter-js-widgets/extension  enabled 
      - Validating: OK
```


https://8080-dot-3326024-dot-devshell.appspot.com/pipeline/#/pipelines





gs://bp-kubeflow-pipelines//workflow-1-wnrmr/




# 参考リンク
https://www.kubeflow.org/docs/guides/components/jupyter/


https://github.com/kubeflow/pipelines/issues/179
https://github.com/kubeflow/kubeflow/issues/1130

