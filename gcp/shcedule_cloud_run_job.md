# Cloud ShedulerとCloud Run Jobを連携させて定期的にジョブを実行させる方法

## 参考

- [Cloud Run Jobsを使用してジョブをスケジュール実行する](https://tech.rhythm-corp.com/schedule-jobs-to-run-using-cloud-run-jobs/)

## 使用するGCPサービスの概説

1. Cloud Scheduler

    設定したスケジュールに従って定期的にHTTPサーバへのアクセス、Pub/Subのトピックへのメッセージを送信する機能を提供。  
    スケジュールの設定はPOSIX(Unix)のcron形式で記述できる。

2. Cloud Run Job

    コンテナイメージ(Docker イメージ)化した処理(ジョブ)をコンテナで実行する機能を提供。  
    必要に応じてスケーリングするなど便利な機能を提供してくれる。  
    コンテナを実行するノード(サーバ)はデフォルトだとGoogleが管理しているノードプールを使うらしく、実行した分だけしか料金がかからない。  
    サーバーレス機能に分類される。  

    Cloud Runには、Cloud Run Serviceというものもあり、こちらはコンテナ上で動作するHTTPサーバをホスティングする機能。  
    Cloud Run Serviceのサービスを作成して有効化すると、そのサービスにアクセスするためのURLが発行され、そのURLにアクセスするとその処理はコンテナ上で動作するHTTPサーバで実行される。  
    URLにアクセスしないとコンテナが停止されるらしく（どのぐらいアクセスされないと停止するのかは不明、本当に停止するの？）、コンテナが立ち上がっていなければ料金がかからない。  
    処理量に合わせてスケーリングしてくれるらしい。  
    コンテナを実行するノード(サーバ)はデフォルトだとGoogleが管理しているノードプールを使うらしい。

    コンテナの実行環境としては自身のプロジェクト上のGKEクラスタも指定できる。  
    GPUが使えるノード(仮想サーバ)を使用したい場合には自身のプロジェクト上のGKEにGPUが有効なノードプール(GPUが有効な仮想サーバ)を作成してそこで動作させるよう設定する必要がある。  
    GKEのノードプール内のノードは常に起動しているわけではなく、負荷に応じて停止するので（設定に依存する部分もあるが）、費用が節約できる。  
    ただし、GKEではクラスタ(内部に複数のノードプールを持つことができる)の管理費用、GKEのサービス提供用ノード(仮想サーバ)用の費用は処理を実行していなくてもかかってくるため注意が必要。

> 補足
>
> Cloud Runで実行先を自身のGKE上にする場合、GKE(内部的にはKubenetes)上にサーバレス環境を提供するためのKnativeというコンポーネントが追加される。  
> Knativeとは、「サーバーレスのクラウドネイティブ・アプリケーションをデプロイ、実行、管理するためのコンポーネント」らしいが、要は内部的にKubenetesが提供する各種機能を使って簡単にサーバレス環境を構築するためのKubenetesの拡張コンポーネントかなと思う。（ちゃんと調べていない）

## 設定手順

| 変数名 | 意味 | 補足 |
|:---|:---|:---|
|PROJECT_ID|プロジェクト名||
|PROJECT_NUMBER|プロジェクト番号||
|REGION_NAME|リージョン名||
|SERVICE_ACCOUNT_FOR_CLOUD_RUN_JOB_INVOKE|Cloud Run JobのジョブをRest APIで起動するために使用するサービスアカウント||
|JOB_NAME|ジョブ名||
|DOCKER_IMAGE_NAME|Dockerイメージ名||

1. `${REGION_NAME}`に`${JOB_NAME}`という名前でCloud Run Jobのジョブを作成する。

    1. イメージを柵瀬資するためのDockerfileを作成する。

       ```
       FROM google/cloud-sdk:latest

       ENV APP_HOME /app
       COPY script.sh $APP_HOME/
       RUN chmod +x $APP_HOME/script.sh

       CMD $APP_HOME/script.sh
       ```

    2. ジョブ実行用のコンテナイメージを作成して、それをコンテナイメージを格納するコンテナレジストリに登録する。

        ```sh
        gcloud builds submit --tag gcr.io/${PROJECT_ID}/${DOCKER_IMAGE_NAME}:latest
        ```

    3. Cloud Run Jobのジョブを作成する。

       ```sh
       gcloud beta run jobs create ${JOB_NAME} \
       --image gcr.io/${PROJECT_ID}/${DOCKER_IMAGE_NAME} \
       --region ${REGION_NAME}
       ```

2. Cloud Run JobのジョブをRest APIで起動するために使用するサービスアカウントに `roles/run.invoker` 権限を追加する。

    ```sh
   gcloud projects add-iam-policy-binding ${PROJECT_ID} \
   --member="serviceAccount:${SERVICE_ACCOUNT_FOR_CLOUD_RUN_JOB_INVOKE}" \
   --role=roles/run.invoker
   ```

3. Cloud Shedulerの設定を以下のようにする。

    |設定|値|補足|
    |:---|:---|:---|
    |ターゲットタイプ|HTTP||
    |URL|https://${REGION_NAME}-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/${PROJECT_NUMBER}/jobs/${JOB_NAME}:run||
    |HTTPメソッド|POST||
    |Authヘッダー|OAuth トークンを追加||
    |サービスアカウント|${SERVICE_ACCOUNT_FOR_CLOUD_RUN_JOB_INVOKE}||
    |範囲|https://www.googleapis.com/auth/cloud-platform|Cloud Schedulerのデフォルトのスコープらしいが、意味はよく分からない。とりあえず動いたのでこれでよいのだと思う。|
