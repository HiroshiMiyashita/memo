# Cloud ShedulerとCloud Run Jobを連携させて定期的にジョブを実行させる方法

## 概要

GCP(Google Cloud)上でCloud ShedulerとCloud Run Jobを利用して定期的に処理を実行させるための方法について記述。  

> 手順はMarkdownとして記述。

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

1. `${REGION_NAME}`に`${JOB_NAME}`という名前でCloud Run Jobのジョブを作成

    1. イメージを作成するためのDockerfileを作成

       ```
       FROM google/cloud-sdk:latest

       ENV APP_HOME /app
       COPY script.sh $APP_HOME/
       RUN chmod +x $APP_HOME/script.sh

       CMD $APP_HOME/script.sh
       ```

    2. ジョブ実行用のコンテナイメージを作成して、それをコンテナイメージを格納するコンテナレジストリに登録

        ```sh
        gcloud builds submit --tag gcr.io/${PROJECT_ID}/${DOCKER_IMAGE_NAME}:latest
        ```

        > 参考
        > 
        > [Gcloud コマンドラインリファレンス（gcloud builds submit）](https://cloud.google.com/sdk/gcloud/reference/builds/submit)

        > 注意
        > 
        > `${DOCKER_IMAGE_NAME}`に大文字が含まれているとビルドエラーとなる。

    3. Cloud Run Jobのジョブを作成

       ```sh
       gcloud beta run jobs create ${JOB_NAME} \
       --image gcr.io/${PROJECT_ID}/${DOCKER_IMAGE_NAME} \
       --region ${REGION_NAME}
       ```

       > 参考
       > 
       > [Gcloud コマンドラインリファレンス（gcloud beta run jobs create）](https://cloud.google.com/sdk/gcloud/reference/beta/run/jobs/create) 

2. Cloud Run JobのジョブをRest APIで起動するために使用するサービスアカウントに `roles/run.invoker` 権限を追加

    ```sh
   gcloud projects add-iam-policy-binding ${PROJECT_ID} \
   --member="serviceAccount:${SERVICE_ACCOUNT_FOR_CLOUD_RUN_JOB_INVOKE}" \
   --role=roles/run.invoker
   ```

   > 参考
   > 
   > [Gcloud コマンドラインリファレンス（gcloud projects add-iam-policy-binding）](https://cloud.google.com/sdk/gcloud/reference/projects/add-iam-policy-binding) 

3. Cloud Shedulerを以下のように設定

    |設定|値|補足|
    |:---|:---|:---|
    |スケジュール|"0 */3 * * *"|3時間ごとに実施する場合|
    |タイムゾーン|Asia/Tokyo|日本時刻を使用する場合|
    |ターゲットタイプ|HTTP||
    |URL|https://${REGION_NAME}-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/${PROJECT_NUMBER}/jobs/${JOB_NAME}:run||
    |HTTPメソッド|POST||
    |Authヘッダー|OAuth トークンを追加||
    |サービスアカウント|${SERVICE_ACCOUNT_FOR_CLOUD_RUN_JOB_INVOKE}||
    |範囲|https://www.googleapis.com/auth/cloud-platform|Cloud Schedulerのデフォルトのスコープらしいが、意味はよく分からない。とりあえず動いたのでこれでよいのだと思う。|

    ```sh
    gcloud scheduler jobs create http ${JOB_NAME}-scheduler \
    --schedule="0 */3 * * *" \
    --time-zone="Asia/Tokyo" \
    --uri="https://${REGION_NAME}-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/${PROJECT_NUMBER}/jobs/${JOB_NAME}:run" \
    --http-method=POST \
    --oauth-service-account-email="${SERVICE_ACCOUNT_FOR_CLOUD_RUN_JOB_INVOKE}" \
    --oauth-token-scope="https://www.googleapis.com/auth/cloud-platform"
    ```

    > 参考
    > 
    > [Gcloud コマンドラインリファレンス（gcloud scheduler jobs create http）](https://cloud.google.com/sdk/gcloud/reference/scheduler/jobs/create/http)  
    > [cron ジョブ スケジュールの構成](https://cloud.google.com/scheduler/docs/configuring/cron-job-schedules)
