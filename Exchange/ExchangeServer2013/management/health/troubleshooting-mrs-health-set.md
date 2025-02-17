---
title: Troubleshooting MRS Health Set
TOCTitle: Troubleshooting MRS Health Set
ms:assetid: 21947ed6-1584-4db9-9cd6-f6c1de22e352
ms:mtpsurl: https://technet.microsoft.com/library/ms.exch.scom.mrs(v=EXCHG.150)
ms:contentKeyID: 49720741
ms.reviewer:
ms.topic: article
description: How to troubleshoot the MRS health set in Exchange 2013
manager: serdars
ms.author: serdars
author: msdmaguire
f1.keywords:
- NOCSH
mtps_version: v=EXCHG.150
---

# Troubleshooting MRS Health Set

_**Applies to:** Exchange Server 2013_

The Mailbox Replication Service (MRS) health set monitors the overall health of the MRS service.

## Explanation

The MRS service is monitored by using the following probes and monitors.

<table>
<colgroup>
<col/>
<col/>
<col/>
<col/>
</colgroup>
<thead>
<tr class="header">
<th>Probe</th>
<th>Health Set</th>
<th>Dependencies</th>
<th>Associated Monitors</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p>MRSServiceCrashingProbe</p></td>
<td><p>MRS</p></td>
<td><p>Information Store</p></td>
<td><p>MRSServiceCrashingMonitor</p></td>
</tr>
</tbody>
</table>

For more information about probes and monitors, see [Server health and performance](../../server-health-and-performance-exchange-2013-help.md).

## User Action

It's possible that the service recovered after it issued the alert. Therefore, when you receive an alert that specifies that the health set is unhealthy, first verify that the issue still exists. If the issue does exist, perform the appropriate recovery actions outlined in the following sections.

## Verifying the issue still exists

1. Identify the health set name and the server name in the alert.

2. The message details provide information about the exact cause of the alert. In most cases, the message details provide sufficient troubleshooting information to identify the root cause. If the message details are not clear, do the following:

   1. Open the Exchange Management Shell, and then run the following command to retrieve the details of the health set that issued the alert:

      ```powershell
      Get-ServerHealth <server name> | ?{$_.HealthSetName -eq "<health set name>"}
      ```

      For example, to retrieve the MRS health set details about server1.contoso.com, run the following command:

      ```powershell
      Get-ServerHealth server1.contoso.com | ?{$_.HealthSetName -eq "MRS"}
      ```

   2. Review the command output to determine which monitor reported the error. The **AlertValue** value for the monitor that issued the alert will be `Unhealthy`.

   3. Rerun the associated probe for the monitor that is in an unhealthy state. Refer to the table in the Explanation section to find the associated probe. To do this, run the following command:

      ```powershell
      Invoke-MonitoringProbe <health set name>\<probe name> -Server <server name> | Format-List
      ```

      For example, assume that the failing monitor is **MRSServiceCrashingMonitor**. The probe associated with that monitor is **MRSServiceCrashingProbe**. To run that probe on server1.contoso.com, run the following command:

      ```powershell
      Invoke-MonitoringProbe MRS\MRSServiceCrashingProbe -Server server1.contoso.com | Format-List
      ```

   4. In the command output, review the **Result** value of the probe. If the value is **Succeeded**, the issue was a transient error, and it no longer exists. Otherwise, refer to the recovery steps outlined in the following sections.

## Common Issues

When you receive an alert from a health set, the email message contains the following information:

- Name of the server that sent the alert

- Time and date when the alert occurred

- Authentication mechanism used, and credential information

- Full exception trace of the last error, including diagnostic data and specific HTTP header information

  You can use the information in the full exception trace to help troubleshoot the issue. The exception generated by the probe contains a Failure Reason that describes why the probe failed.

### Mailbox Locked

When a Mailbox is locked, you may receive an alert that resembles the following:

*MailboxIdentity: namprd03.prod.outlook.com/Microsoft Exchange Hosted Organizations/example.com/User6 MailboxGuid: Primary (00000000-abcd-01234-5678-1234567890ab) RequestFlags: IntraOrg, Pull, Protected Database: exampledb-db089 Exception: MapiExceptionADUnavailable: Unable to prepopulate the cache for user ...*

This indicates that a mailbox is locked. To unlock the mailbox, run the following command:

```powershell
New-MailboxRepairRequest -CorruptionType LockedMoveTarget -Identity <mailboxIdentity> [-Archive]
```

**Note**: In this command, replace \<*mailboxIdentity*\> with the name of the mailbox that's provided in the email message as **MailboxIdentity**. If the mailbox is an archive mailbox, you must include the **-Archive** flag. You can determine whether a mailbox is a primary or archive mailbox by viewing the **MailboxGuid** field in the alert.

### Corrupt Migration Job

When a corrupted migration job occurs, you may receive an alert that resembles the following:

*Notification thrown by MailboxMigration at 9/7/2012 9:08:32 PM. Details: Diagnostic Information: ProcessCacheEntry: First Organization :: /o=ExchangeLabs/ou=Exchange Administrative Group (FYDIBOHF23SPDLT)/cn=Recipients/cn=e80fc128879e452ebc882f6bca7007fa-Migration.8*

Corruption occurs when the migration meta-data has encountered issues. Upon corruption, Microsoft will receive a Watson report that will be investigated.To recover from this issue, you must remove the migration batch, and then re-create the batch. To do this, follow these steps:

1. To remove the corrupted batch, run the following command:

   ```powershell
   Remove-MigrationBatch -Identity
   ```

2. To re-create the batch job, run the following command:

   ```powershell
   New-MigrationBatch -Local -Name
   ```

For more information, see [Exchange PowerShell](/powershell/exchange/)

### MailboxMigration alert: CriticalError

When a critical error occurs during mail migration, you may receive an alert that resembles the following:

> Notification thrown by MailboxMigration at 9/7/2012 9:08:32 PM. Details: Diagnostic Information: ProcessCacheEntry: First Organization :: /o=ExchangeLabs/ou=Exchange Administrative Group (FYDIBOHF23SPDLT)/cn=Recipients/cn=e80fc128879e452ebc882f6bca7007fa-Migration.8

To resolve this issue, you must retry the migration. To do this, run the following command, or press the **Start** button on the Exchange admin center (EAC).

```powershell
Start-MigrationBatch -Identity <BatchName>
```

When this issue occurs, a Dr. Watson message is sent to Microsoft for investigation.

### The Migration Exchange Replication Service is not running

When you see this error reason, you can verify the health of the service by running the following command:

```powershell
Test-MRSHealth <servername> -MonitoringContext:$true
```

You can also try to start the service by running the following command:

```powershell
Start-Service msexchangemailboxreplication
```

### MSExchangeMailboxReplication RCP Ping Failed

When you see this error reason, you may receive an alert that resembles the following:

*An issue with MRS was detected at 6/26/2012 6:08:47 AM. Details: MRS RPC Ping check for server \<ServerName\> failed with the following error: The RPC endpoint for the Microsoft Exchange Mailbox Replication service couldn't respond:*

When this issue occurs, you can verify the health of the service by running the following command:

```powershell
Test-MRSHealth <servername> -MonitoringContext:$true
```

You can also try to restart the service by running the following command:

```powershell
Restart-Service msexchangemailboxreplication
```

### MSExchangeMailboxReplication Service is repeatedly crashing

When the MSExchangeMailboxReplication service crashes or stops responding, you may receive an alert that resembles the following:

*The MRS process has crashed at least 3 times in last 01:00:00. \<b\>Watson Message:\</b\> Watson report about to be sent for process id: 41432, with parameters: E12, \<ServerName\>, 15.00.0516.024, MSExchangeMailboxReplication, M.Exchange.MailboxReplicationService, M.E.M.BaseJob.BeginJob, System.ApplicationException, 7ec9, 15.00.0516.024. ErrorReportingEnabled: True.*

When this issue occurs, you can verify the health of the service by running the following command:

```powershell
Test-MRSHealth <servername> -MonitoringContext:$true
```

You can also try to restart the service by running the following command:

```powershell
Restart-Service msexchangemailboxreplication
```

### MSExchangeMailboxReplication is not scanning MDB queues

When the MSExchangeMailboxReplication service fails to scan queues, you may receive an alert that resembles the following:

*An issue with MRS was detected at 6/12/2012 6:20:44 PM. Details: MRS queue scan check for server \<servername\> failed with the following error: The Microsoft Exchange Mailbox Replication Service isn't scanning mailbox database queues for jobs. Last scan age: 04:38:02.1959439..*

When this issue occurs, you can verify the health of the service by running the following command:

```powershell
Test-MRSHealth <servername> -MonitoringContext:$true
```

You can also try to restart the service by running the following command:

```powershell
Restart-Service msexchangemailboxreplication
```

### Additional troubleshooting steps

1. Start IIS Manager, and then connect to the server that is reporting the issue to verify that the **MSExchangeServicesAppPool** application pool is running.

2. In IIS Manager, click **Application Pools**, and then recycle the **MSExchangeServicesAppPool** application pool by running the following command from the Shell:

   ```powershell
   %SystemRoot%\System32\inetsrv\Appcmd recycle apppool MSExchangeServicesAppPool
   ```

3. Rerun the associated probe as shown in step 2c in the Verifying the issue still exists section.

4. If the issue still exists, recycle the IIS service by using the IISReset utility, or by running the following command:

   ```powershell
   Iisreset /noforce
   ```

5. Rerun the associated probe as shown in step 2c in the Verifying the issue still exists section.

6. If the issue still exists, restart the server.

7. After the server restarts, rerun the associated probe as shown in step 2c in the Verifying the issue still exists section.

8. If the probe continues to fail, you may need assistance to resolve this issue. Contact a Microsoft Support professional to resolve this issue. To contact a Microsoft Support professional, visit [Support for business](https://support.microsoft.com/supportforbusiness/productselection) and then select **Servers** \> **Exchange Server**. Because your organization may have a specific procedure for directly contacting Microsoft Product Support Services, be sure to review your organization's guidelines first.

## For More Information

[What's new in Exchange 2013](../../what-s-new-in-exchange-2013-exchange-2013-help.md)

[Exchange PowerShell](/powershell/exchange/)