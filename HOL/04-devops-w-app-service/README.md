# DevOps

## Overview

In this lab, you will learn how to:

* Create a sample application and configure it for CI
* Create infrastructure as code
* Create a CI/CD pipeline
* Deploy the application to Azure

## Prerequisites

> NOTE: This HoL does _not_ depend on other modules to be completed.

* A VSTS and Azure Account are required for the completion of the lab.
* Create a project in Visual Studio Team Services using the defaults with Git as the repo type.
* .NET Core installed on your development machine. Use [https://www.microsoft.com/net/learn/get-started/](https://www.microsoft.com/net/learn/get-started/) to install.

## Exercises

This hands-on-lab has the following exercises:

1. [Exercise 1: Create an Application](#ex1)
1. [Exercise 2: Create the Environment](#ex2)
1. [Exercise 3: Create the CI/CD Pipeline](#ex3)
1. [Exercise 4: Deploy and test the application](#ex4)

### Exercise 1: Create an Application<a name="ex1"></a>

----

For the purpose of this lab, we are going to create a .NET Core application. You will do this from the jump box.

1. If not already connected, remote into the jump box. See [HOL 2](./02-configure-source-apps/)

1. Launch a PowerShell window

1. Use the `dotnet` command below to create a new MVC application, restore dependencies, build, and run it.

    ```powershell
    cd\
    md dotnetapp
    cd dotnetapp
    dotnet new mvc -f netcoreapp2.0
    dotnet restore
    dotnet build
    dotnet run
    ```

1. By default, the running webapp will be running on `http://localhost:5000`. It should look like the screenshot below.

    ![Base App Screenshot][1]

### Exercise 2: Create the Environment<a name="ex2"></a>

----

Now that we have an app we need to get the supporting infrastructure in Azure to support it. We will be using a CI/CD pipeline and want to deploy the infrastructure as part of that pipeline. This will be done via an Azure Resource Manager templates. A wealth of Resource Manager templates can be found on the [Azure GitHub repo](https://github.com/Azure/azure-quickstart-templates).

For this HoL, we are going to use the template for [Deploying a Web App with custom deployment slots](https://github.com/Azure/azure-quickstart-templates/tree/master/101-webapp-custom-deployment-slots).

> Note: While this template has a _"Deploy to Azure"_ button, we will be using it in our CI/CD pipeline.

1. Download the _azuredeploy.json_ and _azuredeploy.paramaters.json_ file to your local directory. We will use PowerShell to pull the files locally

    ````powershell
    Invoke-WebRequest -Uri 'https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/101-webapp-custom-deployment-slots/azuredeploy.json'  -OutFile azuredeploy.json
    Invoke-WebRequest -Uri 'https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/101-webapp-custom-deployment-slots/azuredeploy.parameters.json' -OutFile azuredeploy.parameters.json

    ````

1. Your local directory should look like the screenshot below.

  ![Folder Screenshot][2]

### Exercise 3: Create the CI/CD Pipeline<a name="ex3"></a>

----

Before adding new files, we will define files that we don't want to track in the repo.

1. Create a file called `.gitignore` in the root of the directory.

1. Add the git ignore information on [https://raw.githubusercontent.com/OmniSharp/generator-aspnet/master/templates/gitignore.txt](https://raw.githubusercontent.com/OmniSharp/generator-aspnet/master/templates/gitignore.txt) page. It is template for ignoring C# build and working files from the repo.

    ````powershell
    Invoke-WebRequest -Uri 'https://raw.githubusercontent.com/OmniSharp/generator-aspnet/master/templates/gitignore.txt' -Outfile .gitignore
    ````

1. The code below will initiate the repo, add all the files to track, and commit them to the repo. From the root directory of the project:

      ```powershell
      git init
      git add -A
      git commit -m "Add initial files"
      ```

1. Connect the Git repo to VSTS. The following commands set the upstream directory and push the files to the remote repo.

    ```powershell
    git remote add origin <Your repo git url here>
    git push -u origin --all
    ```

### Build

----

With the code in VSTS, the CI/CD pipeline needs to be configured.

1. From the build screen, create a new build definition.

    ![Create Build Screenshot][3]

1. Select `VSTS Git` as the source with the default and then `Continue`.

1. We are going to build the pipeline from a blank template. From the `Select template` screen, Select `Empty process`.

1. The CI task screen should be open and should look like the screenshot below.

    ![Empty CI Tasks][4]

1. Add the following three .dotnet core tasks:

    * The restore command task
    * The build command task
    * The publish command task

1. Add `app/*.csproj` in the Path to project(s) section of the restore and build commands.

1. Add the following to the _Arguments_ section of the publish task

    ```powershell
      --configuration $(BuildConfiguration) --output $(build.artifactstagingdirectory)
    ```

1. Uncheck the `Add project name to publish path` from the publish task.

1. Add a task for `Publish Artifact`. Configure it based on the image below.

    ![Publish Artifact][5]

1. Before we can run a build with this pipeline, a `BuildConfiguration` variable needs to be configured. Click on the Variables tab, and add a variable `BuildConfiguration` with a value of `release`.

1. Click `Save and Queue`

1. The last thing we are going to do is package up our environment to consume in our deployment phase. Navigate to the `Tasks` tab

1. Add an agent phase by clicking on the _"..."_ in the Process bar above the tasks.

1. Add an agent phase and then a `Publish Artifacts` task.

1. Select `env` as the path to publish, `envdrop` as the _Artifact Name_, and _Visual Studio Team Services/TFS_ as the publish location. The Build Definition will look like the image below with all the tasks set up.

    ![Build Definition][8]

    The last thing to configure in the Build definition is how it is triggered.

1. Click on the _Triggers_ tab and click _Enable continuous integration_

1. Accept the defaults and save the definition. This setting enables a build when check ins are made on the master branch of our project.

### Deployment

----

Now that we have our Build definition set up, we need to create our release pipeline.

1. Click on the Release tab and then then _New Definition_ button. Similar to the Build Definition steps, we are going to start with an _Empty process_.

    ![New Release Pipeline][6]

1. Name the environment _Development_ and then click on _Add_ above artifacts to pull the build into the release definition.

1. Select the build drop down from the menu and accept the defaults that auto populate.

1. Click _Add_. With artifacts added to our pipeline, we need to add the release tasks.

1. Click the _Tasks_ tab, shown in the image below.

    ![Add Release Tasks][7]

1. Click the _+_ next to _Agent Phase_ and add an Azure Resource Group Deployment. Select your subscription from the drop down menu and click _Authorize_ to connect VSTS to your subscription. Configure the task with the following settings:

    * Resource Group: `$(resourceGroup)`
    * Location: `$(location)`
    * Override template parameters: `-baseResourceName $(baseResourceName`

1. These variables need to be set in the _Variables_ tab. Example below:

    ![Release Variables][9]

1. Click the _'...'_ next to _Template_ and select the _azuredeploy.json_ file inside of the _envdrop_ folder. Do the same for _Template parameters_ and select the _azuredeploy.parameters.json_ file.

1. With the parameters set. Click _Save_.

1. Now that the Resource Group deployment is set. Add a new task for Azure App Service Deploy. Click the _+_ in the Agent Phase bar.

1. Select the same azure subscription and then set the following options.

    * App type: Web App
    * App service name:$(baseResourceName)Portal _// This will use the varialbe but is based off the parameter name in the ARM template_

### Automate Deployment

----

With the CI and CD configured it is time to deploy the application in an automated pipeline.

1. From inside the release definition, open the _Pipeline_ tab.

1. Click on the lightning bolt in the artifacts object and enable the _Continuous deployment trigger_.

1. With this pipeline, when you check code into the _master_ branch of the repo and it will kickoff a build and release.

### Exercise 4:  Test<a name="ex4"></a>

----

1. With the successful deployment, open your Azure Portal and navigate to the WebApp.

1. Click on the url to launch the site.

## References

_ADVANCED TOPIC (Preview Feature as of 02/27/2018)_ - [CI as Code via YAML file](https://docs.microsoft.com/en-us/vsts/build-release/actions/build-yaml).

  [1]: ./media/HOL05_01.png
  [2]: ./media/HOL05_02.PNG
  [3]: ./media/HOL05_03.png
  [4]: ./media/HOL05_04.png
  [5]: ./media/HOL05_05.png
  [6]: ./media/HOL05_06.png
  [7]: ./media/HOL05_07.png
  [8]: ./media/HOL05_08.png
  [9]: ./media/HOL05_09.png

## Summary

In this hands-on lab, you learned how to:

* Create a Visual Studio build pipeline
* Configure source control integration for continuous integration
* Create a release pipeline

----

Copyright 2016 Microsoft Corporation. All rights reserved. Except where otherwise noted, these materials are licensed under the terms of the MIT License. You may use them according to the license as is most appropriate for your project. The terms of this license can be found at https://opensource.org/licenses/MIT.
