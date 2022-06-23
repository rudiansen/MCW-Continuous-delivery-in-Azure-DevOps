# Continuous Delivery in Azure DevOps hands-on lab step-by-step

## Contents

<!-- TOC -->

- [Exercise 2: Continuous Delivery / Continuous Deployment with GitHub](#exercise-2-continuous-delivery--continuous-deployment-with-github)
    - [Task 1: Set up Cloud Infrastructure](#task-1-set-up-cloud-infrastructure)
    - [Task 2: Deploy to Azure Web Application](#task-2-deploy-to-azure-web-application)
    - [Task 3: Continuous Deployment with GitHub Actions](#task-3-continuous-deployment-with-github-actions)
    - [Task 4: Branch Policies in GitHub (Optional)](#task-4-branch-policies-in-github-optional)

<!-- /TOC -->

## Exercise 2: Continuous Delivery / Continuous Deployment with GitHub

Duration: 40 minutes

The Fabrikam Medical Conferences developer workflow has been improved. We are ready to consider migrating from running on-premises to a cloud implementation to reduce maintenance costs and facilitate scaling when necessary. We will take steps to run the containerized application in the cloud as well as automate its deployment.

**Help references**

|                                       |                                                                        |
| ------------------------------------- | ---------------------------------------------------------------------- |
| **Description**                       | **Link**                                                              |
| What is Continuous Delivery? | <https://docs.microsoft.com/devops/deliver/what-is-continuous-delivery> |
| Continuous delivery vs. continuous deployment | <https://azure.microsoft.com/overview/continuous-delivery-vs-continuous-deployment/> |
| Microsoft Learn - Introduction to continuous delivery | <https://docs.microsoft.com/learn/modules/introduction-to-continuous-delivery> |
| Microsoft Learn - Explain DevOps Continuous Delivery and Continuous Quality | <https://docs.microsoft.com/learn/modules/explain-devops-continous-delivery-quality/> |

### Task 1: Set up Cloud Infrastructure

First, we need to set up the cloud infrastructure. We will use PowerShell scripts and the Azure Command Line Interface (CLI) to set this up.

1. Open your local GitHub folder for your `mcw-continuous-delivery-lab-files` repository.

2. Open the `deploy-infrastructure.ps1` PowerShell script in the `infrastructure` folder. Add a custom lowercase three-letter abbreviation for the `$studentprefix` variable on the first line.

    ```pwsh
    $studentprefix = "Your 3 letter abbreviation here"  # <-- Modify this value
    $resourcegroupName = "fabmedical-rg-" + $studentprefix
    $cosmosDBName = "fabmedical-cdb-" + $studentprefix
    $webappName = "fabmedical-web-" + $studentprefix
    $planName = "fabmedical-plan-" + $studentprefix
    $location1 = "eastasia"
    $location2 = "southeastasia"
    ```

3. Note the individual calls to the Azure CLI for the following:
    - Creating a Resource Group

        ```pwsh
        # Create resource group
        az group create `
            --location $location1 `
            --name $resourcegroupName
        ```

    - Creating an Azure Cosmos DB Database

        ```pwsh
        # Create Azure Cosmos DB database
        az cosmosdb create `
            --name $cosmosDBName `
            --resource-group $resourcegroupName `
            --locations regionName=$location1 failoverPriority=0 isZoneRedundant=False `
            --locations regionName=$location2 failoverPriority=1 isZoneRedundant=True `
            --enable-multiple-write-locations `
            --kind MongoDB
        ```

    - Creating an Azure App Service Plan

        ```pwsh
        # Create Azure App Service Plan
        az appservice plan create `
            --name $planName `
            --resource-group $resourcegroupName `
            --sku S1 `
            --is-linux
        ```

    - Creating an Azure Web App

        ```pwsh
        # Create Azure Web App with NGINX container
        az webapp create `
            --resource-group $resourcegroupName `
            --plan $planName `
            --name $webappName `
            --deployment-container-image-name nginx
        ```

4. Log into Azure using the Azure CLI.

    ```pwsh
    az login
    az account set --subscription <your subscription guid>
    ```

    **Note**: Your subscription plan guid is the `id` field that comes back in the response JSON. In the following example, the subscription guid is `726da029-91f0-4dc1-a728-f25664374559`.

    ```json
    {
    "cloudName": "AzureCloud",
    "homeTenantId": "8f4781a5-82b9-4181-a022-4e9e91028be4",
    "id": "726da029-91f0-4dc1-a728-f25664374559",
    "isDefault": true,
    "managedByTenants": [],
    "name": "Your Azure Subscription Name",
    "state": "Enabled",
    "tenantId": "8f4781a5-82b9-4181-a022-4e9e91028be4",
    "user": {
      "name": "your-name@your-domain.com",
      "type": "user"
    }
    ```

5. Run the `deploy-infrastructure.ps1` PowerShell script.

    ```pwsh
    cd ./infrastructure
    ./deploy-infrastructure.ps1
    ```

    >**Note**: Depending on your system, you may need to change the PowerShell Execution Policy. You can read more about this process [here.](https://docs.microsoft.com/powershell/module/microsoft.powershell.core/about/about_execution_policies)

6. Browse to [the Azure portal](https://portal.azure.com) and verify creation of the resource group, Azure Cosmos DB instance, the App Service Plan, and the Web App.

    ![Azure Resource Group containing cloud resources to which GitHub will deploy containers via the workflows defined in previous steps.](../media/hol-ex2-task1-step6-1.png "Azure Resource Group")

7. Open the `seed-cosmosdb.ps1` PowerShell script in the `infrastructure` folder of your lab files GitHub repository and add the same custom lowercase three-letter abbreviation we used in step 2 for the `$studentprefix` variable on the first line. Also update the `$githubAccount` variable with your GitHub account name.

    ```pwsh
    $studentprefix = "Your 3 letter abbreviation here"
    $githubAccount = "Your github account name here"
    $resourcegroupName = "fabmedical-rg-" + $studentprefix
    $cosmosDBName = "fabmedical-cdb-" + $studentprefix
    ```

8. Observe the call to fetch the MongoDB connection string for the Azure Cosmos DB database.

    ```pwsh
    # Fetch Azure Cosmos DB Mongo connection string
    $mongodbConnectionString = `
        $(az cosmosdb keys list `
            --name $cosmosDBName `
            --resource-group $resourcegroupName `
            --type connection-strings `
            --query 'connectionStrings[0].connectionString')
    ```

9. The call to seed the Azure Cosmos DB database is using the MongoDB connection string passed as an environment variable (`MONGODB_CONNECTION`) to the `fabrikam-init` docker image we built in the previous exercise using `docker-compose`.

    ```pwsh
    # Seed Azure Cosmos DB database
    docker run -ti `
        -e MONGODB_CONNECTION="$mongodbConnectionString" `
        ghcr.io/$githubAccount/fabrikam-init:main
    ```

    >**Note**: Before you pull this image, you may need to authenticate with the GitHub Docker registry. To do this, run the following command before you execute the script. Fill the placeholder appropriately. Use your PAT when it prompts for the password.

    ```pwsh
    docker login ghcr.io -u [USERNAME]
    ```

10. Run the `seed-cosmosdb.ps1` PowerShell script. Browse to [the Azure portal](https://portal.azure.com) and verify that the Azure Cosmos DB instance has been seeded.

    ![Azure Cosmos DB contents displayed via the Azure Cosmos DB explorer in the Azure Cosmos DB resource detail.](../media/hol-ex2-task1-step10-1.png "Azure Cosmos DB Seeded Contents")

     >**Note**: If the `seed-cosmosdb.ps1` script cannot find the `fabrikam-init` image, you may need to check the possible versions by looking at the `fabrikam-init` package page in your `mcw-continuous-delivery-lab-files` repository in GitHub.

    ![fabrikam-init package details displayed in the mcw-continuous-delivery-lab-files repository in GitHub.](../media/hol-ex2-task1-step10-2.png "fabrikam-init package details in GitHub")

11. Below the `sessions` collection, select **Scale & Settings** (1) and **Indexing Policy** (2).

    ![Opening indexing policy for the sessions collection.](../media/hol-ex2-task1-step11.png "Indexing policy configuration")

12. Create a Single Field indexing policy for the `startTime` field (1). Then, select **Save** (2).

    ![Creating an indexing policy for the startTime field.](../media/hol-ex2-task1-step12.png "startTime field indexing")

13. Open the `configure-webapp.ps1` PowerShell script in the `infrastructure` folder of your lab files GitHub repository and add the custom lowercase three-letter abbreviation you have been using for the `$studentprefix` variable on the first line.

    ```pwsh
    $studentprefix = "Your 3 letter abbreviation here"
    $resourcegroupName = "fabmedical-rg-" + $studentprefix
    $cosmosDBName = "fabmedical-cdb-" + $studentprefix
    $webappName = "fabmedical-web-" + $studentprefix
    ```

14. Observe the call to configure the Azure Web App using the MongoDB connection string passed as an environment variable (`MONGODB_CONNECTION`) to the web application.

    ```pwsh
    # Configure Web App
    az webapp config appsettings set `
        --name $webappName `
        --resource-group $resourcegroupName `
        --settings MONGODB_CONNECTION=$mongodbConnectionString
    ```

15. Run the `configure-webapp.ps1` PowerShell script. Browse to [the Azure portal](https://portal.azure.com) and verify that the environment variable `MONGODB_CONNECTION` has been added to the Azure Web Application settings.

    ![Azure Web Application settings reflecting the `MONGODB_CONNECTION` environment variable configured via PowerShell.](../media/hol-ex2-task1-step15-1.png "Azure Web Application settings")

### Task 2: Deploy to Azure Web Application

Once the infrastructure is in place, then we can deploy the code to Azure. In this task, you will deploy the application to an Azure Web Application using a PowerShell script that makes calls with the Azure CLI.

1. Take the GitHub Personal Access Token you obtained in the Before the Hands-On Lab guided instruction and assign it to the `CR_PAT` environment variable in PowerShell. We will need this environment variable for the `deploy-webapp.ps1` PowerShell script, but we do not want to add it to any files that may get committed to the repository since it is a secret value.

    ```pwsh
    $env:CR_PAT="<GitHub Personal Access Token>"
    ```

2. Open the `deploy-webapp.ps1` PowerShell script in the `infrastructure` folder of your lab files GitHub repository and add the same custom lowercase three-letter abbreviation we used in step 1 for the `$studentprefix` variable on the first line and add your GitHub account name for the `$githubAccount` variable on the second line.

    ```pwsh
    $studentprefix = "Your 3 letter abbreviation here"
    $githubAccount = "Your github account name here"
    $resourcegroupName = "fabmedical-rg-" + $studentprefix
    $webappName = "fabmedical-web-" + $studentprefix
    ```

3. The call to deploy the Azure Web Application is using the `docker-compose.yml` file we modified in the previous exercise.

    ```pwsh
    # Deploy Azure Web App
    az webapp config container set `
        --docker-registry-server-password $env:CR_PAT `
        --docker-registry-server-url https://ghcr.io `
        --docker-registry-server-user $githubAccount `
        --multicontainer-config-file ./../docker-compose.yml `
        --multicontainer-config-type COMPOSE `
        --name $webappName `
        --resource-group $resourcegroupName
    ```

4. Run the `deploy-webapp.ps1` PowerShell script.

    > **Note**: Make sure to run the `deploy-webapp.ps1` script from the `infrastructure` folder

5. Browse to [the Azure portal](https://portal.azure.com) and verify that the Azure Web Application is running by checking the `Log stream` blade of the Azure Web Application detail page.

    ![Azure Web Application Log Stream displaying the STDOUT and STDERR output of the running container.](../media/hol-ex2-task2-step5-1.png "Azure Web Application Log Stream")

6. Browse to the `Overview` blade of the Azure Web Application detail page and find the web application URL. Browse to that URL to verify the deployment of the web application.

    ![The Azure Web Application Overview detail in Azure portal.](../media/hol-ex2-task2-step6-1.png "Azure Web Application Overview")

    ![The Contoso Conference website hosted in Azure.](../media/hol-ex2-task2-step6-2.png "Azure hosted Web Application")

### Task 3: Continuous Deployment with GitHub Actions

With the infrastructure in place, we can set up continuous deployment with GitHub Actions.

1. Open the `deploy-sp.ps1` PowerShell script in the `infrastructure` folder of your lab files GitHub repository and add the same custom lowercase three-letter abbreviation we used in a previous exercise for `$studentprefix` variable on the first line. Note the call to create a Service Principal.

    ```pwsh
    $studentprefix ="Your 3 letter abbreviation here"
    $resourcegroupName = "fabmedical-rg-" + $studentprefix

    $id = $(az group show `
        --name $resourcegroupName `
        --query id)

    az ad sp create-for-rbac `
        --name "fabmedical-$studentprefix" `
        --sdk-auth `
        --role contributor `
        --scopes $id
    ```

2. Execute the `deploy-sp.ps1` PowerShell script. Copy the resulting JSON output for use in the next step.

    ```pwsh
    {
        "clientId": "...",
        "clientSecret": "...",
        "subscriptionId": "...",
        "tenantId": "...",
        "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
        "resourceManagerEndpointUrl": "https://management.azure.com/",
        "activeDirectoryGraphResourceId": "https://graph.windows.net/",
        "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
        "galleryEndpointUrl": "https://gallery.azure.com/",
        "managementEndpointUrl": "https://management.core.windows.net/"
    }
    ```

3. Create a new repository secret named `AZURE_CREDENTIALS`. Paste the JSON output copied from Step 2 to the secret value and save it.

4. Edit the `docker-publish.yml` file in the `.github\workflows` folder. Add the following job to the end of this file:

    > **Note**: Make sure to change the student prefix for the last action in the `deploy` job.

    ```yaml
      deploy:
        # The type of runner that the job will run on
        runs-on: ubuntu-latest

        # Steps represent a sequence of tasks that will be executed as part of the job
        steps:
        # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
        - uses: actions/checkout@v2                

        - name: Login on Azure CLI
          uses: azure/login@v1.1
          with:
            creds: ${{ secrets.AZURE_CREDENTIALS }}

        - name: Deploy WebApp
          shell: pwsh
          env:
            CR_PAT: ${{ secrets.CR_PAT }}
          run: |
            cd ./infrastructure
            ./deploy-webapp.ps1 -studentprefix hbs  # <-- This needs to
                                                    # match the student
                                                    # prefix we use in
                                                    # previous steps.
    ```

5. Commit the YAML file to your `main` branch. A GitHub action should begin to execute for the updated workflow.

    > **Note**: Make sure that your Actions workflow file does not contain any syntax errors, which may appear when you copy and paste. They are highlighted in the editor or when the Action tries to run, as shown below.

    ![GitHub Actions workflow file syntax error.](../media/github-actions-workflow-file-error.png "Syntax error in Actions workflow file")

6. Observe that the action builds the docker images, pushes them to the container registry, and deploys them to the Azure web application.

    ![GitHub Action detail reflecting Docker ](../media/hol-ex3-task2-step8-1.png "GitHub Action detail")

7. Perform a `git pull` on your local repository folder to fetch the latest changes from GitHub.

### Task 4: Branch Policies in GitHub (Optional)

In many enterprises, committing to `main` branch is restricted. Branch policies are used to control how code gets to `main` branch. This allows you to set up gates on delivery, such as requiring code reviews and status checks. In this task, you will create a branch protection rule and see it in action.

>**Note**: Branch protection rules apply to Pro, Team, and Enterprise GitHub users.

1. In your lab files GitHub repository, navigate to the `Settings` tab and select the `Branches` blade.

    ![GitHub Branch settings for the repository](../media/hol-ex2-task3-step1-1.png "Branch Protection Rules")

2. Select the `Add rule` button to add a new branch protection rule for the `main` branch. Be sure to specify `main` in the branch name pattern field. Enable the following options and choose the `Create` button to create the branch protection rules:

   - Require pull request reviews before merging
   - Require status checks to pass before merging
   - Require branches to be up to date before merging

    ![Branch protection rule creation form](../media/hol-ex2-task3-step2-1.png "Create a new branch protection rule in GitHub")

3. With the branch protection rule in place, direct commits and pushes to the `main` branch will be disabled. Verify this rule by making a small change to your `README.md` file. Attempt to commit the change to the `main` branch in your local repository followed by a push to the remote repository.

    ```pwsh
    PS C:\Workspaces\lab\mcw-continuous-delivery-lab-files> git add .

    PS C:\Workspaces\lab\mcw-continuous-delivery-lab-files> git commit -m "Updating README.md"

    [main cafa839] Updating README.md
    1 file changed, 2 insertions(+)
    PS C:\Workspaces\lab\mcw-continuous-delivery-lab-files> git push

    Enumerating objects: 5, done.
    Counting objects: 100% (5/5), done.
    Delta compression using up to 32 threads
    Compressing objects: 100% (3/3), done.
    Writing objects: 100% (3/3), 315 bytes | 315.00 KiB/s, done.
    Total 3 (delta 2), reused 0 (delta 0), pack-reused 0
    remote: Resolving deltas: 100% (2/2), completed with 2 local objects.
    remote: error: GH006: Protected branch update failed for refs/heads/main.
    remote: error: At least 1 approving review is required by reviewers with write access.
    To https://github.com/YOUR_GITHUB_ACCOUNT/mcw-continuous-delivery-lab-files.git
    ! [remote rejected] main -> main (protected branch hook declined)
    error: failed to push some refs to 'https://github.com/YOUR_GITHUB_ACCOUNT/mcw-continuous-delivery-lab-files.git'
    ```

**[⬅️ back to home](../README.md#contents)**  **[⬆️ back to top](#contents)**