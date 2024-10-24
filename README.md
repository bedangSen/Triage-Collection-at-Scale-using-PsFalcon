
# Triage collection at scale using PsFalcon

> Some of the following steps have been taken from PSFalcon's wiki, which can be found [here](https://github.com/CrowdStrike/psfalcon/wiki).

## 1. Creating an API Key for Falcon from the Console
To start using PSFalcon for automation, you will need to create an API key. This key will allow PSFalcon to access the CrowdStrike APIs on your behalf.

### Steps:
1. Navigate to:
   - `Support and resources` > `Resources and tools` > `API clients and keys`

2. Click on `Create API client` and enter the following details:
   - Client Name
   - Description
   - **Scope:**
     1. Hosts: Read
     2. Real time Response: Read & Write
     3. Real time Response (admin): Write
     4. Machine Learning Exclusions: Read & Write

3. Click on `Create` and copy the following key details to a secure location:
   - Client ID
   - Secret
   - Base URL

4. Click on `Done`.

## 2. Enabling Real-Time Response from the Console

Before deploying or interacting with endpoints via Real-Time Response (RTR), you need to enable RTR in the Falcon Console by setting up a response policy for your host groups.

### Steps:
1. Navigate to `Host setup and management` > `Response and containment` > `Response policies`.
2. Click on **Create policy** and enter the following details:
   - **Policy name**: (e.g., "RTR Policy for Windows Hosts")
   - **Platform**: Windows
3. Enable the following features and click `Save`:
   - `Real-Time Response`
   - `Custom Scripts`
   - `get`
   - `put`
   - `run`
   - `put-and-run`
4. Go to the **Assigned host groups** tab and click on **Assign host groups to policy**.
5. Select the group(s) you want to assign the policy to and click **Assign group**.
   > If there is no existing group, you will need to create one.

## 3. Downloading the Module Using PowerShell
Before interacting with the CrowdStrike Falcon API, you need to install the PSFalcon PowerShell module, which provides all the necessary functions.

### Steps:

1. **Optional:** Verify Your Execution Policy to ensure you can install the module.
   - The module requires an ExecutionPolicy of RemoteSigned or lower. To check and change it, use the following commands:
     ```powershell
     Get-ExecutionPolicy
     Set-ExecutionPolicy RemoteSigned
     ```

1. Download the PSFalcon Module by running:
   ```powershell
   Install-Module -Name PSFalcon -Scope CurrentUser
   ```
   If you are updating the module or need to overwrite an existing installation, you can use the `-Force` flag:
   ```powershell
   Install-Module -Name PSFalcon -Scope CurrentUser -Force
   ```

## 4. Setting Up the Module
Once installed, the PSFalcon module needs to be imported into your PowerShell session and authenticated to make API requests.

### Steps:
1. [Import the PSFalcon module](https://github.com/CrowdStrike/psfalcon/wiki/Importing,-Syntax-and-Output#Import-the-Module):
   ```powershell
   Import-Module -Name PSFalcon
   ```

1. List all available PSFalcon commands:
   ```powershell
   Get-Command -Module PSFalcon
   ```

1. [Authenticate](https://github.com/CrowdStrike/psfalcon/wiki/Authentication#get-an-auth-token) with your Client ID and Secret:
   ```powershell
   Request-FalconToken
   ```

## 5. Setting Up the Forensics Collector in CrowdStrike Cloud
To collect forensic data, you need to upload the collector binaries to the CrowdStrike cloud and configure machine learning exclusions (if necessary).

### Steps:
1. **Uploading Collectors:** Upload the executable for the collector:
   > :warn: When you use this command to upload a file, it changes the case to lowercase. Make sure you use lowercase in the following commands to reference the file uploaded to "put" Files. 
   ```powershell
   Send-FalconPutFile -Path .\File.exe
   ```
   (optional) To upload multiple files from a directory:
   ```powershell
   Get-ChildItem C:\tools\*.exe | Send-FalconPutFile
   ```

1. **Machine Learning Exclusion:** If needed, create an exclusion for the binary to prevent detection by machine learning algorithms:
   ```powershell
   New-FalconMlExclusion -Value "file.exe" -ExcludedFrom "blocking","extraction" -GroupId "all"
   ```
   To verify the exclusion:
   ```powershell
   Get-FalconMlExclusion
   ```

## 6. Exporting Hosts Information to a CSV File
Before deploying or collecting from hosts, it's useful to export information on all hosts such as Host IDs, Group IDs, Hostnames, OS, etc. This dataset will help identify the relevant hosts.

### Steps:
1. To export all host details into a CSV file for later use, run:
   ```powershell
   Get-FalconHost -Detailed | Export-Csv -Path "./AllHosts.csv" -NoTypeInformation
   ```

This command exports the details for all hosts in your CrowdStrike environment into the `AllHosts.csv` file. You can use this file to identify host IDs, group IDs, hostnames, OS information, and more.

:warning: Please note that the field names might be different. For the purpose of clarity:
   + Host ID is represented by the column name: `device_id`
   + Group ID can be found in the `groups` column, which may contain multiple groups for each host.

## 7. Running the Collector
Once the collectors are uploaded, you can deploy them to hosts or groups of hosts to gather forensic data.

### Steps:
1. **Running on Hosts:**
   ```powershell
   $deployResult = Invoke-FalconDeploy -File "file.exe" -HostId "host-id-1", "host-id-2", "host-id-3" -Timeout 600 -QueueOffline $true
   ```

1. **Running on Groups:**
   ```powershell
   $deployResult = Invoke-FalconDeploy -File "file.exe" -GroupId "group-id-1", "group-id-2" -Timeout 600 -QueueOffline $true
   ```

1. **Running on All Hosts:**
   ```powershell
   $deployResult = Invoke-FalconDeploy -File "file.exe" -GroupId "all" -Timeout 600 -QueueOffline $true
   ```

## 8. Collecting the Output at Scale
Once the collectors have run successfully, you can collect the forensic data from each host. Below are the steps to retrieve and confirm the collection from multiple hosts.

### Steps:

1. **Define the hosts**  
   Define the hosts from which you want to collect the forensic data. You can target a single host, multiple hosts, a group, multiple groups, or all hosts.

   - **For one host:**
     ```powershell
     $hostIds = @("host-id-1")
     ```

   - **For multiple hosts:**
     ```powershell
     $hostIds = @("host-id-1", "host-id-2", "host-id-3")
     ```

   - **For one group:**
     ```powershell
     $groupId = "your-group-id"
     $hostIds = Get-FalconHost -Filter "groups:['$groupId']" -Detailed | Select-Object -ExpandProperty device_id
     ```

   - **For multiple groups:**
     ```powershell
     $groupIds = @("group-id-1", "group-id-2", "group-id-3")
     $hostIds = foreach ($groupId in $groupIds) { 
        Get-FalconHost -Filter "groups:['$groupId']" -Detailed | Select-Object -ExpandProperty device_id 
     }
     ```

   - **For all hosts:**
     ```powershell
     $hostIds = Get-FalconHost -Detailed | Select-Object -ExpandProperty device_id
     ```

1. **Run the `get` command**  
   After defining your target hosts, start the batch Real-time Response session and issue the `get` command to collect the forensic data:

   ```powershell
   $BatchId = (Start-FalconSession -Id $hostIds -Timeout 60).batch_id; 
   $batchResult = Invoke-FalconBatchGet -FilePath "C:\triage_output\Collection.zip" -BatchId $BatchId -OptionalHostId $hostIds; 
   $sessionIds = $batchResult.hosts | ForEach-Object { $_.session_id };
   ```

1. **Confirm that the files have been retrieved**
   Once the batch session is complete, you can confirm that the files have been successfully retrieved by running:
   
   ```powershell
   $confirmedFiles = $sessionIds | ForEach-Object { Confirm-FalconGetFile -SessionId $_ }; 
   $confirmedFiles
   ```

1. **Download the retrieved files**
   After confirming that the files are available, you can download them from each host using the following command. This will loop through each confirmed file and download it to the specified local path:

   ```powershell
   $confirmedFiles | ForEach-Object { Receive-FalconGetFile -Sha256 $_.sha256 -SessionId $_.session_id -Path "$($_.session_id)_Collection.zip" }
   ```


## 9. Clean-Up Phase
After the collection process is complete, it is recommended to clean up unnecessary files left on the hosts. This includes deleting the forensic collection directory and any binaries deployed during the process.

### Steps:
1. **Delete Collection Directory on the Host:**
   You can use Real-Time Response to delete the forensic collection directory `C:\triage_output\` and its contents from each host.
   ```powershell
   Invoke-FalconCommand -HostId "host-id" -Command 'rmdir /S /Q C:\triage_output'

1. **Delete Binary Pulled from the Cloud:** 
   If you deployed a binary from the CrowdStrike cloud to the host, you should also remove it from the C:\Windows\Temp\ or other directories where it was deployed.
   ```powershell
   Invoke-FalconCommand -HostId "host-id" -Command 'del C:\Windows\Temp\file.exe'
   ```
