<properties
   pageTitle="Service Fabric App upgrade using PowerShell| Microsoft Azure"
   description="This article walks through the experience of deploying a Service Fabric application, changing the code, and rolling out an upgrade using PowerShell."
   services="service-fabric"
   documentationCenter=".net"
   authors="mani-ramaswamy"
   manager="timlt"
   editor=""/>

<tags
   ms.service="service-fabric"
   ms.devlang="dotnet"
   ms.topic="article"
   ms.tgt_pltfrm="NA"
   ms.workload="NA"
   ms.date="07/17/2015"
   ms.author="subramar"/>



# Service Fabric Application Upgrade using PowerShell

The most frequently used and recommended upgrade approach is the monitored rolling upgrade.  Service Fabric monitors the health of the application being upgraded based on a set of health policies. When the applications in an upgrade domain (UD) have been upgraded, Service Fabric evaluates the application health and determines whether to proceed to the next upgrade domain or fail the upgrade based on the health policies. A monitored application upgrade can be performed using the managed or native APIs, PowerShell, or REST. For instructions on performing an upgrade using Visual Studio, please see [uprading your application using Visual Studio](service-fabric-application-upgrade-tutorial.md).

Service Fabric monitored rolling upgrade allows the application administrator to configure the health evaluation policy that Service Fabric uses to determine the application is healthy. In addition, it also allows the administrator to configure the action to be taken when the health evaluation fails such as automatically roll-back. This section walkthrough a monitored upgrade for one of the SDK samples using PowerShell.

## Step 1: Build and deploy the Visual Objects sample

These steps can be done by downloading the application from github, and adding the **webgl-utils.js** and **gl-matrix-min.js** files into the project as mentioned in the sample's readme file. Without that the application will not work. After adding these to the project, build and publish the application by right clicking on the application project, **VisualObjectsApplication** and selecting the publish command in the Service Fabric menu item as follows.  See [Service Fabric Application Upgrade Tutorial](service-fabric-application-tutorial.md) for more information.  Alternatively, one may use PowerShell to deploy your application.

> [AZURE.NOTE] Before any of the Service Fabric commands may be used in PowerShell, one has to first connect to the cluster by using the `Connect-ServiceFabricCluster` cmdlet. Similarly, it is assumed that the Cluster has already been setup on your local machine. See the article on [setting up your Service Fabric development environment](service-fabric-get-started.md).

After building the project in Visual Studio, one may use the PowerShell command **Copy-ServiceFabricApplicationPackage** to copy the application package to the ImageStore, followed by registering the application to the Service Fabric runtime by using the **Register-ServiceFabricApplicationPackage** cmdlet, and finally starting an instance of the application by using the **New-ServiceFabricApplication** cmdlet.  These three steps are analogous to using the Deploy menu item in Visual Studio.

Now, you can use [Service Fabric Explorer to view the cluster and the application](service-fabric-visualizing-your-cluster.md). The application has a web service that can be navigated to in Internet Explorer by typing [http://localhost:8081/visualobjects](http://localhost:8081/visualobjects) in the address bar.  You should see some floating visual objects moving around in the screen.  Additionally, one may use **Get-ServiceFabricApplication** to check the application status.

## Step 2: Update the Visual Objects sample

You might notice that with the version that was deployed in Step 1, the visual objects do not rotate. Let us upgrade this application to one where the visual objects also rotate.

Select the VisualObjects.ActorService project within the VisualObjects solution, and open the StatefulVisualObjectActor.cs file. Within that file navigate to the method `MoveObject`, and comment out `this.State.Move()` and uncomment `this.State.Move(true)`. This change will make the objects rotate after the service is upgraded.

We also need to update the *ServiceManifest.xml* file (under PackageRoot) of the project **VisualObjects.ActorService**. Update the *CodePackage* and the service version to 2.0, and the corresponding lines in the *ServiceManifest.xml* file.
You can use the Visual Studio *Edit Manifest Files* option after you right-click on the solution to make the manifest file changes - see [Service Fabric Application Upgrade Tutorial](service-fabric-application-tutorial.md).


After the changes are made, the manifest should look like the following (highlighted portions show the changes):

```xml
<ServiceManifestName="VisualObjects.ActorService"Version="2.0"xmlns="http://schemas.microsoft.com/2011/01/fabric"xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

<CodePackageName="Code"Version="2.0">
```

Now, we need to update the *ApplicationManifest.xml* file (found under the **VisualObjects** project under the **VisualObjects** solution) to use version 2.0 of the **VisualObjects.ActorService** project and also update the Application version to 2.0.0.0 from 1.0.0.0. Now, the corresponding lines in the *ApplicationManifest.xml* should read like the following:

```xml
<ApplicationManifestxmlns:xsd="http://www.w3.org/2001/XMLSchema"xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"ApplicationTypeName="VisualObjects"ApplicationTypeVersion="2.0.0.0"xmlns="http://schemas.microsoft.com/2011/01/fabric">

 <ServiceManifestRefServiceManifestName="VisualObjects.ActorService"ServiceManifestVersion="2.0" />
```


Now, build the project by selecting just the **ActorService** project and right clicking and selecting build in Visual Studio (if you select Rebuild all, you might have to update the versions for the other projects as well in its *ServiceManifest.xml* and in the *ApplicationManifest.xml*, since the code would have changed). Now let's package the updated application by right clicking on *VisualObjectsApplication*, selecting the Service Fabric Menu and choosing Package. This should create an application package that can be deployed.  Your updated application is ready to be deployed now.


## Step 3:  Decide on health policies and upgrade parameters

Please familiarize yourself with the [application upgrade parameters](service-fabric-application-upgrade-parameters.md) and the [upgrade process](service-fabric-application-upgrade.md) to get a good understanding of the various upgrade parameters, timeouts and health criterion applied. For this walkthrough, we will leave the service health evaluation criterion to be the default (and recommended values) which mean that all services and instances should be _healthy_ after the upgrade.  Let's however increase the *HealthCheckStableDuration* to 60 seconds (so that the services will be healthy for at least 20 seconds before the upgrade proceeds to the next upgrade domain).  Let's also set the *UpgradeDomainTimeout* to be 1200 seconds and the *UpgradeTimeout* to be 3000 seconds. Finally, let's also set the *UpgradeFailureAction* to rollback thereby requesting that Service Fabric rollback the application to the previous version if it encounters any issues during upgrade. Thus, the upgrade parameters we will specify when starting the upgrade (in Step 4) will be the following:

FailureAction = Rollback

HealthCheckStableDurationSec = 60

UpgradeDomainTimeoutSec = 1200

UpgradeTimeout = 3000


## Step 4: Prepare application for upgrade

Now, the application is built and ready to be upgraded. If you open up a PowerShell window as administrator and type **Get-ServiceFabricApplication**, it should let you know that it is Application Type 1.0.0.0 of **VisualObjects** that's been deployed.  The application package is stored under the following relative path where you uncompressed the Service Fabric SDK - *Samples\Services\Stateful\VisualObjects\VisualObjects\obj\x64\Debug*. You should find a "Package" folder in that directory - this is where the application package is stored. Please check the timestamps to ensure that it is the latest build (and you may need to modify the paths appropriately as well).

Now let's copy the updated application package to the Service Fabric ImageStore (where the application packages are stored by Service Fabric). The parameter *ApplicationPackagePathInImageStore* informs Service Fabric where it can find the application package. We have put the updated application in "VisualObjects\_V2" with the following command (you may have to modify paths again appropriately).

```powershell
Copy-ServiceFabricApplicationPackage  -ApplicationPackagePath .\Samples\Services\Stateful\VisualObjects\VisualObjects\obj\x64\Debug\Package
-ImageStoreConnectionString fabric:ImageStore   -ApplicationPackagePathInImageStore "VisualObjects\_V2"
```

The next step is to register this application with Service Fabric, which can be performed using the following command:

```powershell
Register-ServiceFabricApplicationType -ApplicationPathInImageStore "VisualObjects\_V2"
```

If the above command doesn't succeed it is likely that you need a rebuild of all services. As mentioned in Step 2, you may have to update your WebService version as well.

## Step 5: Start the application upgrade

Now, we are all set to start the application upgrade by using the following command:

```powershell
Start-ServiceFabricApplicationUpgrade -ApplicationName fabric:/VisualObjects -ApplicationTypeVersion 2.0.0.0 -HealthCheckStableDurationSec 60 -UpgradeDomainTimeoutSec 1200 -UpgradeTimeout 3000   -FailureAction Rollback -Monitored
```


Note the application name is as was described in the *ApplicationManifest.xml* file. Service Fabric uses this name to identify which application is getting upgraded. If you set the timeouts to be too short, you may encounter a failure message that states the problem. Refer to the troubleshooting section, or increase the timeouts.

Now, as the application upgrade proceeds, you can monitor it using Service Fabric Explorer, or using the following PowerShell command: **Get-ServiceFabricApplicationUpgrade fabric:/VisualObjects**.

In a few minutes, the status using the above PowerShell command should state that all upgrade domains were upgraded (completed). And you should find that the visual objects in your browser window will now have started rotating!

You may want to try changing the versions and moving from version 2 to version 3 as an exercise, or even from version 2 back to version 1 (yes, you can upgrade from v2 to v1). Play with timeouts and health policies to make yourself familiar. When you are deploying to an Azure cluster, the parameters used will be different than those that worked when deploying to a local cluster - it is recommended to set the timeouts conservatively.


## Next steps

[Uprading your Application Using Visual Studio](service-fabric-application-upgrade-tutorial.md) walks you through an application upgrade using Visual Studio.

Control how your application upgrades by using [Upgrade Parameters](service-fabric-application-upgrade-parameters.md).

Make your application upgrades compatible by learning how to use [Data Serialization](service-fabric-application-upgrade-data-serialization.md).

Learn how to use advanced functionality while upgrading your application by referring to [Advanced Topics](service-fabric-application-upgrade-advanced.md).

Fix common problems in application upgrades by referring to the steps in [Troubleshooting Application Upgrades ](service-fabric-application-upgrade-troubleshooting.md).

