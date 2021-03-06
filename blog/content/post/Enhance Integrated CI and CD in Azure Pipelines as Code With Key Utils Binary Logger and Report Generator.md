﻿+++
author = "Rahul Rai"
categories = ["azure", "devops", "compute"]
date = "2019-12-03T00:00:00"
draft = false
tags = ["azure function", "devops", "azure pipelines"]
title = "Enhance Integrated CI and CD in Azure Pipelines as Code With Key Utils: Binary Logger and Report Generator"
type = "post"
+++

If you are using Azure DevOps for building and deploying your .NET core applications, then you should consider the following.

1. [Azure Pipelines](https://azure.microsoft.com/en-au/services/devops/) now supports composing both the build and release stages as code. You can now combine your CI and CD pipeline definitions into a single pipeline definition that lives within the same repository as the application code.
2. Turn on binary logging in MSBuild so that you receive exhaustive structured logs from the build process. You can visualize these logs using [MSBuild Structured Log Viewer](http://msbuildlog.com/) to inspect the build process in great detail.
3. Add [Coverlet](https://github.com/tonerdo/coverlet), which is a cross-platform code coverage framework to collect code coverage data, and generate neat code coverage reports with [Report Generator](https://github.com/danielpalme/ReportGenerator).

To demonstrate the concepts that we discussed, we will build an integrated CI/CD Azure Pipeline which will build and deploy an Azure Function to Azure. Let's first briefly discuss the utilities that I mentioned above.

## MSBuild Binary Log and Viewer

For most of us, the MSBuild process is a black box that chews source code and spits out application binaries. To address concerns with the opaqueness of build logs, MSBuild started supporting [structured logging](https://stackify.com/what-is-structured-logging-and-why-developers-need-it/) since version 15.3. Structured Logs are a complimentary feature to the file and the console loggers, which MSBuild has been supporting for a while now. You can enable binary logging on MSBuild by setting the _/bl_ switch and optionally specifying the name of the log file, which is _build.binlog_ in the following command.

```bash
$ msbuild.exe MySolution.sln /bl:build.binlog
```

The binary log file generated by executing the previous command is not human readable. The [MSBuild Structured Log Viewer](https://msbuildlog.com/) utility helps you visualize the logs and makes navigation through the logs a breeze.

## Coverlet and Report Generator

[Coverlet](https://github.com/tonerdo/coverlet) is a cross-platform tool written in .NET core that measures coverage of unit tests in .NET and .NET core projects. The easiest way to use Coverlet is to include the Coverlet NuGet package in your test projects and use MSBuild to execute the tests. You can use the following commands to perform the two operations.

```bash
$ dotnet add package coverlet.msbuild
$ dotnet test /p:CollectCoverage=true
```

[Report Generator](https://github.com/danielpalme/ReportGenerator) is a visualization tool that takes a code coverage file in a standard format such as Cobertura and Clover and converts it to other useful formats such as HTML. Due to its integration with Azure Pipelines, you will be able to see the code coverage results alongside build results in the same pane.

## Integrated CI/CD Pipeline

Azure DevOps now supports writing [Release pipelines as code](https://azure.microsoft.com/en-au/resources/videos/build-2019-yaml-release-pipelines-in-azure-devops/). You can now compose a single definition that includes both the build and release stages and keep it in the source code. Shortly, we will build a YAML definition that includes both the build and release stages of our application.

## The Sample Application

I created a simple Azure function named _GetLoLz_ that returns a string with as many occurrences of the text "LoL" as requested by setting the query parameter named _count_ of the HTTP request as follows.

```
[GET] http://<hostname>.<domain name>/api/GetLoLz?count=<int>
```

Let's get started.

## Source Code

You can download the source code of the sample application from my GitHub repository. {{< sourceCode src="https://github.com/rahulrai-in/LoLFx" >}}

I used .NET Core 3.0, Visual Studio 2019, and Azure Function v2 to build the application.

## The LoL Function

In Visual Studio (or VSCode), [create a new Function app project](https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-your-first-function-visual-studio) named _AzFunction.LoL_. Create a new class file and add the following HTTP function to your project.

```cs
[FunctionName(nameof(GetLoLz))]
public static IActionResult GetLoLz(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get")]
    HttpRequest request,
    ILogger log)
{

    log.LogInformation("C# HTTP trigger function processed a request.");
    return int.TryParse(request.Query["count"], out var count)
        ? (ActionResult)new OkObjectResult($"{string.Join(string.Empty, Enumerable.Repeat("LoL.", count))}")
        : new BadRequestObjectResult("Please pass a count on the query string");
}
```

Launch the application now and visit the following URL from your browser or send a GET request from an API client such as Postman. Since your application might be available at a different port, check the Azure Function Tool CLI console for the address of your HTTP function.

```
http://localhost:7071/api/GetLoLz?count=1000
```

Let's now add a few tests for the function that you just created. Create a xUnit test project in your solution and add the _coverlet.msbuild_ and _coverlet.collector_ NuGet packages to it. Now, add a few tests to the project. For details, check out the _AzFunction.LoL.Tests_ project in the GitHub repository of this application.

## Provision Azure Resources

Let's use the [AZ CLI](https://docs.microsoft.com/en-us/cli/azure/) utility to create a Resource Group and an Azure Function App for our project. The following commands will create a Resource Group named _FxDemo_, a storage account named _lolfxstorage_, and an Azure Function named _LoLFx_. You can choose other names for your resources and substitute them in the commands below. Remember to use the command _az login_ to log in before executing the following commands.

```bash
$ az group create -l australiaeast -n FxDemo
$ az storage account create -n lolfxstorage -g FxDemo -l australiaeast --sku Standard\_LRS
$ az functionapp create --resource-group FxDemo --consumption-plan-location australiaeast --name LoLFx --storage-account lolfxstorage --runtime dotnet --os-type Linux
```

Log in to the Azure portal and verify the status of all the resources. If all of them are up and running, then you are good to go.

## Composing CI/CD Pipeline - Build Stage

Azure Pipelines automatically picks the build and release definition from the _azure-pipelines.yml_ file at the root of your repository. You can store your code in any popular version control system such as GitHub and use Azure Pipelines to build, test, and deploy the application. Use [this guide](https://docs.microsoft.com/en-us/azure/devops/pipelines/ecosystems/dotnet-core) to connect your version control system to Azure DevOps. We will now define our integrated build and release pipeline.

Create a file named _azure-pipelines.yml_ at the root of the repository and add the following code to it.

```json
trigger:
  branches:
    include:
      - '\*'
stages:
  - stage: Build
    jobs:
      - job: Build\_Linux
        pool:
          vmImage: ubuntu-18.04
        variables:
          buildConfiguration: Release
        steps:
          - template: .azure-pipelines/restore-packages.yml
          - template: .azure-pipelines/build.yml
          - template: .azure-pipelines/test.yml
          - template: .azure-pipelines/publish.yml
  - stage: Deploy
    jobs:
      - job: Deploy\_Fx
        pool:
          vmImage: ubuntu-18.04
        variables:
          azureSubscription: LoLFxConnection
          appName: LolFx
        steps:
          - template: .azure-pipelines/download-artifacts.yml
          - template: .azure-pipelines/deploy.yml
```

I like to adopt the Single Responsibility Principle (SRP) everywhere in my application code. Azure Pipelines extends SRP to DevOps by giving you the flexibility to write the definition of individual steps that make up your pipeline in independent files. You can reference the files that make up the pipeline in the _azure-pipelines.yml_ file to compose the complete pipeline.

You can see in the previous definition that this pipeline will be triggered whenever Azure Pipelines detect a commit on any branch of the repository. Next, we defined two stages in the pipeline named _Build_ and _Deploy_ that will kick-off to build and deploy the application respectively. We used the same type of agent (VM with Ubuntu image) to carry out the tasks of building and deploying our application. Even if you are using an XPlat framework such as .NET core, you should always use agents that have the same OS platform as your application servers. A common platform ensures that tests can verify the behavior of your application on the actual platform, and any platform-related issues get surfaced immediately.

As you can see that I have stored all the templates in a folder named _.azure-pipelines_ at the root of the repository. Let's go through the templates that we used for the build stage. The first step that will execute is _restore-packages,_ which will restore the NuGet packages of the solution.

```json
steps:
- task: DotNetCoreCLI@2
  displayName: Restore Nuget
  inputs:
   command: restore
```

The next step in the sequence is _build_, which will build the solution. We will set the _/bl_ flag to instruct MSBuild to generate a binary log. Since we want to investigate this log file later in case of issues, we will publish the file that contains the binary log as a build artifact.

```json
steps:
- task: MSBuild@1
  displayName: Build - $(buildConfiguration)
  inputs:
    configuration: $(buildConfiguration)
    solution: AzFunction.sln
    clean: true
    msbuildArguments: /bl:"$(Build.SourcesDirectory)/BuildLog/build.binlog"
- task: PublishPipelineArtifact@1
  displayName: Publish BinLog
  inputs:
    path: $(Build.SourcesDirectory)/BuildLog
    artifact: BuildLog
```

The next step of this stage is _test_, which will execute all the tests. There are three individual tasks in this stage. The first task executes the tests in the _AzFunction.LoL.Tests_ project with necessary flags set that generates code coverage report in Cobertura format in a folder named _Coverage_. The next step installs the Report Generator tool. The Report Generator tool takes the test report and converts it to an HTML report that would be visible in the Azure Pipelines dashboard. The final task publishes the Cobertura report that you can download later.

```json
steps:
- task: DotNetCoreCLI@2
  displayName: Run Tests
  inputs:
    command: test
    projects: 'AzFunction.LoL.Tests/AzFunction.LoL.Tests.csproj'
    arguments: '--configuration $(buildConfiguration) /p:CollectCoverage=true /p:CoverletOutputFormat=cobertura /p:CoverletOutput=$(Build.SourcesDirectory)/TestResults/Coverage/'
- script: |
   dotnet tool install dotnet-reportgenerator-globaltool --tool-path .
   ./reportgenerator -reports:$(Build.SourcesDirectory)/TestResults/Coverage/\*\*/coverage.cobertura.xml -targetdir:$(Build.SourcesDirectory)/CodeCoverage -reporttypes:'HtmlInline\_AzurePipelines;Cobertura'
  displayName: Create Code Coverage Report
- task: PublishCodeCoverageResults@1
  displayName: Publish Code Coverage Report
  inputs:
    codeCoverageTool: Cobertura
    summaryFileLocation: '$(Build.SourcesDirectory)/CodeCoverage/Cobertura.xml'
    reportDirectory: '$(Build.SourcesDirectory)/CodeCoverage'
```

The final step in the build stage is _publish_. This step is responsible for publishing the application as an artifact that will be used by the next stage, which is responsible for deploying the published artifact. The following code listing presents the three tasks responsible for publishing the build artifact.

```json
steps:
- task: DotNetCoreCLI@2
  displayName: Publish App
  inputs:
    command: publish
    publishWebProjects: false
    arguments: '-c $(buildConfiguration) -o out --no-build --no-restore'
    zipAfterPublish: false
    modifyOutputPath: false
    workingDirectory: AzFunction.LoL
- task: CopyFiles@2
  displayName: Copy App Output to Staging Directory
  inputs:
    sourceFolder: AzFunction.LoL/out
    contents: '\*\*/\*'
    targetFolder: $(Build.ArtifactStagingDirectory)/function
- task: PublishPipelineArtifact@1
  displayName: Publish Artifact
  inputs:
    path: $(Build.ArtifactStagingDirectory)/function
    artifactName: Function
```

In the previous listing, the first step executes the _dotnet publish_ command and emits the application binaries in a folder named _out_. The next step copies the contents of the _out_ folder to another folder named _function_ in the _ArtifactStagingDirectory_. The final step publishes the _function_ folder as a build artifact so that the contents of this folder can be deployed to Azure Function by the next stage.

## Adding ARM Service Connection

Before we discuss the _Deploy_ stage of the pipeline, we will take a detour to connect our Azure Pipeline to our Azure subscription to deploy our function.

Azure Pipelines allows you to define [service connections](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints) so that you don't have to store external service credentials and nuances of connecting to an external service within the tasks of your Azure Pipelines. Let's add a service connection to Azure Resource Manager so that we can reference it within our release pipeline.

The service connection wizard is available in the _Service Connection_ page inside the _Project Settings_ page. You can refer to the documentation of [service connection](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints) to find your way to this wizard. I will walk you through the steps that I took for creating an Azure Resource Manager service connection to my subscription. In the service connection wizard, select Azure Resource Manager as the service that we want to connect with.

{{< img src="/Create Resource Manager Service Connection.png" alt="Create New Service Connection" >}}

Next, we need to provide a service principal or generate a Managed Identify in Azure AD, which will allow this service connection to connect to our Azure subscription. Select the _Service principal (automatic)_ option so that the wizard can generate and store the service principal automatically.

{{< img src="/Generate Service Principal.png" alt="Generate Service Principal" >}}

In the next step of the wizard, set the scope of connection to _Subscription_. After selecting the option, the wizard will populate the next input controls with subscriptions and resource groups that you can access. After selecting the subscription and resource group that you created previously, enter a name for this connection. The name that you assign to the connection will be used in the pipeline to refer to the service connection, which will abstract all the connection nuances from the task. I named my service connection _LoLFxConnection_.

{{< img src="/Set Connection Details.png" alt="Set Connection Details" >}}

After saving the information, we will be able to use the name of the connection _LoLFxConnection_ to connect to Azure from our pipeline.

## Composing CI/CD Pipeline - Deploy Stage

Let's discuss the steps that make up the _Deploy_ stage of our pipeline. The first step in this stage is _download-artifact,_ which downloads the _Function_ artifact from the artifact produced by the _Build_ stage.

```json
steps:
- task: DownloadPipelineArtifact@2
  displayName: Download Build Artifacts
  inputs:
    buildType: current
    downloadType: single
    downloadPath: '$(System.ArtifactsDirectory)'
    artifactName: Function
```

Finally, the last step in the pipeline named _deploy_ is responsible for publishing the release artifact to Azure function.

```json
steps:
- task: AzureFunctionApp@1
  displayName: Azure Function App Deploy
  inputs:
    azureSubscription: LoLFxConnection
    appName: '$(appName)'
    package: '$(System.ArtifactsDirectory)'
```

Note that we referenced the Azure subscription using the name of the service connection that we created previously.

We are done with the setup now. Commit the changes and push the branch to GitHub so that Azure Pipelines can initiate the CI/CD process.

## Report Generator in Action

After the pipeline that we defined runs to completion, navigate to the Azure Pipelines dashboard to verify whether both the stages defined in our pipeline succeeded.

{{< img src="/Build and Release Stages Succeeded.png" alt="Build and Release Stages Succeeded" >}}

After the _Build_ stage of the pipeline completes, you will find that a new tab named _Code Coverage_ appears on the Azure Pipelines dashboard. The Report Generator tool generated the report that appears under this tab.

{{< img src="/Code Coverage Report.png" alt="Code Coverage Report" >}}

Let's now download the binary logs generated by the build and study it with the MSBuild Log Viewer tool.

## Binary Log in Action

Navigate to the Artifacts published by the Pipeline and download the _build.binlog_ file produced by the build stage of the pipeline.

{{< img src="/Download Binlog.png" alt="Download Binlog" >}}

You can open this file with the MSBuild Log Viewer tool to investigate the build process. Apart from assembly reference conflicts, and the build process, you will be able to see all the Environment variables of the build server in this report and list the build tasks that take the most amount of time.

{{< img src="/MSBuild Log Viewer.png" alt="MSBuild Log Viewer" >}}

Build logs are critical for assessing issues with assembly conflicts, and diagnosing why the build is taking a long time on the server.

## The LoLFx Function

I will use cURL to send a request to the function that the pipeline deployed to my Azure Function resource. Following is the command and the output generated by it.

```bash
$ curl https://lolfx.azurewebsites.net/api/GetLoLz?count=10

LoL.LoL.LoL.LoL.LoL.LoL.LoL.LoL.LoL.LoL.
```

To conclude, we went through the steps to compose an integrated CI/CD pipeline that builds and deploys an Azure Function. We also discussed a few essential utilities that can add value to your CI/CD pipeline viz. Binary logging, MSBuild Log Viewer, Coverlet, and Report Generator. I hope this article encourages you to evangelize these key features of Azure DevOps.

I am working on a soon to be released **FREE** developer quick start guide on the Istio Service Mesh. Follow me on Twitter [@rahulrai_in](https://twitter.com/rahulrai_in) or subscribe to this blog to access it before everyone else.

{{< subscribe >}}
