# Cloud ShedulerとPub/SubとCloud Run Jobを連携させて定期的にジョブを実行させる方法

## 概要

1. Cloud Schedulerで定期的にPub/Subの定期実行用Topicにメッセージを送信
2. Pub/Subの定期実行用のTopicのサブスクリプションで、Cloud Run Jobを実行するためのAPI(Httpでアクセスできるエンドポイント)を実行
3. 2.のAPIを受けてCloud Run Jobのジョブが輝度する

## 準備

1. Pub/Subのサブスクリプションで指定するサービスアカウントのトークンを取得するために使われるGoogleが管理するサービスアカウントに対して isam.serviceAccountTokenCreator ロールを付与する.

    ```sh
    gcloud projects add-iam-policy-binding ${PROJECT_NAME} \
    --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-pubsub.iam.gserviceaccount.com" \
    --role=roles/iam.serviceAccountTokenCreator
    ```

2. Pub/Subのサブスクリプションで指定するサービスアカウントに対してCloud Run Jobを開始できるよう roles/run.invoker ロールを付与する.

   ```sh
   gcloud projects add-iam-policy-binding ${PROJECT_NAME} \
   --member="serviceAccount:${SERVICE_ACCOUNT_FOR_CLOUD_RUN_JOB_INVOKE}" \
   --role=roles/run.invoker
   ```
