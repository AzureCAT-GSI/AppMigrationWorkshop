# DevOps with Containers

## Overview

In this lab, you will create a VSTS CI/CD pipeline that will deploy one of the apps in a container to the container host.

## Prerequisites

* You have created the environment described in HOL 1 Setup
* You have run through the configuration for the Container Host VM as described in HOL 7
* You have a Dockerfile image like the one created in HOL 7

## Exercises

This hands-on-lab has the following exercises:
1. [Exercise 1: Setup Network Security Groups, Firewall Ports, and Puplic IP for Container Host](#ex1)
1. [Exercise 2: Push code to VSO](#ex2)
1. [Exercise 3: Create an Azure Container Registry](#ex3)
1. [Exercise 4: Create a Build definitions](#ex4)
1. [Exercise 5: Create a Release definition](#ex5)

### Exercise 1 - Setup Network Security Groups, Firewall Ports, and Puplic IP for Container Host<a name="ex1"></a>
1. Login to https://shell.azure.com
    ```powershell
    Select-AzureRmSubscription -Subscription "<your subscription name>"
    ```
1. Create a public IP in Azure and add it to your Windows Container Host
    ```powershell
    #set your vairable nems for resource group, location, the Windows container host VM's NIC, and new PIP name
    $rgName = "<your resource group name>"
    $location = "<your deployment location>"
    $nicName = "<your container host vm name>-cnt-NIC-001"
    $dnsLabel = "<your container host vm name>"
    $pipName = "<your container host vm name>-cnt-NIC-001-PIP"

    #Deploy the new PIP
    $pip = New-AzureRmPublicIpAddress -ResourceGroupName $rgName -Name $pipName -Location $location -AllocationMethod Dynamic -DomainNameLabel $dnsLabel

    #Bind the new PIP to the Windows Container Host VM's NIC
    $nic = Get-AzureRmNetworkInterface -Name $nicName -ResourceGroupName $rgName
    $nic.IpConfigurations[0].PublicIpAddress = $pip 
    Set-AzureRmNetworkInterface -NetworkInterface $nic

    #Validate that a Public IP Address has been assigned
    (Get-AzureRmPublicIpAddress -Name $pipName -ResourceGroupName $rgName).IpAddress
    ```
1. RDP/Login to the Windows Container Host and open the needed firewall ports. Open a cmd or powershell prompt.
    ```powershell
    netsh advfirewall firewall add rule name="Docker Rule" dir=in action=allow protocol=TCP localport=2375
1. Create a Rule on your NSG to allow remote management of docker on your host vm.
    ```powershell
    $rgName = "<your resource group name>"
    $nsgName = "<your NSG name>"
    
    #Add a rule for docker deployments to your existing NSG
    $nsg = Get-AzureRmNetworkSecurityGroup -Name  $nsgName -ResourceGroupName $rgName `
        | Add-AzureRmNetworkSecurityRuleConfig -Name docker-rule -Description "Docker Rule" -Access Allow `
        -Protocol Tcp -Direction Inbound -Priority 200 -SourceAddressPrefix Internet -SourcePortRange * `
        -DestinationAddressPrefix 10.0.0.5  -DestinationPortRange 2375 |  Set-AzureRmNetworkSecurityGroup
    ```

1. Validate from your development environment that you can connect to the host sever.
    ```powershell
    #Get your Publick IP Address and the DNS Lable
    (Get-AzureRmPublicIpAddress -Name $pipName -ResourceGroupName $rgName).IpAddress
    (Get-AzureRmPublicIpAddress -Name $pipName -ResourceGroupName $rgName).DnsSettingsText

    docker --host tcp://<your Windows Contianer Host Public IP address> info
    PS C:\WINDOWS\system32> docker --host tcp://13.66.48.150 info
        Containers: 0
        Running: 0
        Paused: 0
        Stopped: 0
        Images: 2
        Server Version: 17.06.2-ee-6
        Storage Driver: windowsfilter
        Windows: 
        Logging Driver: json-file
        Plugins:
        Volume: local
        Network: l2bridge l2tunnel nat null overlay transparent
        Log: awslogs etwlogs fluentd json-file logentries splunk syslog
        Swarm: inactive
        Default Isolation: process
        Kernel Version: 10.0 14393 (14393.2068.amd64fre.rs1_release.180209-1727)
        Operating System: Windows Server 2016 Datacenter
        OSType: windows
        Architecture: x86_64
        CPUs: 2
        Total Memory: 7GiB
        Name: apm3pjq-cnt
        ID: KUHW:KYVP:OMYO:XW5A:DKIR:QDAA:2X7Z:NRZV:JYNQ:5ZF3:FXQI:NDYW
        Docker Root Dir: C:\ProgramData\docker
        Debug Mode (client): false
        Debug Mode (server): false
        Registry: https://index.docker.io/v1/
        Experimental: false
        Insecure Registries:
        127.0.0.0/8
        Live Restore Enabled: false
    ```


### Exercise 2 - Push your application VSO<a name="ex2"></a>

This exercise should be done from the Container host machine

1. Open an browser and login to your Visual Studio Online account

1. From the Visual Studio online main menu, click `Code`

1. Click `Initialize`

    ![image](./media/2018-03-21_1-18-31.png)

1. Git is not installed on the container host so let's create a container that has Git so we can upload our source. Open a PowerShell session

1. Create a folder for our new docker container, copy our jobs web site source and create a new docker file. We need to the jobs website source in the local folder so we can copy it into the container.

    > Note: Assumes that the jobs web site source is in c:\jobswebsite

    ````powershell
    md c:\dockerimages
    cd dockerimages
    md gitcontainer
    cd gitcontainer
    md jobswebsite
    xcopy C:\jobswebsite\*.* . /s
    New-Item dockerfile -ItemType File
    notepad.exe dockerfile
    ````
1. Paste the following into the new file and save

    ````docker
    FROM microsoft/windowsservercore:10.0.14393.2007

    ENV GIT_VERSION 2.16.2
    ENV GIT_PATCH_VERSION 1

    RUN powershell -Command $ErrorActionPreference = 'Stop' ; \
        [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12 ; \
        Invoke-WebRequest $('https://github.com/git-for-windows/git/releases/download/v{0}.windows.{1}/MinGit-{0}-64-bit.zip' -f $env:GIT_VERSION, $env:GIT_PATCH_VERSION) -OutFile 'mingit.zip' -UseBasicParsing ; \
        Expand-Archive mingit.zip -DestinationPath c:\mingit ; \
        Remove-Item mingit.zip -Force ; \
        setx /M PATH $('c:\mingit\cmd;{0}' -f $env:PATH)

        COPY /jobswebsite c:\jobswebsite

        ENTRYPOINT powershell.exe
    ````

1. Build the container

    > Note: The container name used in the example is minigitwin.

    ````powershell
    docker build -t minigitwin .
    ````
1. Run the container in interactively

    ````powershell
    docker run -it minigitwin powershell
    ````

1. From the Visual Studio online portal, generate temporary credentials for Git. We need these to clone the repository. Add an alias and a password and click `Save Git Credentials`
    > You can also create a personal access token (PAT)

    ![image](./media/2018-03-21_2-15-36.png)

1. From the container command line, clone the repository

    ````powershell
    cd\
    git clone https://YOUR VSO ACCOUNT].visualstudio.com/_git/[YOUR REPO NAME]

    Cloning into '[YOUR REPO NAME]'...
    Username for 'https://[YOUR VSO ACCOUNT].visualstudio.com': [YOUR GIT ALIAS]
    Password for 'https://[YOUR GIT ALIAS]@YOUR VSO ACCOUNT].visualstudio.com':[YOUR GENERATED GIT CREDENTIALS]

    cd [YOUR REPO NAME]
    ````

1. Configure Git, add the source files and push them to the remote repo

    ````powershell
    git config --global user.email [YOUR EMAIL]
    git config --global user.name [YOUR NAME]
    git add *
    git commit -m "adding source files"
    git push
    ````

1. In Visual Studio online, verify that you were able to add the files

    ![image](./media/2018-03-21_2-56-53.png)

### Exercise 3 - Create an Azure Container Registry<a name="ex3"></a>

Azure Container Registry will serve as a place to save your container images, which you can later pull and deploy to different environment. For more information on Azure container registry go to (link).

1. Open a browser and Log in to your Azure subscription

1. Click on `Create a new resource` and type `Azure Container Registry` 

    ![image](./media/2018-03-21_0-44-12.png)

1. Click `Create`

    ![image](./media/hol9-1.png)

1. Create a unique registry name and deploy to the existing resource group you have set up for this lab (in this example `AppMigrationACR`)

1. Enable the `Admin user` and select the `Standard SKU`

    ![image](./media/2018-03-21_0-48-02.png)

1. Click `Create`

    ![image](./media/2018-03-21_0-48-02.png)

### Exercise 4 - Create a Build definition<a name="ex4"></a>

1. Log on to your Visual Studio Online account

1. Create a new project (in this example `AppMigrationVSO`)

    ![image](./media/2018-03-21_0-52-22.png)

1. Go to `settings > Services`

    ![image](./media/2018-03-21_0-55-07.png)

1. Click on Create New Service Endpoint. Choose Azure Resource Manager.

    ![image](./media/hol9-3.png)

1. Select the right subscription and the App Migration Resource Group. Click `OK`.

    ![image](./media/hol9-4.png)

1. From the main menu, click on `Builds and Releases > Build`

    ![image](./media/2018-03-21_0-57-40.png)

1. Click on `New definition`

    ![image](./media/hol9-5.png)

1. Pick the current VSTS repo as a source.

    ![image](./media/hol9-6.png)

1. Click on `Start with an Empty Process`.

    ![image](./media/hol9-7.png)

1. Click `Add a task to the phase`.

    ![image](./media/hol9-8.png)

1. Search for `Docker`. Select the Docker result and click `Add`.

    ![image](./media/hol9-9.png)

1. Select the `Build an image` task

1. Select Azure Container Registry as the Container Registry Type. Pick the ACR created in Exercise 1. Leave the action as `Build an image`

1. For the docker file, browse and select the source we uploaded in the previous step

    ![images](./media/2018-03-21_3-06-37.png)

    ![images](./media/hol9-10.png)

1. Add a second Docker task. This time configure action to `Push an image`. Again, make sure to include the Source tags and Latest tag.

    ![images](./media/hol9-11.png)

1. Click on `Save and queue > Save & Queue` to create a build.

    ![image](./media/2018-03-21_1-14-48.png)

1. View the logs for the build and ensure the container is built and pushed to ACR (this may take > 10 mins or more)

### Exercise 5 - Create a Release definition<a name="ex5"></a>

1. Once the build is complete click on `Releases`.

    ![image](./media/hol9-12.png)

1. Click on `Start with an empty process`.

    ![image](./media/hol9-13.png)

1. Click on `View environment tasks`

    ![image](./media/hol9-14.png)

1. Click `Add task` and search for Docker. Select the Docker result.

    ![image](./media/hol9-15.png)

1. Configure the action to Run Docker command and add the following command. Replace the values with your Docker `Host VM name` and your `ACR name`.

    ```powershell
    docker -H tcp://[YOUR DOCKER HOST VM NAME].cloudapp.azure.com:2375 run -d --restart no [YOUR ACR NAME].azurecr.io/appmodernizationsamples:latest -p 8080
    ```

## Summary

In this hands-on lab, you learned how to:

* Upload source to VSO
* Create an Azure Container Registry
* Create a Build definition to create your container and push to an ACR
* Create a Release definition to push your container to a host

----
Copyright 2016 Microsoft Corporation. All rights reserved. Except where otherwise noted, these materials are licensed under the terms of the MIT License. You may use them according to the license as is most appropriate for your project. The terms of this license can be found at https://opensource.org/licenses/MIT.
