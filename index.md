---
layout: default
---

## Usage

The **LOGS.BAT** file will automatically prompt for Local Admin Privileges and run a newly created PS1 script via -ExecutionPolicy Bypass.

**Right Click** the above **.bat** link and choose Save As or **Save Link As**:
![SaveAs](https://raw.githubusercontent.com/johnem-msft/ESUCHK/master/assets/images/saveas.png)

Once the file has saved locally open it: ![OpenFile](https://raw.githubusercontent.com/johnem-msft/ESUCHK/master/assets/images/openfile.png)

When attempting to run  the .bat file you may be prompted by Smart Screen Filter in Windows
![MoreInfo](https://raw.githubusercontent.com/johnem-msft/ESUCHK/master/assets/images/moreruncombo.png)<br>
Choose: **More info** and then **Run anyway**

Accept User Account Control options to run the according PowerShell script with full execution rights.

**Note**: if you experience any issue running the .bat file, you are able to run the PS1 file in PowerShell ISE as an Administrator after running the command:<br>
Set-ExecutionPolicy Unrestricted


## Details

> Latest Version is 1.6.3<br>
> Project migrated from Technet Gallery on 5/2020 due to site retirement<br>
> <a href="https://docs.microsoft.com/en-us/teamblog/technet-gallery-retirement">TechNet Gallery Retirement</a>

#### Collection Methods

- We now have the option to run collections independently – By default EventSearch, SetupDiag, Windows Update, Network Diagnostics, PowerCFG, GPResult and Misc Collections are all checked to be collected – so there is no need to modify the check-boxes unless you only want to collect a single group of logs

- TOP-ERRORS.TXT will also now also open in a GUI Out-Gridview allowing filtering if you choose the option (unchecked by default):
  - Choose which logs to collect:
  - ![LOGSgui](https://raw.githubusercontent.com/johnem-msft/ESUCHK/master/assets/images/logsgui.png)
  - ![outGridView](https://raw.githubusercontent.com/johnem-msft/ESUCHK/master/assets/images/outgridview.png)
    - Full filtering options in Out-GridView are also available:
	- ![filterGridView](https://raw.githubusercontent.com/johnem-msft/ESUCHK/master/assets/images/logsgui/master/assets/images/filtergridview.png)

- Additional Helper Files & collections:
  - All files are written to %temp%\LOGS\GPRESULT
    - This location should be zipped after collection, expect .log/.txt to compress 90%
  - TOP-ERRORS.TXT contains a count of duplicated errors against a given day - this can help to spot certain scenarios but will also include false positives
  - WER-SUMMARY.TXT contains a count of all Windows Error Reports queried from C:\Users\All Users\Microsoft\Windows\WER\ReportArchive\
  -DXDiag, MSINFO32, CBS, DISM and Windows Update Logs are all collected
    - Get-WindowsUpdateLog is executed and parsed ETL files are written to Windows-Update.log
  - All *EVTX Logs containing "error" or "fail" are collected in the /EVTX/ folder by default
  - Setupdiage.exe will download and run by default unless you uncheck it
    - Setupdiag.exe will run as a job and should take less than 10 minutes - after 10 minutes the collection for this task will abort
	- Utilize this collection for Setup, In-Place and Feature Upgrade scenarios only - logs: SetupDiag-Log.zip & SetupDiag*.log
  - NOTE: Additional separate setup collections exist in /SETUP/ w/ SETUP-OOBE & SetupAct-Regex.TXT helper files
  - /Network/ folder contains basic queries against SMB information, proxy info and WLAN Reports
  - /POWER/ folder contains basic POWERCFG and Sleep-Study queries


### Scripts

#### Batch Wrapper

```batch
@echo ####LOGSv.1.6.3 BAT-WRAPPER#####  
SET ScriptDirectory=%~dp0 
CD /D %ScriptDirectory% 
type LOGS.BAT > LOGS_.PS1 
MORE /E +10 %ScriptDirectory%LOGS_.PS1 > %ScriptDirectory%LOGS.PS1 
 
SET PowerShellScriptPath=%ScriptDirectory%LOGS.PS1 
PowerShell -NoProfile -ExecutionPolicy Bypass -Command "& {Start-Process PowerShell -ArgumentList '-NoProfile -ExecutionPolicy Bypass -File ""%PowerShellScriptPath%""' -Verb RunAs}"; 
PAUSE 
```

#### PowerShell Script

```powershell
####LOGS.PS1 v.1.6.3#####  
## PLEASE RUN THIS SCRIPT IN POWERSHELL ISE AS AN ADMINISTRATOR  
## REMEMBER TO >> > > SET-EXECUTIONPOLICY UNRESTRICTED < < <<   
  
###  Changes in 1.6.3: 
###  Added DISM /Online /Get-Packages to Installed_Updates.TXT 
###  Added miglog.xml, *APPRAISER*.xml to /SETUP/ Collection 
###  Added /Windows/Minidump/*.DMP to DMP collection  
  

  
        ## Misc Logs Collection: #7        
        if($Global:miscCollect -eq $True)  
            {  
            getMSINFO #function & job  
                wh "...`n"  
            PrinterCheck #function  
                wh "...`n"  
            getProcesses #function  
                wh "...`n"  
            getApps #function - make job - takes a min  
                wh "...`n"  
            SetupLogs #function  
                wh "...`n"       
            sysProductCheck #function  
                wh "...`n"                 
            reservedCheck #function  
                wh "...`n"  
            fltmcCheck #function  
                wh "...`n"  
            getDXDiag #function  
                wh "...`n"  
            regLang #function  
                wh "...`n"  
            autoRotate #function  
            getMISCLogs #function  
                wh "...`n"  
            getDrivers #function  
                wh "...`n"   
            getAV #function  
                wh "...`n"            
             }  
        
  
#### RECEIVING JOBS SECTION ###...   
  
        #EventSearchJob  
        if($Global:EventsCollect -eq $True)  
        {          
            wh "`nWaiting for EventSearchJob to complete...`n"  
  
            Receive-Job -Name EventSearchJob -OutVariable eventSearch -Wait   
            $search = $eventSearch.Line  
        }  
  
  
        if($Global:SetupDiagCollect -eq $True)  
        {  
            #SetupDiagJob - Receive-Job  
            $stamp = (Get-Date -format "hh:mm tt")  
            wh "`nWaiting for SetupDiagJob to complete..."  
            wh "`nTime Stamp: $stamp"  
            wh "`nThis can take up to 10 minutes ..."  
  
            Do{  
              SLEEP 15  
                wh "."  
                if((Get-Job -name SetupDiagJob).State -eq "Completed")  
                    { Receive-Job -Name SetupDiagJob  
                           wh "`nSetupDiag Completed!"                         
                        Break                      }  
                                }Until($Cancel.SetupDiag -eq $True)  
            wh `n  
                                               
            #Receive file and copy  
            Receive-Job -Name SetupDiagJob -Wait   
            Copy-Item $tempDir\Logs*.zip $tempDir\LOGS\SetupDiag-Log.zip  
            Copy-Item $tempDir\setupdiag*.log $tempDir\LOGS\  
            Remove-Item $tempDir\Logs*.zip  
        }  
  
       
        if($Global:UpdatesCollect -eq $True)  
        {  
            #GetUpdates Job via:  
            #UpdateHelper <--- GetUpdates Job has to finish first!  
            #Checking Status of GetUpdates Job...  
            wh "Checking Status of GetUpdates Job...`n"  
            If ((Get-Job -Name GetUpdates).State -eq "Failed")  
                { wh "`nGetUpdates Job Failed!`n" }  
                    Else{  
                            Receive-Job -Name GetUpdates -wait  
                            Move $env:USERPROFILE\Desktop\WindowsUpdate.log $TempDir\LOGS\Windows-Update.log -Force  
                            wh "`n Writing Update Helper Info to UPDATE-ERRORS.TXT ... `n"  
                            UpdateHelper #run the update helper function  
                        }               
        } #End getting GetUpdates-job       
  
        #Finishing EventSearch  
        if($Global:EventsCollect -eq $True)  
            {  
                writeSearch #function  
            }  
  
#Wait on MSINFO...  
if($Global:miscCollect -eq $True)  
{  
    wh "`n Waiting for MSINFO32 to Complete ...`n"  
    do{ start-sleep 1 }  
    Until((get-process | select-string -Pattern "msinfo").Pattern -cne "msinfo")  
        werHint #function  
}  
  
  
if((Get-Host).Version.Major -cge 5) ##WIN7 Does not Support Transcript  
    {  
  
Stop-Transcript   
  
        do{  
    start-sleep 1  
    }  
    Until((get-item $tempDir\LOGS\Event-Search.TXT).Length -cne 0)  
      
    }  
  
wh "`nLog Collection Completed! `nLogs are available in %temp%\LOGS\`n"    
wh "`nHit Any Key or Close ...`n"  
  
Start-Sleep 1  
  
Start Explorer.exe $explore  
  
PAUSE  
  
## LOGS.PS1 1.6.3  ##     
## JOHNEM 8-2019 ##   
## EOF ##
```

