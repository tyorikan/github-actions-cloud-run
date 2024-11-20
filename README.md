# GitHub Actions Cloud Run deploy

<walkthrough-project-setup>
</walkthrough-project-setup>

### 選択されたプロジェクト名を確認する
```bash
gcloud config get project
```
正しく設定されていれば、以下のように表示されます。
```
Your active configuration is: [cloudshell-17515]
spider124-111
```
もし正しく設定されていない (unset 表示されている) 場合、以下のコマンドを実行して下さい。 **[YOUR_PROJECT_ID]** はご自身のプロジェクト ID に置き換えてください。
```bash
gcloud config set project [YOUR_PROJECT_ID]
```

## 環境の準備
### シェル環境変数の設定
プロジェクトで繰り返し使用する値をシェルの環境変数に設定します。 **GITHUB_ACCOUNT** は自身の GitHub アカウント、または Organation ID に置き換えてください。  
**長時間の離席などにより Cloud Shell のセッションが切断された場合、必ずこの手順を再度実行して環境変数を設定し直して下さい。**
```bash
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects list --filter="$(gcloud config get-value project)" --format="value(PROJECT_NUMBER)")
export GITHUB_ACCOUNT=[YOUR_GITHUB_ACCOUNT]
export CR_DEPLOY_SA=cr-deployer
export CR_EXEC_SA=gh-actions-demo-service
export RUN_SERVICE=$CR_EXEC_SA
export PUBSUB_TOPIC=gh-actions-demo-topic
```

## API の有効化
ハンズオンで必要になるサービスの API を有効化します。
```bash
gcloud services enable run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  compute.googleapis.com \
  iam.googleapis.com \
  iamcredentials.googleapis.com \ 
  secretmanager.googleapis.com \
  cloudresourcemanager.googleapis.com \
  sts.googleapis.com
```

## IAM の準備
Cloud Run デプロイ用のサービスアカウントの作成
```bash
gcloud iam service-accounts create ${CR_DEPLOY_SA}
```
Cloud Run 実行用のサービスアカウントの作成
```bash
gcloud iam service-accounts create ${CR_EXEC_SA}
```

### Role の付与
GitHub Actions 実行で利用するサービス アカウントに **Cloud Run 管理者** と **Artifact Registry 書き込み** の権限を割り当てます。サービスアカウントも指定する場合、**サービス アカウント ユーザー**権限も追加。
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:${CR_DEPLOY_SA}@${PROJECT_ID}.iam.gserviceaccount.com --role=roles/run.admin
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:${CR_DEPLOY_SA}@${PROJECT_ID}.iam.gserviceaccount.com --role=roles/artifactregistry.writer
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:${CR_DEPLOY_SA}@${PROJECT_ID}.iam.gserviceaccount.com --role=roles/iam.serviceAccountUser
```

Cloud Run 実行ユーザとなるサービス アカウントに、必要に応じた権限を付与。
```bash
# e.g. Pub/Sub パブリッシャーと BigQuery データ編集者 Role を付与
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:${CR_EXEC_SA}@${PROJECT_ID}.iam.gserviceaccount.com --role=roles/pubsub.publisher
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:${CR_EXEC_SA}@${PROJECT_ID}.iam.gserviceaccount.com --role=roles/bigquery.dataEditor
```

### Artifact Registry リポジトリの作成
```bash
gcloud artifacts repositories create ${RUN_SERVICE} \
    --repository-format=docker \
    --location=asia-northeast1
```

## GitHub の準備
1. demo repository（このリポジトリ）を fork し、コードをローカルに clone します。
2. **[GitHub 側の設定]** Secret を設定します。
**Settings** -> **Secrets and variables** -> **Actions** を選択し、右ペインから **Variables** タブを選択します。
**Repository Variable** に下表の値を追加します。下表の"GCP_PROJECT_NUMBER" 及び "GCP_SA_ID" にはプロジェクト番号やプロジェクト名が自動的に入力されています。もし空欄になっている場合は手作業で追記して下さい。


| Name | Value | Note |
-------|--------|------ 
| PROJECT_ID | <walkthrough-project-id/> |数字|
| REGION | asia-northeast1 ||
| SERVICE | gh-actions-demo-service ||
| CLOUD_BUILD_SA_ID | cr-deployer@<walkthrough-project-id>.iam.gserviceaccount.com|"@"の後にプロジェクト ID が含まれているか|
| Cloud_RAN_SA_ID | gh-actions-demo-service@<walkthrough-project-id>.iam.gserviceaccount.com|"@"の後にプロジェクト ID が含まれているか|
| WORKLOAD_IDENTITY_PROVIDER | projects/<walkthrough-project-number>/locations/global/workloadIdentityPools/github-actions-pool/providers/github-actions-provider |"projects/"の後にプロジェクト番号が入る|

## Workload Idenitty 連携の準備
GitHub Actions で Cloud Build を呼び出すための GitHub Actions の設定を行います。
1. (**[Cloud Console](https://console.cloud.google.com) での操作**) **IAM** -> **Workload Identity 連携** へ移動し、プロバイダを追加します。下表の通り入力します。  

設定項目 | 値
--------|------
ID プール名|github-actions-pool
プロバイダ | OIDC
プロバイダ名|github-actions-provider
発行元|https://token.actions.githubusercontent.com
オーディエンス|デフォルト
属性のマッピング ((Google*)=(OIDC*))|google.subject=assertion.sub attribute.repository_owner=assertion.repository_owner
属性条件|assertion.repository_owner=='YOUR_GITHUB_ACCOUNT'

2. GitHub Actions から Cloud Build を呼び出すため、Cloud Build で利用するサービス アカウントに対し、Workload Identity ユーザーの権限を追加します。
```bash
gcloud iam service-accounts add-iam-policy-binding ${CR_DEPLOY_SA}@${PROJECT_ID}.iam.gserviceaccount.com \
    --role=roles/iam.workloadIdentityUser \
    --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github-actions-pool/attribute.repository_owner/${GITHUB_ACCOUNT}"
```

## 利用する Pub/Sub トピックを作成する
```bash
gcloud pubsub topics create ${PUBSUB_TOPIC}
```

## 試してみる
**main ブランチに何か変更をコミットしましょう**
