# Organizationの移行方法

## 準備

1. 移行元、移行先の両方でorganizationに対してOwner権限を持っていることを確認する。

2. 移行元、移行先の両方でアクセストークンを取得する。

    アクセストークンの権限についてはフル？

## 手順

1. マイグレーション情報の表示

    ```sh
    curl \
    -H "Accept: application/vnd.github+json" \ 
    -H "Authorization: token ${TOKEN}" \
    https://api.github.com/orgs/${DEST_ORGANIZATION}/migration
    ```

2. 

    ```sh
    curl \
    -X POST \
    -H "Accept: application/vnd.github+json" \ 
    -H "Authorization: token ${TOKEN}" \
    https://api.github.com/orgs/${DEST_ORGANIZATION}/migrations \
    -d '{"repositories":["${SRC_ORGANIZATION}/${SRC_REPOSITORY}"], "lock_repositories":true}'
    ```
  