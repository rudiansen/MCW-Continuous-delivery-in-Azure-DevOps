![Microsoft Cloud Workshops](https://github.com/Microsoft/MCW-Template-Cloud-Workshop/raw/main/Media/ms-cloud-workshop.png "Microsoft Cloud Workshops")

<div class="MCWHeader1">
Continuous delivery in Azure DevOps
</div>

<div class="MCWHeader2">
Hands-on lab step-by-step
</div>

<div class="MCWHeader3">
November 2021
</div>

Information in this document, including URL and other Internet website references, is subject to change without notice. Unless otherwise noted, the example companies, organizations, products, domain names, e-mail addresses, logos, people, places, and events depicted herein are fictitious, and no association with any real company, organization, product, domain name, e-mail address, logo, person, place or event is intended or should be inferred. Complying with all applicable copyright laws is the responsibility of the user. Without limiting the rights under copyright, no part of this document may be reproduced, stored in or introduced into a retrieval system, or transmitted in any form or by any means (electronic, mechanical, photocopying, recording, or otherwise), or for any purpose, without the express written permission of Microsoft Corporation.

Microsoft may have patents, patent applications, trademarks, copyrights, or other intellectual property rights covering subject matter in this document. Except as expressly provided in any written license agreement from Microsoft, the furnishing of this document does not give you any license to these patents, trademarks, copyrights, or other intellectual property.

The names of manufacturers, products, or URLs are provided for informational purposes only and Microsoft makes no representations and warranties, either expressed, implied, or statutory, regarding these manufacturers or the use of the products with any Microsoft technologies. The inclusion of a manufacturer or product does not imply endorsement of Microsoft of the manufacturer or product. Links may be provided to third-party sites. Such sites are not under the control of Microsoft and Microsoft is not responsible for the contents of any linked site or any link contained in a linked site, or any changes or updates to such sites. Microsoft is not responsible for webcasting or any other form of transmission received from any linked site. Microsoft is providing these links to you only as a convenience, and the inclusion of any link does not imply endorsement of Microsoft of the site or the products contained therein.

© 2022 Microsoft Corporation. All rights reserved.

Microsoft and the trademarks listed at <https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks> are trademarks of the Microsoft group of companies. All other trademarks are property of their respective owners.

## Contents

<!-- TOC -->

- [Continuous Delivery in Azure DevOps hands-on lab step-by-step](#continuous-delivery-in-azure-devops-hands-on-lab-step-by-step)
  - [Abstract and learning objectives](#abstract-and-learning-objectives)
  - [Overview](#overview)
  - [Solution architecture](#solution-architecture)
  - [Requirements](#requirements)
  - [Before the hands-on lab](content/Before%20the%20HOL.md)
    - Task 1: Create the Project Repo in GitHub
    - Task 2: Create GitHub Personal Access Token
    - Task 3: Create Azure DevOps Personal Access Token
    - Task 4: Create Azure DevOps Project
    - Task 5: Connect Azure Board with GitHub
  - [Exercise 1: Continuous Integration](content/Exercise%201%20-%20Continuous%20Integration.md)
    - Task 1: Set up Local Infrastructure
    - Task 2: Build Automation with GitHub Registry
    - Task 3: Editing the GitHub Workflow File Locally
  - [Exercise 2: Continuous Delivery / Continuous Deployment](content/Exercise%202%20-%20Continuous%20Delivery%20-%20Continuous%20Deployment.md)
    - Task 1: Set up Cloud Infrastructure
    - Task 2: Deploy to Azure Web Application
    - Task 3: Continuous Deployment with GitHub Actions
    - Task 4: Branch Policies in GitHub (Optional)
  - [Exercise 3: Monitoring, Logging, and Continuous Deployment with Azure](content/Exercise%203%20-%20Continuous%20Deployment%20with%20Azure%20DevOps.md)
    - Task 1: Continuous Deployment with Azure DevOps Pipelines
    - Task 2: Linking Git commits to Azure DevOps issues
  - [After the hands-on lab](content/After%20the%20HOL.md)
    - Task 1: Tear down Azure Resources
  
<!-- /TOC -->

# Continuous Delivery in Azure DevOps hands-on lab step-by-step

## Abstract and learning objectives

In this hands-on lab, you will learn how to implement a solution with a combination of Azure CLI commands and Azure DevOps to enable continuous delivery with several Azure PaaS services.

At the end of this workshop, you will be better able to implement solutions for continuous delivery with GitHub in Azure, as well create Azure CLI commands to provision Azure resources, create an Azure DevOps project with a GitHub repository, and configure continuous delivery with GitHub.

## Overview

Fabrikam Medical Conferences provide conference website services tailored to the medical community. Over ten years, they have built conference sites for a small conference organizer. Through word of mouth, Fabrikam Medical Conferences has become a well-known industry brand handling over 100 conferences per year and growing.

Websites for medical conferences are typically low-budget websites because the conferences usually have between 100 to 1500 attendees. At the same time, the conference owners have significant customization and change demands that require turnaround on a dime to the live sites. These changes can impact various aspects of the system from UI through to the back end, including conference registration and payment terms.

## Solution architecture

![Solution architecture diagram illustrating the use of GitHub and Azure DevOps.](media/diagram.png "Desired solution architecture")

## Requirements

1. Microsoft Azure subscription must be Pay-As-You-Go or MSDN.
   - Trial subscriptions will _not_ work.
   - To complete this lab setup, ensure your account includes the following:
     - Has the [Owner](https://docs.microsoft.com/azure/role-based-access-control/built-in-roles#owner) built-in role for the subscription you use.
     - Is a [Member](https://docs.microsoft.com/azure/active-directory/fundamentals/users-default-permissions#member-and-guest-users) user in the Azure AD tenant you use. If you are a Guest user, please make sure you have `Application administrator` and `Authentication policy administrator` assignment roles in order to be able to create Service Principal in Azure AD.

2. If the students need to use Microsoft-hosted Azure Pipelines Agents to run CI/CD pipelines in Azure DevOps, they will need to request a free grant of parallel jobs in Azure Pipelines via [this form](https://aka.ms/azpipelines-parallelism-request). It should be completed at least **2-3 business days** before the lab in order to run parallel jobs in Azure DevOps. The other option is to use self-hosted agents. If you need to go the self-hosted route, refer to the [Azure Pipelines Agents documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=browser).

3. A [GitHub](https://github.com) account.

4. Local machine or a Virtual Machine configured with:

    - A browser, preferably Chrome, to be consistent with the lab implementation tests.

5. [Git for Windows](https://gitforwindows.org/)

6. [PowerShell](https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell?view=powershell-7)

    - As you will be running PowerShell scripts, make sure that the ExecutionPolicy is set properly. Consult [the Microsoft PowerShell documentation on execution policies](https://docs.microsoft.com/powershell/module/microsoft.powershell.core/about/about_execution_policies) for more details.

7. [Docker Desktop for Windows](https://docs.docker.com/desktop/windows/install/)

8. [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)

9. Angular - minimum 8.3.4.

    - Angular depends on `Node.js` and `npm`.
    - Consult [Setting up the local environment and workspace](https://angular.io/guide/setup-local) on the Angular site for guidance.
    - This lab has been tested with [Node.js](https://nodejs.org/en/download/) version 16.13.0, which includes npm 8.1.0.

10. [Visual Studio Code](https://code.visualstudio.com/download) *(optional)*

**[⬆ back to top](#contents)**