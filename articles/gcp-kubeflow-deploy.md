---
title: "Kubeflowお試し"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gcp", "kubeflow", "gke", "anthos"]
published: false
---

以下を一通りやってみる
https://www.kubeflow.org/docs/gke/deploy/

# Set up Google Cloud Project

Cloud Shell での作業 (free tier ではなく paid account で実施)

```sh
gcloud config set project <YOUR PROJECT NAME>
gcloud config set compute/region asia-northeast1
gcloud services enable compute.googleapis.com container.googleapis.com iam.googleapis.com servicemanagement.googleapis.com cloudresourcemanager.googleapis.com ml.googleapis.com meshconfig.googleapis.com
```

Anthos サービスメッシュを初期化する工程を行う (https://cloud.google.com/service-mesh/docs/archive/1.4/docs/gke-install-new-cluster#setting_credentials_and_permissions)．

```sh
curl --request POST --header "Authorization: Bearer $(gcloud auth print-access-token)" --data '' https://meshconfig.googleapis.com/v1alpha1/projects/${GOOGLE_CLOUD_PROJECT}:initialize
```

Identity Pool does not exist っていわれた．どうやら temporary な GKE がいるらしい．
https://github.com/kubeflow/website/issues/2121
先述のドキュメントを参考にして最低限のリソースで temporary な GKE を用意する (前提条件は 2 つ，vcpu が４つ以上のマシンタイプの 4 ノード以上の構成と wrorkload-pool 設定)．

```sh
gcloud container clusters create temp-gke --machine-type=e2-standard-4 --num-nodes=4 --enable-ip-alias --release-channel regular --workload-pool=${GOOGLE_CLOUD_PROJECT}.svc.id.goog --labels=mesh_id="proj-$(gcloud projects describe ${GOOGLE_CLOUD_PROJECT} --format="value(projectNumber)")"
```

CPUS_ALL_REGIONS が 32 で 48 が必要だったので以下のように申請した．
https://cloud.google.com/compute/quotas#requesting_additional_quota
asia-northeast1 の CPUS が 24 で 48 が必要だったので同じように申請した (上の CPUS_ALL_REGIONS もこれで更新されるので申請はこれに集約できた)．
asia-northeast1 の IN_USE_ADDRESSES が 8 で 12 が必要だったので同じように申請した．
上記コマンド再実行した．

```sh
gcloud container clusters create temp-gke --machine-type=e2-standard-4 --num-nodes=4 --enable-ip-alias --release-channel regular --workload-pool=${GOOGLE_CLOUD_PROJECT}.svc.id.goog --labels=mesh_id="proj-$(gcloud projects describe ${GOOGLE_CLOUD_PROJECT} --format="value(projectNumber)")"
# 上記申請で結構でかいクラスタだなと思ったら regional の gke だったので各 zone で--num-nodes=4 が必要で asia-northeast1 は 3zones があるため 12 ノードのクラスタになっていたためだった
# 一応 zone 設定も書いておく
gcloud config set compute/zone asia-northeast1-b
# パブリックに出ている GKE の認証を一応確認しておく (上の zone 設定をしたので region を指定しないと zone で探してしまうので注意)
gcloud container clusters get-credentials temp-gke --region asia-northeast1
# ちゃんとうごく
kubectl get nodes
```

ついでに k8s の認証周りを確認する．

```sh
# 設定を確認
kubectl config view
```

auth-provider に gcloud コマンドが設定されているので自分のメールアドレスで k8s では認証されていそう．
https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl#authentication
上記を読むと，gcloud container clusters create した時点で kubeconfig に書き込まれているらしい (つまり，上記の gcloud container clusters get-credentials はいらなかった)．

```sh
# k8s 側での設定を確認する
kubectl get clusterrolebinding cluster-admin -o json
```

system:masters という group に cluster-admin がバインドされている．
system:からはじまるグループはデフォルトで検出されるシステムのようだった．基本的には IAM で管理されるのもシステム側で IAM に権限が振られていたら特定のグループに入れるような形で実現していそう．
以下はそれを拡張したサービスと思われる Google グループへの k8s での権限付与の説明
https://cloud.google.com/kubernetes-engine/docs/how-to/role-based-access-control#google-groups-for-gke

改めて，Anthos Service Mesh の初期化を行う ({}が返ってきたら成功)．

```sh
curl --request POST --header "Authorization: Bearer $(gcloud auth print-access-token)" --data '' https://meshconfig.googleapis.com/v1alpha1/projects/${GOOGLE_CLOUD_PROJECT}:initialize
```

ここら辺の処理は結構面倒なので，最新のハンズオンにもあるように install_asm スクリプトで実行した方がよさそう (今回は kubeflow のインストール手順に従っている)．
https://medium.com/google-cloud-jp/implementing-anthos-service-mesh-3d8205bf48ed

```sh
# temp-gke クラスタの削除
gcloud container clusters delete temp-gke --region=asia-northeast1
```

ちなみに，Anthos Service Mesh と Anthos Config Management の一部機能は 2020 年 11 月と 2020 年 2 月から Anthos ライセンスとは別で提供されるようになったようだ．
https://cloud.google.com/service-mesh/docs/pricing?hl=ja
https://cloud.google.com/service-mesh/docs/archive/1.6/docs/release-notes#November_12_2020
Anthos Config Management は明確に Anthos ライセンスがいらないと書いていないのでよくわからないが動画で Config Management も GCP 内であれば無料であると述べられていた．
https://youtu.be/6p5IqnqHiAE?t=840

# Set up OAuth for Cloud IAP

以下にアクセスして OAuth 同意画面の設定をする (おそらくこのプロジェクトで共通の OAuth 2.0 の窓口となるので設定してある場合は必要ない)
https://console.cloud.google.com/apis/credentials/consent

```text
User Type: 外部
アプリ名: Kubeflow
ユーザサポートメール: 自分のメールアドレス
デベロッパーの連絡先: 自分のメールアドレス
承認済みドメイン: 空欄
scope: 設定しない
テストユーザー: 自分だけ設定
```

以下で認証情報の作成
https://console.cloud.google.com/apis/credentials
OAuth クライアント ID の作成を行う

```text
アプリケーションの種類: ウェブアプリケーション
アプリケーションの名前: ウェブ クライアント kubeflow
承認済みの JavaScript 生成元: 設定しない
承認済みドメイン: https://iap.googleapis.com/v1/oauth/clientIds/<CLIENT_ID>:handleRedirect
```

CLIENT_ID は XXX.apps.googleusercontent.com の形式の ID

# Management cluster set up

Cloud Shell での作業

```sh
# yq のインストール
GO111MODULE=on go get github.com/mikefarah/yq/v3
yq --version
# projectとzoneの設定
gcloud config set project <YOUR PROJECT NAME>
gcloud config set compute/zone asia-northeast1-b
# management clusterの設定
MGMT_PROJECT=$GOOGLE_CLOUD_PROJECT
MGMT_DIR=~/kf-deployments-kubeflow/management
MGMT_NAME="${GOOGLE_CLOUD_PROJECT}"
LOCATION=asia-northeast1-b
mkdir -p $MGMT_DIR
# management clusterのコードを取得
# 以下コマンドで failed to set とでるが後で設定するのでいったん無視 (後で設定するので)
kpt pkg get https://github.com/kubeflow/gcp-blueprints.git/management@v1.2.0 ${MGMT_DIR}
cd $MGMT_DIR
make get-pkg
# 以下で instance/Kptfile と upstream/management/Kptfile に設定が入る (実行ディレクトリ以下の設定の全リストを得るには `kpt cfg list-setters .` を実行する)
kpt cfg set -R . name "${MGMT_NAME}"
kpt cfg set -R . gcloud.core.project "${MGMT_PROJECT}"
kpt cfg set -R . location "${LOCATION}"
# 以下コマンドで設定検証できる
make hydrate-cluster
# 一旦 kubeconfig がないことを確認する
kubectl config get-contexts
# クラスタを作成する
make apply-cluster
# gcloud container create での context が入っている
kubectl config get-contexts
# チュートリアルに従うために以下を実行する (コンテキストの名前書き換えと namespace の設定を行っていそう)
make create-context
# kubeconfig の内容を確認 (さっきと比べて，name と namespace が書き換わっているはず)
kubectl config get-contexts
# ここから Config Connector のインストール
# 以下コマンドで設定検証できる
make hydrate-kcc
# インストール実行 (サービスアカウント${MGMT_NAME}-cnrm-system@${MGMT_PROJECT}.iam.gserviceaccount.com 作成と Config Connector インストールを行う)
make apply-kcc
# 管理対象のプロジェクトでの権限を付与する．
# 今回は managed project は同じプロジェクトでやる
MANAGED_PROJECT=$GOOGLE_CLOUD_PROJECT
# いったん IAM 確認 (*-cnrm-system@*みたいなサービスアカウントがないことを確認)
gcloud projects get-iam-policy $MANAGED_PROJECT
# Kptfileに管理対象のプロジェクト (managed project)を設定する
kpt cfg set ./instance managed-project "${MANAGED_PROJECT}"
# サービスアカウントの作成と設定を行う
gcloud beta anthos apply ./instance/managed-project/iam.yaml
gcloud projects get-iam-policy $MANAGED_PROJECT
```

ここら辺の手順が kubeflow のリポジトリに依存しており微妙な気がしてきた．
make の中で実行されている anthoscli は gcloud の中に入ってる気がする... (手順が古くなっている可能性がある)
公式の ASM や Config Connector のインストール方法に従った方が良いかもしれない．
結論としては，management cluster とは要するに Config Connector が設定されたクラスタ (適切な権限設定も含む)があればよさそう．

# Deploy using kubectl and kpt

```sh
gcloud config set project <YOUR PROJECT NAME>
# Kubeflow Pipeline はリージョナルクラスタでうまく動かないらしいのでzoneクラスタで行う
# https://github.com/kubeflow/gcp-blueprints/issues/6
# あえてmanagement clusterと違うリージョンにデプロイしてみる
gcloud config set compute/zone asia-northeast1-c
# kubeflowとmanagement clusterの設定をいれる
KF_NAME=dousu-kubeflow-test
KF_PROJECT="${GOOGLE_CLOUD_PROJECT}"
KF_DIR=~/kf-deployments/${KF_NAME}
MGMT_NAME="${GOOGLE_CLOUD_PROJECT}"
MGMTCTXT="${MGMT_NAME}"
mkdir -p $KF_DIR
kpt pkg get https://github.com/kubeflow/gcp-blueprints.git/kubeflow@v1.2.0 `dirname "${KF_DIR}"`
cd "${KF_DIR}"
make get-pkg
# kpt の変数確認
kpt cfg list-setters .
# NVIDIA Tesla K80 が使えるか調べる (参照: https://cloud.google.com/compute/docs/gpus )
# GPUは N1 汎用タイプでだけ使えるので注意
gcloud compute accelerator-types list
# T4 しか使えなさそうだったのでtesla-k80で検索して該当場所をt4に置換 (これを設定せずmake applyした場合は一旦make delete-gcpで作り直す)
sed "s/nvidia-tesla-k80/nvidia-tesla-t4/" upstream/manifests/gcp/v2/cnrm/cluster/cluster.yaml
# Makefile の set-values で<hoge>となっている部分を環境変数に合わせて書く
# kubectl の設定 (namespaceのデフォルト設定をしていないといけないらしい)および management cluster で namespace を作成する
kubectl config use-context "${MGMTCTXT}"
kubectl create namespace "${KF_PROJECT}"
kubectl config set-context --current --namespace "${KF_PROJECT}"
# IAP の認証情報を入力する
# 先述のステップで設定済みなので以下で取得
# https://console.cloud.google.com/apis/credentials
export CLIENT_ID=<Your CLIENT_ID>
export CLIENT_SECRET=<Your CLIENT_SECRET>
# Kptfile に変数をセット
make set-values
# Kubeflow をデプロイ
make apply
# "unknown field "env" in v1alpha1.ProxyConfig"というエラーが出た．ASM は istioctl のバージョンが違うらしい
# https://github.com/kubeflow/manifests/issues/1490
# istioctl をインストール
mkdir ~/asm-istio; cd ~/asm-istio
# ASM のサービスアカウントがあることを確認 (先述のASM初期化ステップで作られているはず)
gcloud projects get-iam-policy ${PROJECT_ID} | grep -B 1 'roles/meshdataplane.serviceAgent'
curl -LO https://storage.googleapis.com/gke-release/asm/istio-1.4.10-asm.18-linux.tar.gz
tar xzf istio-1.4.10-asm.18-linux.tar.gz
cd istio-\*/
export PATH=$PWD/bin:$PATH
# バージョンが*-asm.*になっていることを確認する
istioctl version
# kubeflow のデプロイを再実行
cd "${KF_DIR}"
# webhook.cert-manager.io が unavailable だったり，Application の kind がなくてエラーが出たりした場合は make apply を再実行する (Image と Profile でも出たが再実行した)
make apply
# デプロイした kubeflow クラスタを確認
# コンテキストが変わっていることを確認
kubectl config get-contexts
kubectl -n kubeflow get all
# iap のアクセス権を付与
gcloud projects add-iam-policy-binding "${KF_PROJECT}" --member=user:<EMAIL> --role=roles/iap.httpsResourceAccessor
# kubeflow の ingress を確認
kubectl -n istio-system get ingress
# アクセス先を取得
export HOST=$(kubectl -n istio-system get ingress envoy-ingress -o=jsonpath={.spec.rules[0].host})
# 以下にブラウザでアクセスできる．
echo https://$HOST
# IAP で Google の認証に飛ぶので権限を与えた Email アドレスでアクセスする．
```

default-profile が表示されれば OK

試しに Nvidia GPU が 1 つの Jupyter Notebook Server を Web UI から作成してみると，GPU 付のノードプールが自動で作られるのを確認した．
そのあと，GPU ドライバが必要になるので以下を参考にインストールする．
https://cloud.google.com/kubernetes-engine/docs/how-to/gpus#installing_drivers

```sh
# kube-systemにdaemonsetがはいる
kubectl --context $KF_NAME apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/master/nvidia-driver-installer/cos/daemonset-preloaded.yaml
```

# Using Your Own Domain

Cloud Shell での作業

https://www.kubeflow.org/docs/gke/custom-domain/

# Enabling TPU and GPU

https://www.kubeflow.org/docs/gke/pipelines/enable-gpu-and-tpu/

# Pipelines on GCP

https://www.kubeflow.org/docs/gke/pipelines/

# Clean Up

```sh
KF_DIR=~/kf-deployments/${KF_NAME}
cd $KF_DIR
make delete-gcp
MGMT_DIR=~/kf-deployments-kubeflow/management
cd $MGMT_DIR
make delete-cluster
MANAGED_PROJECT=$GOOGLE_CLOUD_PROJECT
# serviceAccountの前にdeletedとついているのを確認
gcloud projects get-iam-policy $MANAGED_PROJECT
```
