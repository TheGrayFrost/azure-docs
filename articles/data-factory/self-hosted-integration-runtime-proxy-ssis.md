---
title: Configure a self-hosted integration runtime as a proxy for SSIS
description: Learn how to configure a self-hosted integration runtime as a proxy for an Azure-SSIS Integration Runtime. 
ms.service: data-factory
ms.subservice: integration-services
ms.topic: conceptual
author: swinarko
ms.author: sawinark
ms.custom: seo-lt-2019, devx-track-azurepowershell
ms.date: 07/19/2021
---

# Configure a self-hosted IR as a proxy for an Azure-SSIS IR in Azure Data Factory

[!INCLUDE[appliesto-adf-xxx-md](includes/appliesto-adf-xxx-md.md)]

This article describes how to run SQL Server Integration Services (SSIS) packages on an Azure-SSIS Integration Runtime (Azure-SSIS IR) in Azure Data Factory (ADF) with a self-hosted integration runtime (self-hosted IR) configured as a proxy. 

With this feature, you can access data and run tasks on premises without having to [join your Azure-SSIS IR to a virtual network](./join-azure-ssis-integration-runtime-virtual-network.md). The feature is useful when your corporate network has a configuration too complex or a policy too restrictive for you to inject your Azure-SSIS IR into it.

This feature can only be enabled on SSIS Data Flow Task and Execute SQL/Process Tasks for now. 

Enabled on Data Flow Task, this feature will break it down into two staging tasks whenever applicable: 
* **On-premises staging task**: This task runs your data flow component that connects to an on-premises data store on your self-hosted IR. It moves data from the on-premises data store into a staging area in your Azure Blob Storage or vice versa.
* **Cloud staging task**: This task runs your data flow component that doesn't connect to an on-premises data store on your Azure-SSIS IR. It moves data from the staging area in your Azure Blob Storage to a cloud data store or vice versa.

If your Data Flow Task moves data from on premises to cloud, then the first and second staging tasks will be on-premises and cloud staging tasks, respectively. If your Data Flow Task moves data from cloud to on premises, then the first and second staging tasks will be cloud and on-premises staging tasks, respectively. If your Data Flow Task moves data from on premises to on premises, then the first and second staging tasks will be both on-premises staging tasks. If your Data Flow Task moves data from cloud to cloud, then this feature isn't applicable.

Enabled on Execute SQL/Process Tasks, this feature will run them on your self-hosted IR. 

Other benefits and capabilities of this feature allow you to, for example, set up your self-hosted IR in regions that are not yet supported by an Azure-SSIS IR, and allow the public static IP address of your self-hosted IR on the firewall of your data sources.

## Prepare the self-hosted IR

To use this feature, you first create a data factory and set up an Azure-SSIS IR in it. If you have not already done so, follow the instructions in [Set up an Azure-SSIS IR](./tutorial-deploy-ssis-packages-azure.md).

You then set up your self-hosted IR in the same data factory where your Azure-SSIS IR is set up. To do so, see [Create a self-hosted IR](./create-self-hosted-integration-runtime.md).

Finally, you download and install the latest version of self-hosted IR, as well as the additional drivers and runtime, on your on-premises machine or Azure virtual machine (VM), as follows:
- Download and install the latest version of [self-hosted IR](https://www.microsoft.com/download/details.aspx?id=39717).
- If you use Object Linking and Embedding Database (OLEDB), Open Database Connectivity (ODBC), or ADO.NET connectors in your packages, download and install the relevant drivers on the same machine where your self-hosted IR is installed, if you haven't done so already.  

  If you use the earlier version of the OLEDB driver for SQL Server (SQL Server Native Client [SQLNCLI]), [download the 64-bit version](https://www.microsoft.com/download/details.aspx?id=50402).  

  If you use the latest version of OLEDB driver for SQL Server (MSOLEDBSQL), [download the 64-bit version](https://www.microsoft.com/download/details.aspx?id=56730).  
  
  If you use OLEDB/ODBC/ADO.NET drivers for other database systems, such as PostgreSQL, MySQL, Oracle, and so on, you can download the 64-bit versions from their websites.
- If you use data flow components from Azure Feature Pack in your packages, [download and install Azure Feature Pack for SQL Server 2017](https://www.microsoft.com/download/details.aspx?id=54798) on the same machine where your self-hosted IR is installed, if you haven't done so already.
- If you haven't done so already, [download and install the 64-bit version of Visual C++ (VC) runtime](https://www.microsoft.com/en-us/download/details.aspx?id=40784) on the same machine where your self-hosted IR is installed.

### Enable Windows authentication for on-premises tasks

If on-premises staging tasks and Execute SQL/Process Tasks on your self-hosted IR require Windows authentication, you must also [configure Windows authentication feature on your Azure-SSIS IR](/sql/integration-services/lift-shift/ssis-azure-connect-with-windows-auth). 

Your on-premises staging tasks and Execute SQL/Process Tasks will be invoked with the self-hosted IR service account (*NT SERVICE\DIAHostService*, by default), and your data stores will be accessed with the Windows authentication account. Both accounts require certain security policies to be assigned to them. On the self-hosted IR machine, go to **Local Security Policy** > **Local Policies** > **User Rights Assignment**, and then do the following:

1. Assign the *Adjust memory quotas for a process* and *Replace a process level token* policies to the self-hosted IR service account. This should occur automatically when you install your self-hosted IR with the default service account. If it doesn't, assign those policies manually. If you use a different service account, assign the same policies to it.

1. Assign the *Log on as a service* policy to the Windows Authentication account.

## Prepare the Azure Blob Storage linked service for staging

If you haven't already done so, create an Azure Blob Storage linked service in the same data factory where your Azure-SSIS IR is set up. To do so, see [Create an Azure Data Factory linked service](./quickstart-create-data-factory-portal.md#create-a-linked-service). Be sure to do the following:
- For **Data Store**, select **Azure Blob Storage**.  
- For **Connect via integration runtime**, select **AutoResolveIntegrationRuntime** (not your self-hosted IR), so we can ignore it and use your Azure-SSIS IR instead to fetch access credentials for your Azure Blob Storage.
- For **Authentication method**, select **Account key**, **SAS URI**, **Service Principal**, **Managed Identity**, or **User-Assigned Managed Identity**.  

>[!TIP]
>If you select the **Service Principal** method, grant your service principal at least a *Storage Blob Data Contributor* role. For more information, see [Azure Blob Storage connector](connector-azure-blob-storage.md#linked-service-properties). If you select the **Managed Identity**/**User-Assigned Managed Identity** method, grant the specified system/user-assigned managed identity for your ADF a proper role to access Azure Blob Storage. For more information, see [Access Azure Blob Storage using Azure Active Directory (Azure AD) authentication with the specified system/user-assigned managed identity for your ADF](/sql/integration-services/connection-manager/azure-storage-connection-manager#managed-identities-for-azure-resources-authentication).

![Prepare the Azure Blob storage-linked service for staging](media/self-hosted-integration-runtime-proxy-ssis/shir-azure-blob-storage-linked-service.png)

## Configure an Azure-SSIS IR with your self-hosted IR as a proxy

Having prepared your self-hosted IR and Azure Blob Storage linked service for staging, you can now configure your new or existing Azure-SSIS IR with the self-hosted IR as a proxy in your data factory portal or app. Before you do so, though, if your existing Azure-SSIS IR is already running, you can stop, edit, and then restart it.

1. In the **Integration runtime setup** pane, skip past the **General settings** and **Deployment settings** pages by selecting the **Continue** button. 

1. On the **Advanced settings** page, do the following:

   1. Select the **Set up Self-Hosted Integration Runtime as a proxy for your Azure-SSIS Integration Runtime** check box. 

   1. In the **Self-Hosted Integration Runtime** drop-down list, select your existing self-hosted IR as a proxy for the Azure-SSIS IR.

   1. In the **Staging storage linked service** drop-down list, select your existing Azure Blob Storage linked service or create a new one for staging.

   1. In the **Staging path** box, specify a blob container in your selected Azure Storage account or leave it empty to use a default one for staging.

   1. Select the **Continue** button.

   ![Advanced settings with a self-hosted IR](./media/tutorial-create-azure-ssis-runtime-portal/advanced-settings-shir.png)

You can also configure your new or existing Azure-SSIS IR with the self-hosted IR as a proxy by using PowerShell.

```powershell
$ResourceGroupName = "[your Azure resource group name]"
$DataFactoryName = "[your data factory name]"
$AzureSSISName = "[your Azure-SSIS IR name]"
# Self-hosted integration runtime info - This can be configured as a proxy for on-premises data access 
$DataProxyIntegrationRuntimeName = "" # OPTIONAL to configure a proxy for on-premises data access 
$DataProxyStagingLinkedServiceName = "" # OPTIONAL to configure a proxy for on-premises data access 
$DataProxyStagingPath = "" # OPTIONAL to configure a proxy for on-premises data access 

# Add self-hosted integration runtime parameters if you configure a proxy for on-premises data access
if(![string]::IsNullOrEmpty($DataProxyIntegrationRuntimeName) -and ![string]::IsNullOrEmpty($DataProxyStagingLinkedServiceName))
{
    Set-AzDataFactoryV2IntegrationRuntime -ResourceGroupName $ResourceGroupName `
        -DataFactoryName $DataFactoryName `
        -Name $AzureSSISName `
        -DataProxyIntegrationRuntimeName $DataProxyIntegrationRuntimeName `
        -DataProxyStagingLinkedServiceName $DataProxyStagingLinkedServiceName

    if(![string]::IsNullOrEmpty($DataProxyStagingPath))
    {
        Set-AzDataFactoryV2IntegrationRuntime -ResourceGroupName $ResourceGroupName `
            -DataFactoryName $DataFactoryName `
            -Name $AzureSSISName `
            -DataProxyStagingPath $DataProxyStagingPath
    }
}
Start-AzDataFactoryV2IntegrationRuntime -ResourceGroupName $ResourceGroupName `
    -DataFactoryName $DataFactoryName `
    -Name $AzureSSISName `
    -Force
```

## Enable SSIS packages to use a proxy

By using the latest SSDT as either the SSIS Projects extension for Visual Studio or a standalone installer, you can find a new `ConnectByProxy` property in the connection managers for supported data flow components and `ExecuteOnProxy` property in Execute SQL/Process Tasks.
* [Download the SSIS Projects extension for Visual Studio](https://marketplace.visualstudio.com/items?itemName=SSIS.SqlServerIntegrationServicesProjects)
* [Download the standalone installer](/sql/ssdt/download-sql-server-data-tools-ssdt#ssdt-for-vs-2017-standalone-installer)   

When you design new packages containing Data Flow Tasks with components that access data on premises, you can enable the `ConnectByProxy` property by setting it to *True* in the **Properties** pane of relevant connection managers.

When you design new packages containing Execute SQL/Process Tasks that run on premises, you can enable the `ExecuteOnProxy` property by setting it to *True* in the **Properties** pane of relevant tasks themselves.

![Enable ConnectByProxy/ExecuteOnProxy property](media/self-hosted-integration-runtime-proxy-ssis/shir-proxy-properties.png)

You can also enable the `ConnectByProxy`/`ExecuteOnProxy` properties when you run existing packages, without having to manually change them one by one. There are two options:
- **Option A**: Open, rebuild, and redeploy the project containing those packages with the latest SSDT to run on your Azure-SSIS IR. You can then enable the `ConnectByProxy` property by setting it to *True* for the relevant connection managers that appear on the **Connection Managers** tab of **Execute Package** pop-up window when you're running packages from SSMS.

  ![Enable ConnectByProxy/ExecuteOnProxy property2](media/self-hosted-integration-runtime-proxy-ssis/shir-connection-managers-tab-ssms.png)

  You can also enable the `ConnectByProxy` property by setting it to *True* for the relevant connection managers that appear on the **Connection Managers** tab of [Execute SSIS Package activity](./how-to-invoke-ssis-package-ssis-activity.md) when you're running packages in Data Factory pipelines.
  
  ![Enable ConnectByProxy/ExecuteOnProxy property3](media/self-hosted-integration-runtime-proxy-ssis/shir-connection-managers-tab-ssis-activity.png)

- **Option B:** Redeploy the project containing those packages to run on your SSIS IR. You can then enable the `ConnectByProxy`/`ExecuteOnProxy` properties by providing their property paths, `\Package.Connections[YourConnectionManagerName].Properties[ConnectByProxy]`/`\Package\YourExecuteSQLTaskName.Properties[ExecuteOnProxy]`/`\Package\YourExecuteProcessTaskName.Properties[ExecuteOnProxy]`, and setting them to *True* as property overrides on the **Advanced** tab of **Execute Package** pop-up window when you're running packages from SSMS.

  ![Enable ConnectByProxy/ExecuteOnProxy property4](media/self-hosted-integration-runtime-proxy-ssis/shir-advanced-tab-ssms.png)

  You can also enable the `ConnectByProxy`/`ExecuteOnProxy` properties by providing their property paths, `\Package.Connections[YourConnectionManagerName].Properties[ConnectByProxy]`/`\Package\YourExecuteSQLTaskName.Properties[ExecuteOnProxy]`/`\Package\YourExecuteProcessTaskName.Properties[ExecuteOnProxy]`, and setting them to *True* as property overrides on the **Property Overrides** tab of [Execute SSIS Package activity](./how-to-invoke-ssis-package-ssis-activity.md) when you're running packages in Data Factory pipelines.
  
  ![Enable ConnectByProxy/ExecuteOnProxy property5](media/self-hosted-integration-runtime-proxy-ssis/shir-property-overrides-tab-ssis-activity.png)

## Debug the on-premises tasks and cloud staging tasks

On your self-hosted IR, you can find the runtime logs in the *C:\ProgramData\SSISTelemetry* folder and the execution logs of on-premises staging tasks and Execute SQL/Process Tasks in the *C:\ProgramData\SSISTelemetry\ExecutionLog* folder. You can find the execution logs of cloud staging tasks in your SSISDB, specified logging file paths, or Azure Monitor depending on whether you store your packages in SSISDB, enable [Azure Monitor integration](./monitor-using-azure-monitor.md#monitor-ssis-operations-with-azure-monitor), etc. You can also find the unique IDs of on-premises staging tasks in the execution logs of cloud staging tasks. 

![Unique ID of the first staging task](media/self-hosted-integration-runtime-proxy-ssis/shir-first-staging-task-guid.png)

If you've raised customer support tickets, you can select the **Send logs** button on **Diagnostics** tab of **Microsoft Integration Runtime Configuration Manager** that's installed on your self-hosted IR to send recent operation/execution logs for us to investigate.

## Billing for the on-premises tasks and cloud staging tasks

The on-premises staging tasks and Execute SQL/Process Tasks that run on your self-hosted IR are billed separately, just as any data movement activities that run on a self-hosted IR are billed. This is specified in the [Azure Data Factory data pipeline pricing](https://azure.microsoft.com/pricing/details/data-factory/data-pipeline/) article.

The cloud staging tasks that run on your Azure-SSIS IR are not be billed separately, but your running Azure-SSIS IR is billed as specified in the [Azure-SSIS IR pricing](https://azure.microsoft.com/pricing/details/data-factory/ssis/) article.

## Enable custom/3rd party data flow components 

To enable your custom/3rd party data flow components to access data on premises using self-hosted IR as a proxy for Azure-SSIS IR, follow these instructions:

1. Install your custom/3rd party data flow components targeting SQL Server 2017 on Azure-SSIS IR via [standard/express custom setups](./how-to-configure-azure-ssis-ir-custom-setup.md).

1. Create the following DTSPath registry keys on self-hosted IR if they don’t exist already:
   1. `Computer\HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Microsoft SQL Server\140\SSIS\Setup\DTSPath` set to `C:\Program Files\Microsoft SQL Server\140\DTS\`
   1. `Computer\HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Microsoft\Microsoft SQL Server\140\SSIS\Setup\DTSPath` set to `C:\Program Files (x86)\Microsoft SQL Server\140\DTS\`
   
1. Install your custom/3rd party data flow components targeting SQL Server 2017 on self-hosted IR under the DTSPath above and ensure that your installation process:

   1. Creates `<DTSPath>`, `<DTSPath>/Connections`, `<DTSPath>/PipelineComponents`, and `<DTSPath>/UpgradeMappings` folders if they don't exist already.
   
   1. Creates your own XML file for extension mappings in `<DTSPath>/UpgradeMappings` folder.
   
   1. Installs all assemblies referenced by your custom/3rd party data flow component assemblies in the global assembly cache (GAC).

Here are examples from our partners, [Theobald Software](https://kb.theobald-software.com/xtract-is/XIS-for-Azure-SHIR) and [Aecorsoft](https://www.aecorsoft.com/blog/2020/11/8/using-azure-data-factory-to-bring-sap-data-to-azure-via-self-hosted-ir-and-ssis-ir), who have adapted their data flow components to use our express custom setup and self-hosted IR as a proxy for Azure-SSIS IR.

## Enforce TLS 1.2

If you need to use strong cryptography/more secure network protocol (TLS 1.2) and disable older SSL/TLS versions at the same time on your self-hosted IR, you can download and run the *main.cmd* script that can be found in the *CustomSetupScript/UserScenarios/TLS 1.2* folder of our public preview blob container. Using [Azure Storage Explorer](https://storageexplorer.com/), you can connect to our public preview blob container by entering the following SAS URI:

`https://ssisazurefileshare.blob.core.windows.net/publicpreview?sp=rl&st=2020-03-25T04:00:00Z&se=2025-03-25T04:00:00Z&sv=2019-02-02&sr=c&sig=WAD3DATezJjhBCO3ezrQ7TUZ8syEUxZZtGIhhP6Pt4I%3D`

## Current limitations

- Only data flow components that are built-in/preinstalled on Azure-SSIS IR Standard Edition, except Hadoop/HDFS/DQS components, are currently supported, see [all built-in/preinstalled components on Azure-SSIS IR](./built-in-preinstalled-components-ssis-integration-runtime.md).
- Only custom/3rd party data flow components that are written in managed code (.NET Framework) are currently supported - Those written in native code (C++) are currently unsupported.
- Changing variable values in both on-premises and cloud staging tasks is currently unsupported.
- Changing variable values of type object in on-premises staging tasks won't be reflected in other tasks.
- *ParameterMapping* in OLEDB Source is currently unsupported. As a workaround, please use *SQL Command From Variable* as the *AccessMode* and use *Expression* to insert your variables/parameters in a SQL command. As an illustration, see the *ParameterMappingSample.dtsx* package that can be found in the *SelfHostedIRProxy/Limitations* folder of our public preview blob container. Using Azure Storage Explorer, you can connect to our public preview blob container by entering the above SAS URI.

## Next steps

After you've configured your self-hosted IR as a proxy for your Azure-SSIS IR, you can deploy and run your packages to access data on-premises as Execute SSIS Package activities in Data Factory pipelines. To learn how, see [Run SSIS packages as Execute SSIS Package activities in Data Factory pipelines](./how-to-invoke-ssis-package-ssis-activity.md).
