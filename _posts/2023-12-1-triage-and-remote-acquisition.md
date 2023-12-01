---
published: true
---
# Remote Acquisition and Triage

Pertinent device data will not always reside within a local storage drive. In some instances data needs to be pulled from online storage accounts and local servers. This writeup will attempt to provide a brief overview of what to look out for when performing remote acquisition and triage. It is by no means a solution for every possible use case and is merely collated experience and findings.

> It is important to liaison with the IT Administrator and staff. You will not know their internal server structure better then them.

## Cloud Providers

## Three most popular cloud providers

| Company         | Service         |
| :-------------: | :-------------: |
| Microsoft       | Azure           |
| Amazon          | AWS             |
| Google          | GCP             |

> These providers tend to have [SaaS](https://en.wikipedia.org/wiki/Software_as_a_service) alternatives and front-end services associated to their cloud. For example, `Office 365` and `Google Workspace`.

## Data to acquire

Regardless of whether it is Microsoft's Azure, Amazon's AWS or Google's GCP if the IT infrastructure utilizes some form of cloud provider, **assuming we are granted credentials**, data can be obtained.

## Sign-in and audit logs

All of the providers will have a slightly different user interface, however, the approach to acquire their data remains the same:

```
Browse their tenant, filter to the user you want, download/export the necessary logs.
```

> If the subject of your triage pertains to malware possibly infecting a Azure Domain collecting all logs from the Domain Controller will yield relevant information.

## OneDrive downloads

1. **Forensic image**
    * Utilise your preferred imaging software and perform a logical image over the OneDrive directory
    ![ftk-logical]({{site.baseurl}}/images/ftk-logical.gif "FTK Imager")
2. **Office 365 portal**
    * Navigate to the user and download a .ZIP of the user's OneDrive (*assuming admin*)
    ![onedrive-portal]({{site.baseurl}}/images/onedrive-portal.png "One Drive Portal")
3. **Powershell script**
    * A [powershell script](https://www.sharepointdiary.com/2021/09/how-to-download-all-files-from-onedrive-for-business.html) with the correctly supplied parameters is able to somewhat automate the process

    ```powershell
    #Parameters
    $OneDriveSiteURL = "https://crescent-my.sharepoint.com/personal/salaudeen_crescent_com"
    $DownloadPath ="C:\Temp\OneDrive"
 
    Try {
        #Connect to OneDrive site
        Connect-PnPOnline $OneDriveSiteURL -Interactive
        $Web = Get-PnPWeb
    
        #Get the "Documents" library where all OneDrive files are stored
        $List = Get-PnPList -Identity "Documents"
    
        #Get all Items from the Library - with progress bar
        $global:counter = 0
        $ListItems = Get-PnPListItem -List $List -PageSize 500 -Fields ID -ScriptBlock { Param($items) $global:counter += $items.Count; Write-Progress -PercentComplete `
                    ($global:Counter / ($List.ItemCount) * 100) -Activity "Getting Items from OneDrive:" -Status "Processing Items $global:Counter to $($List.ItemCount)";} 
        Write-Progress -Activity "Completed Retrieving Files and Folders from OneDrive!" -Completed
    
        #Get all Subfolders of the library
        $SubFolders = $ListItems | Where {$_.FileSystemObjectType -eq "Folder" -and $_.FieldValues.FileLeafRef -ne "Forms"}
        $SubFolders | ForEach-Object {
            #Ensure All Folders in the Local Path
            $LocalFolder = $DownloadPath + ($_.FieldValues.FileRef.Substring($Web.ServerRelativeUrl.Length)) -replace "/","\"
            #Create Local Folder, if it doesn't exist
            If (!(Test-Path -Path $LocalFolder)) {
                    New-Item -ItemType Directory -Path $LocalFolder | Out-Null
            }
            Write-host -f Yellow "Ensured Folder '$LocalFolder'"
        }
    
        #Get all Files from the folder
        $FilesColl =  $ListItems | Where {$_.FileSystemObjectType -eq "File"}
    
        #Iterate through each file and download
        $FilesColl | ForEach-Object {
            $FileDownloadPath = ($DownloadPath + ($_.FieldValues.FileRef.Substring($Web.ServerRelativeUrl.Length)) -replace "/","\").Replace($_.FieldValues.FileLeafRef,'')
            Get-PnPFile -ServerRelativeUrl $_.FieldValues.FileRef -Path $FileDownloadPath -FileName $_.FieldValues.FileLeafRef -AsFile -force
            Write-host -f Green "Downloaded File from '$($_.FieldValues.FileRef)'"
        }
    }
    Catch {
        write-host "Error: $($_.Exception.Message)" -foregroundcolor Red
    }
    ```

## Amazon Web Services (AWS)

Download Amazon S3 (Simple Storage Service) Bucket's via the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

```console
#supply credentials
aws configure 

# directory listing
aws s3 ls s3://nameofbucket

# download files to local C: drive
aws s3 cp s3://nameofbucket/secret.txt C:\triagefolder\secret.txt

# download all files using the recursive flag
s3://nameofbucket C:\triagefolder\ ‐‐recursive
```

## Local server

In the instance of a local server (LAN) the approach is **not** going to be as uniform as all local servers will differ and are not bound to the standards, configurations and layouts of the cloud providers.

I found success with the following methodology with a local *Linux* server running *CentOS*.

1. **Connect your PC/laptop to the same network as the server via ethernet cable**
2. **Assign your PC to a useable static host address within the same local network**
    * Server IP could be on a `10.0.1.0/16` network for example
    * `/16 = 255.255.0.0`
    * Determining the number of hosts addresses is `2^(32 - subnet mask)`. In this case, it's `2^(32 - 16) = 2^16 = 65536` usable hosts
    * Valid range from `10.0.0.1` to `10.0.255.254`, excluding the network address (`10.0.0.0`) and the broadcast address (`10.0.255.255`).
    * ![local-network-changes]({{site.baseurl}}/images/local-network-changes.gif "Changing Network Settings")
3. **Access the server**
    * Validate connection to the server via a `ping` command.
    * Browse to the server directory via file explorer using the server host IP address, for example, `\\10.0.1.1`
    * Perform logical image (**requires** the server to mapped as a network drive) otherwise compress the files: `tar -cvf archive_name.tar file_to_archive`
    * A `ipconfig /renew` command might be required as well as changing firewall rules

> If there is a *time constraint* then **compressing** the files/directories of interest can be done over a logical forensic image (compressing preserves some form of file metadata).

## Registry Keys and Group Policy

When copying contents from the cloud to our local storage device we need to consider organisation policy.

> Some organisations block the use of external storage devices connecting to their machines.

Some policies might be enforced via *Group Policy* settings:
1. `Win + R`
2. Type `gpedit.msc`
3. Navigate to the following path: `Computer Configuration -> Administrative Templates -> System -> Removable Storage Access`
4. Double click on the policy and disable it
5. Update *Group Policy*
    * `gpupdate /force`
    * Or for a target computer: `gpupdate /target:computer /force`
    * Or for a target user:     `gpupdate /target:user /force`

Disable the Windows Registry Key responsible for blocking external storage devices:
1. `Win + R`
2. Type `regedit`
3. Go to `HKEY_LOCAL_MACHINE \ SYSTEM \ CurrentControlSet \ Services \ USBSTOR`
4. Change the value to **3** ![usbstor]({{site.baseurl}}/images/usbstor.png "Changing USBSTOR Registry Key")

## FileZilla

In the instance of a FTP (File Transfer Protocol) server utilising a *GUI* tool like the [FileZilla](https://filezilla-project.org/) client is ideal.

Launch FileZilla and select `File` and then `Site Manager`.

Provide the necessary host, username, password and port number (defaults to 21 if left blank). ![filezilla-client]({{site.baseurl}}/images/filezilla-client.png "FileZilla Client")

## Outlook email acquisition

1. Export the mail directly from the local machine via the Outlook executable
2. Setup a delegate account within Office 365 on the user of interest (allows someone else to access another user's mailbox on their behalf)
    1. Navigate to Exchange Admin Center
    2. Access Mailbox Delegation
    3. Select the user's mailbox and add a delegate 

> If you are pulling down emails and waiting for them to sync a good way to verify is by viewing the size of the **.OST** file located in `C:\Users\your_username\AppData\Local\Microsoft\Outlook`.

## Closing thoughts

Within Office 365 if there is concerns about the user deleting and accessing emails the user can be placed in a account [Litigation Hold](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/holds/litigation-holds?view=exchserver-2019) which essentially audits the account and enables the admin to retain emails, even if the user deletes them from their machine.

It is important to remember that the person who signed up for the Azure/365 service (typically the IT admin) will automatically have Global Administrator (GA) access.

In a ideal world you would get the IT admin to create a new user for you with the least amount of privileges you require to perform your acquisition. 

This way all the activity performed is tied to that user account and you cannot be held accountable for actions performed by other user accounts (including the GA!). 

Once the data is acquired the user can be deleted from the tenant.