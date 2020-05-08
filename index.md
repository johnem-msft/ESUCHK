---
layout: default
---

## Usage

The **ESUCHK.BAT** file will automatically prompt for Local Admin Privileges and run a newly created PS1 script via -ExecutionPolicy Bypass.

**Right Click** the above **.bat** link and choose Save As or **Save Link As**:
![SaveAs](https://raw.githubusercontent.com/johnem-msft/ESUCHK/master/assets/images/saveas.png)

Once the file has saved locally open it: ![OpenFile](https://raw.githubusercontent.com/johnem-msft/ESUCHK/master/assets/images/openfile.png)

Accept User Account Control options to run the according PowerShell script with full execution rights.

**Note**: if you experience any issue running the .bat file, you are able to run the PS1 script in PowerShell ISE as an Administrator just choose the ps1 link above and remember to set proper execution rights via:<br>
Set-ExecutionPolicy Unrestricted


## Details

> Latest Version is v0.92<br>
> This Batch File/PowerShell Script is for Windows 7 Only<br>
> ESU Scenario Reference: <a href="http://aka.ms/Windows7ESU">aka.ms/Windows7ESU</a>

#### Errors & Redirection:

- ESUCHK will automatically check for Windows 7 requirements to apply ESU/Extended Security Updates IPK & ATO codes

- If you do not have an update installed which is needed you will see an error indicating the update which is needed:

![ErrorRedirect](https://raw.githubusercontent.com/johnem-msft/ESUCHK/master/assets/images/kb4538483.png)
  - Please install the update mentioned in the web redirect and run the script to check again
  - As seen iin the the above example, you will often be directed to the Windows Update Catalog from the main KB article, be sure to pick the correct version of Windows and architecture when downloading (32bit vs. 64bit)
  
- If all prerequisites are successfully met you will be presented with a GUI where you can Enter your ESU PK

![filterGridView](https://raw.githubusercontent.com/johnem-msft/ESUCHK/master/assets/images/pidgui.png)
  - Enter your PK and select your year and choose OK (Year 1 is selected by default)
  
- slmgr.vbs will automatically be queried for /IPK and /ATO commands

![slmgrSucess](https://raw.githubusercontent.com/johnem-msft/ESUCHK/master/assets/images/success.png)
  - slmgr.vbs will confirm a success after successfully processing these commands with a valid ESU Key

  
### Scripts

#### Batch Wrapper

```batch
@echo #####ESUCHKv0.92 BAT-WRAPPER#####  
SET ScriptDirectory=%~dp0
CD /D %ScriptDirectory%
type ESUCHK.BAT > ESUCHK_.PS1
MORE /E +10 %ScriptDirectory%ESUCHK_.PS1 > %ScriptDirectory%ESUCHK.PS1
 
SET PowerShellScriptPath=%ScriptDirectory%ESUCHK.PS1
PowerShell -NoProfile -ExecutionPolicy Bypass -Command "& {Start-Process PowerShell -ArgumentList '-NoProfile -ExecutionPolicy Bypass -File ""%PowerShellScriptPath%""' -Verb RunAs}"; 
PAUSE
```

#### PowerShell Script

```powershell
### ESU-CHECK ###
$global:esuVer = "ESU-CHECK v.92"
Write-Host "`n"
Write-Host "#### ", "$esuVer", " ###", "`n"

###start transcript###
if($host.name.Contains("Console")){Start-Transcript $env:Temp\ESU-Check.TXT}


###sp1Check###
function sp1Check
{
   $global:winVersion = Get-WmiObject Win32_OperatingSystem
   Write-Host "WinVersion:", $winVersion.version
   if($winVersion.BuildNumber -ige 7601){ Write-Host "SP1: OK!" 
        $global:sp1Check = 1}
        Else{ Write-Host "SP1: Error - Please install Win7 Service Pack 1", $winversion.OSArchitecture -BackgroundColor Red -ForegroundColor White
                Write-Host "https://www.microsoft.com/en-us/download/details.aspx?id=5842"
                    Start-Sleep 5
                    CMD.EXE /c "`"C:\Program Files (x86)\Internet Explorer\iexplore.exe`" https://www.microsoft.com/en-us/download/details.aspx?id=5842"
                    BREAK }
}##\\sp1Check##
   

# Querying installed updates #
function updateQuery
{      
    Write-Host "Checking for updates, this may take a few minutes ..." -nonewline
    $global:updates = Get-WMIObject win32_quickfixengineering
    $global:updates > $env:Temp\ESU-Updates-Check.TXT
    Write-Host "Done!"
                                    
}##\\updateQuery##


# Checking specific updates #
function updateCheck
{
    Param([parameter(Mandatory = $true)][string]$kbnum)
    
    $check = $updates | Select-string $kbnum
    if($check.Matches.Length -gt 0){ Write-Host "KB", $kbnum, ": OK!" }
    Else{
            Write-Host "Missing KB", $kbnum, ": Please install this KB to continue - Reboot Required after install" -BackgroundColor Red -ForegroundColor White
            Write-Host "http://support.microsoft.com/kb/$kbnum", $winversion.OSArchitecture
            SLEEP 5
            CMD /c "`"C:\Program Files (x86)\Internet Explorer\iexplore.exe`" http://support.microsoft.com/kb/$kbnum"
            $global:preReq = $false
            BREAK
        }     
}##//updateCheck##
    

### ipkBox ###
Function ipkBox
{

    Add-Type -AssemblyName System.Windows.Forms
    Add-Type -AssemblyName System.Drawing

    $global:form = New-Object System.Windows.Forms.Form
    $global:form.Text = "$esuVer"
    $global:form.Size = New-Object System.Drawing.Size(350, 200)
    $global:form.StartPosition = 'CenterScreen'

    $OKButton = New-Object System.Windows.Forms.Button  
    $OKButton.Location = New-Object System.Drawing.Point(125,100)  
    $OKButton.Size = New-Object System.Drawing.Size(75,23)  
    $OKButton.Text = 'OK'  
    $OKButton.DialogResult = [System.Windows.Forms.DialogResult]::OK  
    $global:form.AcceptButton = $OKButton  
    $global:form.Controls.Add($OKButton)

    $keyBox = New-Object System.Windows.Forms.TextBox
    $keyBox.Location = New-Object System.Drawing.Point(65,47)
    $keyBox.Size = New-Object System.Drawing.Size(200)
    $keyBox.Text = 'Enter Key...'
    $global:form.Controls.Add($keyBox)
    
    $comboBox = New-Object System.Windows.Forms.ComboBox
    $comboBox.Location = New-Object System.Drawing.Point(65, 70)
    $comboBox.Size = New-Object System.Drawing.Size(200)
    $comboBox.Text = "Year 1"
    $boxAnswers = @("Year 1", "Year 2", "Year 3")
    $comboBox.Items.AddRange($boxAnswers)
    $global:form.Controls.Add($comboBox)
    
    $link = New-Object System.Windows.Forms.LinkLabel
    $link.Location = New-Object System.Drawing.Point(90, 140)
    $link.Size = New-Object System.Drawing.Size(220, 20)
    $link.Text = 'https://aka.ms/Windows7ESU'
    $global:form.Controls.Add($link)
    $link.add_LinkClicked(
        {CMD.EXE /c "`"C:\Program Files (x86)\Internet Explorer\iexplore.exe`" https://aka.ms/Windows7ESU"})

    $mainText = New-Object System.Windows.Forms.Label  
    $mainText.Location = New-Object System.Drawing.Point(65,20)  
    $mainText.Size = New-Object System.Drawing.Size(220, 20)  
    $mainText.Text = 'Enter your Windows 7 ESU Product Key:'  
    $global:form.Controls.Add($mainText)  
    $result = $Global:form.ShowDialog()  
    SLEEP 1  #testing Topmost lag  
    $global:form.Topmost = $true  
    

    
      
    #OK Button ...   
    if($result -eq [System.Windows.Forms.DialogResult]::OK)  
    {  
        $global:key = $keyBox.Text  
        $global:box = $comboBox.SelectedItem
    }
    Else{Write-Host "Aborted..."; BREAK}              

}

### boxCheck ###
Function boxCheck
{
    if($box -eq $null)
    {$global:ActID = "77db037b-95c3-48d7-a3ab-a9c6d41093e0"}
        Else{
                if($box.Contains(1))
                {$global:ActID = "77db037b-95c3-48d7-a3ab-a9c6d41093e0"}
                    elseif($box.Contains(2))
                    {$global:ActID = "0e00c25d-8795-4fb7-9572-3803d91b6880"}
                        elseif($box.Contains(3))
                        {$global:ActID = "4220f546-f522-46df-8202-4d07afd26454"}
            }
}##//boxCheck##


## MAIN ##

# list of KBs to check
$kbs = @( "4490628",
          "4474419",
          "4538483")

## perform checks ##
sp1Check
updateQuery

foreach($kb in $kbs){updateCheck $kb}

if($preReq -ne $false)
{
    ## IPK & ATO ##
    ipkBox
        
    Write-Host "Key Entered = ", $key

    ## slmgr commands and year selection check ##
    slmgr.vbs /IPK $key
    boxCheck
    Start-Sleep 5
    Write-Host "Activation ID Selected = ", $ActID
    slmgr.vbs /ATO $ActID

    ###stop transcript###
    if($host.name.Contains("Console")){Stop-Transcript}
}

## pause ##
Write-Host "Hit any Key to Exit..."
cmd /c pause | out-null

## EOF JohnEM 2020 ##
```

