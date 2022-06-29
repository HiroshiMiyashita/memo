# Cloud Functionsを使用してfirestoreのバックアップを取得する方法

## 準備

1. バックアップ用のGoogle Cloud Strageのバケットをバックアップしたいfirestoreが動作しているGCPプロジェクト上に作成.

    > firebaseプロジェクトはGCPプロジェクトの一種で、firebase console以外にもGCPコンソールからもプロジェクトを操作できる.
    > firebase consoleからはCloud Strageが作れないと思うので、GCPコンソールから作業を行う.

    ここではバケットのURIを

    `gs://<BUCKET_NAME>`

    とする.

    > `<BUCKET_NAME>`の部分を実際のバケット名にすること.

2. Cloud Functionsを実行するアカウントにfirestoreに対するImport Exportを行うための権限を付与.

    > Cloud Functionsを実行するアカウントは`<PROJECT_ID>@appspot.gserviceaccount.com`というアカウントらしい.\
    > `<PROJECT_ID>`はGCPプロジェクトのIDを指定すること.

    Cloud Shellより以下のコマンドを実行する.

    ```sh
    gcloud projects add-iam-policy-binding <PROJECT_ID> \
    --member serviceAccount:<PROJECT_ID>@appspot.gserviceaccount.com \
    --role roles/datastore.importExportAdmin
    ```

    > Cloud ShellはGCPコンソールから起動することができる.

3. エクスポート先のバケットに対してCloud Functionsを実行するアカウントに`admin`権限を付与する.

    Cloud Shellより以下のコマンドを実行する.

    ```sh
    gsutil iam ch serviceAccount:<PROJECT_ID>@appspot.gserviceaccount.com:admin \
    gs://<BUCKET_NAME>
    ```

4. バックアップ用のバケットのライフサイクルルールを変更.

    バケットの設定で、オブジェクトのバージョニングの個数や、古いバージョンのものをどれだけの期間保持するかなどのライフサイクルルールを設定する.\
    設定値は、バックアップ要件に依存.

    > [データライフサイクルを管理するオプション](https://cloud.google.com/storage/docs/control-data-lifecycles?hl=ja)


## Cloud Functionsを作成

1. 以下のようなコードのCloud Functionsを作成してデプロイする.

    > 下記の`<BUCKET_NAME>の部分はバックアップ用のバケット名に変更する.

    ```ts
    import * as admin from "firebase-admin";
    import * as firestore from "@google-cloud/firestore";
    import * as functions from "firebase-functions";

    admin.initializeApp();
    const auth = admin.auth();
    const fsAdm = new firestore.v1.FirestoreAdminClient();

    export const dailyFirestoreBackup = functions.pubsub.schedule("every day 03:00").onRun(async () => {
      const projectId = process.env.GCP_PROJECT || process.env.GCLOUD_PROJECT;
      if (projectId === undefined) return;
      const dbName = fsAdm.databasePath(projectId, "(default)");

      try {
        const resp = await fsClt.exportDocuments({
          name: dbName,
          outputUriPrefix: "gs://<BUCKET_NAME>",
          collectionIds: [],
        });
        console.log(resp);
      } catch (err) {
        console.error(err);
        throw err;
      }
    });
    ```

    > 以下を参考にした.\
    > [Schedule data exports](https://firebase.google.com/docs/firestore/solutions/schedule-export)

    > 上記の`schedule()`の引数で指定している構文は、cronジョブスケジュールの構文とApp Engine Shedule構文が使えるらしい.\
    >
    > [cron ジョブ スケジュールの構成](https://cloud.google.com/scheduler/docs/configuring/cron-job-schedules?hl=ja)\
    > [App Engine Schedule構文](https://cloud.google.com/appengine/docs/standard/python/config/cronref?hl=ja#schedule_format)
