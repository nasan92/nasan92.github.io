---
title: "Stop Azure Backup for multiple SQL on Azure VM Databases with Powershell"
author: Nathanael Santschi
date: 2022-10-27T10:51:21+01:00
draft: false
tags:
  - powershell
  - Azure
  - AzureBackup
  - SQLonAzureVM
categories:
  - Azure
  
---

link: https://learn.microsoft.com/en-us/azure/backup/backup-azure-sql-automation#stop-protection

It had a case where I did need to migrate a lot of databases from one SQL Server on a Azure VM to another VM. After the successful migration there were a lot of old databases on the "old SQL Server" offline and I got a lot of Azure Backup Alerts.
Because I was to lazy to disable the backup for each database by hand, I created the following small script which will do that job. 

## Parameters
First we define parameters
- resource group of **Recovery Service Vault**
- **Recovery Service Vault Name**
- because I had a lot more DBs to stop the Backup I created a variable with Database Names to keep (not stop backup on those)

```powershell
# PARAMETER
$rsg = "RSG-Name"
$vaultname = "RecoveryServicesVaultName"

# Stop Backup on all DBs except those:
$dbsToKeep = "db1tokeep","dp2tokeep","master","model","msdb","tempdb"
```
## Retrieve Recovery Service Vault and all backuped SQL Databases
Then we Get the Recoveryservice vault to be able to query all backuped SQL Items from the vault:

```powershell
$vault = Get-AzRecoveryServicesVault -ResourceGroupName $rsg -Name 

$allsql = Get-AzRecoveryServicesBackupItem -BackupManagementType AzureWorkload -WorkloadType MSSQL -VaultId $vault.ID `
```
## Get just the Databases for which we want to stop the backup
Now we have all SQL Databases which are backuped in the Array **$allsql**
To get all Databases into an Array which names are not in the **$dbsToKeep** Array I do the following:
1. Create a new empty Array $dbsToStopBackup in which I will later store all DB items which do not match the names from the $dbsToKeep Array
2. Now I loop through all db items in the $allsql array and verify if their name is in $dbsToKeep and If not, they will be added to $dbsToStopBackup

```powershell
$dbsToStopBackup = @()
foreach($db in $allsql){
    $exclude = $false
    foreach($name in $dbsToKeep){
        
        if($db.name -match $name){
            $exclude = $true
        }
    }

    if(!($exclude)){
        $dbsToStopBackup += $db
    }
    
}
```

It makes sense to verify if all the correct Dbs are now in the **$dbsToStopBackup** Array. 

## Stop / Disable the backup for those databases
And last but not least we run a loop which disables the backup for all of those databases: 

```powershell
foreach($db in $dbsToStopBackup){
    write-host "Stop Backup for" $db.name "Data will be retained"
    # stop backup BUT retain data
    $bkpItem = Get-AzRecoveryServicesBackupItem -BackupManagementType AzureWorkload -WorkloadType MSSQL -Name $db.name -VaultId $vault.ID
    Disable-AzRecoveryServicesBackupProtection -Item $bkpItem -VaultId $vault.ID -Force
}
```
## the full "script": 

````powershell
# PARAMETER
$rsg = "RSG-Name"
$vaultname = "RecoveryServicesVaultName"

# Stop Backup on all DBs except those:
$dbsToKeep = "db1tokeep","dp2tokeep","master","model","msdb","tempdb"

# Get Backup Vault
$vault = Get-AzRecoveryServicesVault -ResourceGroupName $rsg -Name 
# $bkpItem = Get-AzRecoveryServicesBackupItem -BackupManagementType AzureWorkload -WorkloadType MSSQL -Name "DocuManager" -VaultId $vault.ID

# Get All SQL Databases which are backuped in this vault
# Get-AzRecoveryServicesBackupItem -BackupManagementType AzureWorkload -WorkloadType MSSQL -VaultId $vault.ID

$allsql = Get-AzRecoveryServicesBackupItem -BackupManagementType AzureWorkload -WorkloadType MSSQL -VaultId $vault.ID 

$dbsToStopBackup = @()
foreach($db in $allsql){
    $exclude = $false
    foreach($name in $dbsToKeep){
        
        if($db.name -match $name){
            $exclude = $true
        }
    }

    if(!($exclude)){
        $dbsToStopBackup += $db
    }
    
}

$dbsToStopBackup

foreach($db in $dbsToStopBackup){
    write-host "Stop Backup for" $db.name "Data will be retained"
    # stop backup BUT retain data
    $bkpItem = Get-AzRecoveryServicesBackupItem -BackupManagementType AzureWorkload -WorkloadType MSSQL -Name $db.name -VaultId $vault.ID
    Disable-AzRecoveryServicesBackupProtection -Item $bkpItem -VaultId $vault.ID -Force
}
```

