
**Sherlock Scenario**
```
You have been presented with the opportunity to work as a junior DFIR consultant for a big consultancy. However, they have provided a technical assessment for you to complete. The consultancy Forela-Security would like to gauge your Windows Event Log Analysis knowledge. We believe the Cyberjunkie user logged in to his computer and may have taken malicious actions. Please analyze the given event logs and report back.
```

We are given the archive `logjammer.zip` which contains the Windows event logs (`.evtx`) of the computer `DESKTOP-887GK2L`:
- `Security.evtx`
- `System.evtx`
- `Windows Defender-Operational.evtx`
- `Windows PowerShell-Operational.evtx`
- `Windows Firewall-Firewall.evtx`

To analyze the logs I am going to use the **ELK stack** (Elasticsearch + Kibana), loading all the event logs in an index and searching through them in **Kibana Discover** using **KQL** (Kibana Query Language)
- Each event has its fields under `Event.System` (general info like `EventID`, time, computer) and `Event.EventData` (the actual data of the event)
- If a field name is not exactly the same, we can expand any matching event in Discover and check the exact path in the `JSON` tab

All the activity happened on 27 March 2023, so the time range has to be set to cover that date (or just use `Last 5 years`)

The structure of a query is pretty simple:
```
<field> : <value> and <field> : <value>
```

### 1. When did the user cyberjunkie successfully log into his computer? (UTC)

A successful logon is logged in the `Security` log with the Event ID `4624` (An account was successfully logged on), so we filter for that event ID and the username `CyberJunkie`
```
Event.System.EventID : 4624 and Event.EventData.TargetUserName : "CyberJunkie"
```
- The `@timestamp` of the event is stored in UTC

The user logged in at: `27/03/2023 14:37:09`

This also gives us a starting time from which we can look at what the user did next

### 2. The user tampered with firewall settings on the system. Analyze the firewall event logs to find out the Name of the firewall rule added?

When a rule is added to the Windows Firewall exception list the event with ID `2004` is logged in the firewall log
```
Event.System.EventID : 2004
```
- The output gives 3 rules added:

| RuleName             | Direction | Action | Protocol/Port |
| -------------------- | --------- | ------ | ------------- |
| Metasploit C2 Bypass | Outbound  | Allow  | TCP 4444      |
| Microsoft Edge       | Outbound  | Block  | All           |
| Microsoft Edge       | Inbound   | Block  | All           |

- The 2 Microsoft Edge rules look normal, but the first one opens port `4444`, which is the default port used by Metasploit for reverse shells

The name of the rule is: `Metasploit C2 Bypass`

### 3. Whats the direction of the firewall rule?

From the same event we check the field `Event.EventData.Direction`, where `1` means inbound and `2` means outbound

The direction of the rule is: `Outbound`

### 4. The user changed audit policy of the computer. Whats the Subcategory of this changed policy?

A change in the audit policy is logged in the `Security` log with the Event ID `4719` (System audit policy was changed)
```
Event.System.EventID : 4719
```
- The output gives a single event, but the raw log does not store the name of the subcategory, only its GUID in the field `Event.EventData.SubcategoryGuid`:
```
{0cce9227-69ae-11d9-bed3-505054503030}
```
- Searching the GUID online (or opening the event in Event Viewer, which translates it) we find that it belongs to the `Object Access` category, with `Success` auditing added

The subcategory is: `Other Object Access Events`

### 5. The user "cyberjunkie" created a scheduled task. Whats the name of this task?

The creation of a scheduled task is logged in the `Security` log with the Event ID `4698` (A scheduled task was created)
```
Event.System.EventID : 4698
```
- The name of the task is in the field `Event.EventData.TaskName`: `\HTB-AUTOMATION`
- The field `Event.EventData.TaskContent` contains the whole task in XML format (HTML encoded, so it can be decoded with CyberChef), and the important parts are:
```xml
<RegistrationInfo>
    <Date>2023-03-27T07:51:21.4599985</Date>
    <Author>DESKTOP-887GK2L\CyberJunkie</Author>
    <Description>practice</Description>
    <URI>\HTB-AUTOMATION</URI>
</RegistrationInfo>
<Triggers>
    <CalendarTrigger>
        <StartBoundary>2023-03-27T09:00:00</StartBoundary>
        <Enabled>true</Enabled>
        <ScheduleByDay>
            <DaysInterval>1</DaysInterval>
        </ScheduleByDay>
    </CalendarTrigger>
</Triggers>
...
<Actions Context="Author">
    <Exec>
        <Command>C:\Users\CyberJunkie\Desktop\Automation-HTB.ps1</Command>
        <Arguments>-A cyberjunkie@hackthebox.eu</Arguments>
    </Exec>
</Actions>
```
- The task runs a PowerShell script every day at 09:00, which is a common way of getting persistence

The name of the task is: `HTB-AUTOMATION`

### 6. Whats the full path of the file which was scheduled for the task?

From the task XML found in the previous question, the file is under the `<Command>` tag

The full path of the file is: `C:\Users\CyberJunkie\Desktop\Automation-HTB.ps1`

### 7. What are the arguments of the command?

Also from the task XML, the arguments are under the `<Arguments>` tag

The arguments are: `-A cyberjunkie@hackthebox.eu`

### 8. The antivirus running on the system identified a threat and performed actions on it. Which tool was identified as malware by antivirus?

When Windows Defender takes action against malware, it logs the Event ID `1117` (The antimalware platform performed an action to protect your system) in the Defender log
```
Event.System.EventID : 1117
```
- Under `Event.EventData` we have the fields `Threat Name`, `Severity Name`, `Path` and `Action Name`
- The threat name is `HackTool:MSIL/SharpHound!MSR` with severity `High`

SharpHound is the data collector used by BloodHound to map Active Directory environments

The tool identified as malware is: `SharpHound`

### 9. Whats the full path of the malware which raised the alert?

From the same event we check the `Path` field

The full path of the malware is: `C:\Users\CyberJunkie\Downloads\SharpHound-v1.1.0.zip`

### 10. What action was taken by the antivirus?

From the same event we check the `Action Name` field

The action taken was: `Quarantine`

### 11. The user used Powershell to execute commands. What command was executed by the user?

The commands executed in PowerShell are logged in the PowerShell log with the Event ID `4103` (Module logging, which logs every executed command along with its parameters)
```
Event.System.EventID : 4103
```
- The details are in the fields `Event.EventData.Payload` and `Event.EventData.ContextInfo`
- From the 11 results, most of them are not interesting, except for the one at `2023-03-27 14:58:33` which shows:
```
CommandInvocation(Get-FileHash): "Get-FileHash"
ParameterBinding(Get-FileHash): name="Algorithm"; value="md5"
ParameterBinding(Get-FileHash): name="Path"; value=".\Desktop\Automation-HTB.ps1"
```
- The user calculated the MD5 hash of the same script used in the scheduled task

The command executed is: `Get-FileHash -Algorithm md5 .\Desktop\Automation-HTB.ps1`

### 12. We suspect the user deleted some event logs. Which Event log file was cleared?

When an event log is cleared, the `System` log records the Event ID `104` (The log file was cleared)
```
Event.System.EventID : 104
```
- The output gives a single event, and the name of the cleared log is in the field `Event.UserData.LogFileCleared.Channel`
- This is the log where the firewall rule from question 2 was recorded, so the user was most likely trying to cover their tracks

The cleared event log is: `Microsoft-Windows-Windows Firewall With Advanced Security/Firewall`
