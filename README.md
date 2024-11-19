# GitHub Actions Cloud Run deploy

## Prerequisite

* Enable the Cloud Run and related APIs

```bash
gcloud services enable run.googleapis.com \
  cloudbuild.googleapis.com \
  iamcredentials.googleapis.com \
  artifactregistry.googleapis.com \
  compute.googleapis.com \
  iam.googleapis.com \
  iamcredentials.googleapis.com \
  sts.googleapis.com  \
  secretmanager.googleapis.com
```

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
```

## API の有効化
ハンズオンで必要になるサービスの API を有効化します。
```bash
gcloud services enable run.googleapis.com \
  cloudbuild.googleapis.com \
  iamcredentials.googleapis.com \
  artifactregistry.googleapis.com \
  compute.googleapis.com \
  iam.googleapis.com \
  iamcredentials.googleapis.com \ 
  sts.googleapis.com  \
  secretmanager.googleapis.com
```

## IAM の準備
本ハンズオンで使用するサービス アカウントを作成します。
Cloud Build 用のサービスアカウントの作成
```bash
gcloud iam service-accounts create cloud-build-runner 
```
2. Cloud Run 用のサービスアカウントの作成
```bash
gcloud iam service-accounts create gh-actions-demo-service
```

### Role の付与
Cloud Build で利用するサービス アカウントに **Cloud Build サービス アカウント** ・ **Cloud Deploy オペレーター** ・  **Cloud Run 管理者** ・ **サービス アカウント ユーザー** の権限を割り当てます。
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:cloud-build-runner@${PROJECT_ID}.iam.gserviceaccount.com --role=roles/cloudbuild.builds.builder
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:cloud-build-runner@${PROJECT_ID}.iam.gserviceaccount.com --role=roles/run.admin
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:cloud-build-runner@${PROJECT_ID}.iam.gserviceaccount.com --role=roles/iam.serviceAccountUser
```

Cloud Build の[サービス エージェント](https://cloud.google.com/iam/docs/service-account-types?hl=ja#service-agents)に **Secret Manager 管理者** の権限を割り当てます。この権限は GitHub Actions が Cloud Build のトリガーを起動する際に使用します。
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:service-$PROJECT_NUMBER@gcp-sa-cloudbuild.iam.gserviceaccount.com --role=roles/secretmanager.admin
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
| CLOUD_BUILD_SA_ID | cloud-build-runner@<walkthrough-project-id>.iam.gserviceaccount.com|"@"の後にプロジェクト ID が含まれているか|
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

2. GitHub Actions から Cloud Build を呼び出すため、Cloud Build で利用するサービス アカウントに対し、Workload Identity ユーザーの権限を追加します。
```bash
gcloud iam service-accounts add-iam-policy-binding cloud-build-runner@${PROJECT_ID}.iam.gserviceaccount.com \
    --role=roles/iam.workloadIdentityUser \
    --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github-actions-pool/attribute.repository_owner/${GITHUB_ACCOUNT}"
```

## 試してみる
main ブランチに何か変更をコミットしましょう