firebase-toolsがインストールされていること

1. firebaseでマルチサイトを作成する

    サイトIDは以下の2つとする

    - hoge-site-01
    - hoge-site-02

    > `firebase hosting:sites:list`
    > コマンドで確認可能

2. hoge-site-01用のアプリケーションをビルド

    1. ビルド時の出力先を変更する

        `flutter config --build-dir=build_01`

        > これにより、ビルドの結果を格納するフォルダが`build_01`となる

    2. ビルドする

        `flutter build web --target=lib/main_01.dart`

        > `hoge-site-01`にデプロイするアプリのエントリーポイントが`lib/main_01.dart`であることを前提とした場合

3. hoge-site-02用のアプリケーションをビルド

    1. ビルド時の出力先を変更する

        `flutter config --build-dir=build_02`

    2. ビルドする

        `flutter build web --target=lib/main_02.dart`

        > `hoge-site-02`にデプロイするアプリのエントリーポイントが`lib/main_02.dart`であることを前提とした場合

4. ターゲット名とサイトIDを紐づける

    以下のコマンドを実行してターゲット名とサイトIDを紐づける

    `firebase target:apply hosting site-01 hoge-site-01`
    `firebase target:apply hosting site-02 hoge-site-02`

    > このコマンドにより、ターゲットとそれに対するサイトの紐づけの情報が`.firebaserc`に記述される
    > コマンドでなくてもこのファイルをいじるだけでよい？

5. firebase.jsonの`hosting`に上記で設定したターゲットそれぞれに対する設定を記述する

    以下のサイトなどを参考にする

    [複数のサイトでプロジェクトのリソースを共有する](https://firebase.google.com/docs/hosting/multisites?hl=ja)

6. firebaseにデプロイする

    `firebase deploy`

    > hostingのみをdeployしたい場合には以下のコマンドを使用する
    >
    > `firebase deploy --only hosting`
    