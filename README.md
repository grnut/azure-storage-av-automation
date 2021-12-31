# Antivirus Automation for Azure Storage
##### Authored by: Aviv Shitrit, Inbal Argov, Greg Nuttall | Updated: Dec 31st, 2021  

Antivirus Automation for Azure Storage is an ARM template that sets resources in your environment in order to protect an Azure blob container from malware by scanning every blob that’s uploaded. The project consists of a function triggered when files are uploaded to or updated in a Blob storage container, and a Windows VM that utilizes  Microsoft Defender Antivirus.

<img src="https://raw.githubusercontent.com/Azure/azure-storage-av-automation/main/AvAutoSystem.png"/>

For each blob uploaded to the protected container, the function will send the blob to the VM for scanning and change the blob location according to the scan results:
* If the blob is clean, it’s moved to the clean-files container
* If it contains malware it’s moved to the quarantine container

The Azure function and the VM are connected through a virtual network and communicate using HTTPS requests.  

The system:
* Supports parallel blob scanning
* Is designed for simple use for Dev and Non-Dev users
* Can be customized

List of created resource:
1. Function App
1. App Service Plan
1. Virtual Network
1. Network Security Group
1. Storage Account - Host for the function app
1. Virtual Machine
1. Disk - Storage for the VM
1. Network Interface - NIC for the VM
1. Key Vault
    1. Stores connection string to the storage account to scan as a secret
1. Optional: Public IP - Used to access the VM from public endpoint
# Getting Started - Deployment

## Prerequisites:
* Azure Subscription
* Azure Storage Account with at least one blob container
* Azure Storage Account to host the code package (can be the same as the target Storage Account)
* .Net Core 3.1 SDK installed
* Azure CLI Tools installed - [Install Azure CLI][instalCliUrl]
* Knowledge of PowerShell scripting and Git
### Deployment Steps
1. Use the following command to clone the repo
    ```
    git clone https://github.com/grnut/azure-storage-av-automation.git
    ```
2. Open Storage AV Automation/Scripts/BuildAndDeploy.ps1 and run the script. During the execution, you will be prompted to enter your Azure credentials and the necessary parameters.
    * The script also deploys the ARM template.

    
> :warning: During execution, .zip files (ScanHttpServer.zip, ScanUploadedBlobFunction.zip) will be created in their respective folders.     
>
>If the deployment fails for any reason then these zip files need to be deleted before running the script again. This is because if the script is run twice in a row without this step then the zip will "double up" it's contents and fail to be read properly in Azure.

## Important notes
* Blobs uploaded to the protected container are sent from the Function to VM through HTTPS request inside the Virtual Network.
* Files potentially containing malware are saved locally by the VM for scanning. They're deleted afterwards. So be aware that the VM might be compromised.
* The port number for the communication is hardcoded and can't be passed as parameter.
* The ARM template deployment can take up to 10 minutes so be patient.
* Monitoring the system:
    * Function - you can configure Azure Application Insights resource to monitor the Function logs by going to your Function->Monitor->Configure Application Insights.
    * Scan HTTP Server - logs kept inside the machine in /log/ScanHttpServer.log as a single file.

## Testing by uploading a test malware file (EICAR):
1. On your computer, create a text file using Notepad (or any other text editing program) and copy the following string into it:
X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*

    This is a malware test file, called EICAR (see more information), and should be regarded as malware by all antivirus software. 
    1. In case your computer’s AV marked the file as malicious, you can exclude a dedicated folder from being scanned, and create the EICAR again inside that folder.

2. Upload the EICAR to the designated upload container (any upload method will work)

3. You can see the process of the AV by turning on the Monitor blade of the Azure Function, and see the events being logged 

## Project Structure:
* ScanUploadedBlobFunction - contain the Azure Function Blob Trigger source code.

* ScanHttpServer - contains the HttpServer project that runs on the VM and waits for requests, the VM has in Init Script to start the ScanHttpServer. The script can be found in the same folder and can be modified too. The ScanHttpServer will be run with a simple script that restart the app if it crashes (/ScanHttpServer/runLoop.ps1)

* Build and Deploy Script (BuildAndDeploy.ps1) - will prepare the project for deployment, upload the source code to a host storage account and deploy the arm template using the parameters in the script (the script overrides the ARM template parameters file).

    *  Function Code - build, zipped and uploaded
        * build command:
    
        ```powershell
        dotnet publish <csproj-file-location> -c Release -o <out-path>
        ```

    *  ScanHttpServer Code - build, zipped and uploaded using this command:

        ```powershell
        dotnet publish -c Release -o <csproj-file-location>
        ```

    * The zip file must contain the ScanHttpServer binary files and runLoop.ps1 script to run the server on the VM.

    * Build and Deploy Script Parameters:
        * sourceCodeStorageAccountName - Storage account name to store the source code, must be public access enabled.
        * sourceCodeContainerName - Container name to store the source code, can be new or existing, if already exists must be with public access.
        * subscriptionID - Storage account to scan subscription ID.
        * targetResourceGroup - Storage account to scan resource group name.
        * targetStorageAccountName - Name of the storage account to scan.
        * targetContainerName - Name of the container to scan.
        * deploymentResourceGroupName - Resource group to deploy the AV system to.
        * deploymentResourceGroupLocation - Resource group Geo location.
        * vmUserName - VM username
        * vmPassword - VM password

[instalCliUrl]: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli

[blobTriggerAlternatives]: https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-storage-blob-trigger?tabs=csharp#alternatives

[existingVnetScenario]: https://github.com/Azure/azure-storage-av-automation#existing-vnet-variation

[enableAppInsights]: https://docs.microsoft.com/en-us/azure/azure-functions/functions-monitoring

[azureMonitorLogs]: https://docs.microsoft.com/en-us/azure/azure-functions/functions-monitor-log-analytics?tabs=csharp
