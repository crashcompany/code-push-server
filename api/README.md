# CodePush Server

The CodePush Server is a Node.js application that powers the CodePush Service. It allows users to deploy and manage over-the-air updates for their react-native applications in a self-hosted environment.

Please refer to [react-native-code-push](https://github.com/microsoft/react-native-code-push) for instructions on how to onboard your application to CodePush.

## Deployment

### Local

#### Prerequisites

The CodePush Server requires Azure Blob Storage to operate. For the local setup, there is an option to use emulated local storage with Azurite. 
Please follow Azurite [official documentation](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azurite) to [install](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azurite?tabs=visual-studio#install-azurite) and [run](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azurite?tabs=visual-studio#running-azurite-from-the-command-line) it locally.
Additionally, you need to specify [EMULATED](ENVIRONMENT.md#emulated) flag equals true in the environmental variables.

#### Steps
To run the CodePush Server locally, follow these steps:

1. Clone the CodePush Service repository to your local machine.

2. Copy the `.env.example` file to a new file named `.env` in the root directory:
   ````bash
   cp .env.example .env
   ````
   Fill in the values for each environment variable in the `.env` file according to your development or production setup.
3. Install all necessary dependencies:
   ````bash
   npm install
   ````
4. Compile the server code:
   ````bash
   npm run build
   ````
5. Launch the server with the environment-specific start command:
   ````bash
   npm run start:env
   ````

By default, local CodePush server runs on HTTP. To run CodePush Server on HTTPS:

1. Create a `certs` directory and place `cert.key` (private key) and `cert.crt` (certificate) files there.
2. Set environment variable [HTTPS](./ENVIRONMENT.md#https) to true.
 
> **Warning!** When hosting CodePush on Azure App Service HTTPS is enabled by default.

For more detailed instructions and configuration options, please refer to the [ENVIRONMENT.md](./ENVIRONMENT.md) file.

### Azure

CodePush Server is designed to run as [Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/overview).

#### Prerequisites

To deploy CodePush to Azure, an active Azure account and subscription are needed. 
For more information, follow Azure's [official documentation](https://azure.microsoft.com/en-us/get-started/).
During the deployment process, the included bicep script will create bare minimum Azure services needed to run CodePush Server including:
1. Service plan
2. App Service
3. Storage account

Additionally, for user authentication, a GitHub or Microsoft OAuth application is needed. 
More detailed instructions on how to set up one can be found in the section [OAuth Apps](#oauth-apps).

#### Steps

**NOTE** Please be aware of [project-suffix naming limitations](#project-suffix) for resources in Azure .

0. Install Azure CLI
   * `brew install azure-cli`
1. Login to your Azure account
   * `az login`
2. Select subscription for deployment
   * `az account set --subscription <subscription-id>`
3. Create resource group for CodePush resources: 
   * `az group create --name code-push-server --location koreacentral`
   * 모든 Azure 리소스는 반드시 하나의 Resource Group에 속해야 함
   * Azure의 중요 개념; AWS에는 유사 개념 없음
4. Deploy infrastructure using `codepush-infrastructure.bicep` with the next command
   * `az deployment group create --resource-group code-push-server --template-file ./codepush-infrastructure.bicep --parameters project_suffix=crashco az_location=koreacentral github_client_id=Ov23liYmoMMEmKRh5gq5 github_client_secret=<github-client-secret> microsoft_client_id=<microsoft-client-id> microsoft_client_secret=<microsoft-client-secret>`
   * OAuth parameters (both GitHub and Microsoft) are optional. It is possible to specify them after the deployment in environment settings of Azure WebApp.
   * --parameters 이후의 것들은 전부 bicep 파일 내에 있는 param 변수들이다
      * 위와 같이 입력하지 않으면 CLI 통해서 어차피 다시 물어봄
   * 명령어를 중복 수행해도, 리소스 중복 생성에 대한 에러가 나지 않음
5. Deploy CodePush to the Azure WebApp created during infrastructure deployment ➡️ See [Next Chapter](#deploy-codepush-to-the-azure-webapp-created-during-infrastructure-deployment)

> **Warning!** The created Azure Blob Storage has default access settings. 
> This means that all users within the subscription can access the storage account tables. 
> Adjusting the storage account access settings to ensure proper security is the responsibility of the owner.

## Deploy CodePush to the Azure WebApp created during infrastructure deployment

>  Follow the Azure WebApp [official documentation](https://learn.microsoft.com/en-us/azure/app-service/) "Deployment and configuration" section for detailed instructions.

0. 이미 위의 4단계에서 Azure WebApp이 생성됨
1. 해당 프로젝트를 Azure WebApp의 Git Repo와 연결
   * Azure Portal > App Services > 생성된 WebApp > Deployment > Deployment Center
   * Source = Local Git
   * Git Clone URI 확보
2. Git push를 위한 credential (publishingUserName, publishingPassword) 확보
   * `az webapp deployment list-publishing-credentials --name <web-app-name> --resource-group <resource-group-name>`
3. Credential과 Clone URI를 이용해 remote 등록
   * `git remote add azure https://<publishingUserName>:<publishingPassword>@<web-app-name>.scm.azurewebsites.net:443/<web-app-name>.git`
   * list-publishing-credentials 에서 나온 `scmUri` 값은 정확하지 않음
   * `$`로 시작하는 username을 remote에 추가시 `$` 앞에 `\`(backslash) 붙여야 함
4. Push
   * `git push azure main:master` (로컬의 main 브랜치를 리모트의 master 브랜치와 연결)

## Configure react-native-code-push

In order for [react-native-code-push](https://github.com/microsoft/react-native-code-push) to use your server, additional configuration value is needed.

### Android

in `strings.xml`, add following line, replacing `server-url` with your server.

```
<string moduleConfig="true" name="CodePushServerUrl">server-url</string>
```

### iOS

in `Info.plist` file, add following lines, replacing `server-url` with your server.

```
<key>CodePushServerURL</key>
<string>server-url</string>
```

## OAuth apps

CodePush uses GitHub and Microsoft as identity providers, so for authentication purposes, you need to have an OAuth App registration for CodePush. 
Client id and client secret created during registration should be provided to the CodePush server in environment variables. 
Below are instructions on how to create OAuth App registrations.

### GitHub

1. Go to https://github.com/settings/developers
1. Click on `New OAuth App`
1. `Homepage URL` parameter will be the same as URL of your CodePush application on Azure - `https://codepush-<project-suffix>.azurewebsites.net` (for local development it will be either http://localhost:3000 or https://localhost:8443)
1. `Authorization callback URL` will be `https://codepush-<project-suffix>.azurewebsites.net/auth/callback/github` (for local development it will be either http://localhost:3000/auth/callback/github or https://localhost:8443/auth/callback/github)

### Microsoft

Both work and personal accounts use the same application for authentication. The only difference is property `Supported account types` that is set when creating the app.

1. Register an Azure Registered Application following [official guideline](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app#register-an-application)
1. For option `Supported account types`:
   1. If you want to support both Personal and Work accounts, select `Accounts in any organizational directory (Any Microsoft Entra ID tenant - Multitenant) and personal Microsoft accounts (e.g. Skype, Xbox)`
   1. If you want to only support Work accounts, choose either `Accounts in this organizational directory only (<your directory> - Single tenant)` or `Accounts in any organizational directory (Any Microsoft Entra ID tenant - Multitenant)` depending if you want to support Single or Multitenant authorization. Make sure to set `MICROSOFT_TENANT_ID` envrionment variable in case of using single tenant application.
   1. If you want to only support Personal accounts, select `Personal Microsoft accounts only`
1. Set up Redirect URI(s) depending on the choice you made for `Supported account types`. If you choose both Personal and Work accounts, you need to add both redirect URIs, otherwise just one of the ones:
   1. Personal account: `https://codepush-<project-suffix>.azurewebsites.net/auth/callback/microsoft` (for local development it will be either http://localhost:3000/auth/callback/microsoft or https://localhost:8443/auth/callback/microsoft)
   1. Work account: `https://codepush-<project-suffix>.azurewebsites.net/auth/callback/azure-ad` (for local development it will be http://localhost:3000/auth/callback/azure-ad or https://localhost:8443/auth/callback/azure-ad)
1. Generate secret following this [official guideline](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app#add-credentials)

## Naming limitations

### project-suffix

1. Only letters are allowed.
1. Maximum 15 characters.

## Metrics

Installation metrics allow monitoring release activity via the CLI. For detailed usage instructions, please refer to the [CLI documentation](../cli/README.md#development-parameter).

Redis is required for Metrics to work.

### Steps

1. Install Redis by following [official installation guide](https://redis.io/docs/latest/operate/oss_and_stack/install/install-redis/).
1. TLS is required. Follow [official Redis TLS run guide](https://redis.io/docs/latest/operate/oss_and_stack/management/security/encryption/#running-manually).
1. Set the necessary environment variables for [Redis](./ENVIRONMENT.md#redis).