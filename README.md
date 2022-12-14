# Integration AppScan Source (SAST) and AzureDevOps
</br>
It will help to Integrate AppScan Source on AzureDevOps. It will enable AzureDevOps to start scan, generate report, publish results to AppScan Source Database and AppScan Enterprise and check for Security Gate.<br>
<br>
Requirements:<br>
1 - AppScan Source in Windows Server (it was tested on Windows 2019).<br>
2 - Add AppScan Source bin folder to Windows PATH Environment Variable.<br>
3 - Install Azure Pipeline Agent for Windows in same Windows Server that has AppScan Source. https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-windows?view=azure-devops<br>
3.1 - Add Azure Pipeline Agentr as a Service.<br>
3.2 - Change User Service to same User that has access in AppScan Enterprise.<br>
4 - Enable Long Paths in Windows. It is not mandatory but I guess it will safe some troubleshoot time.<br>
5 - Create AppScan Enterprise token <install_dir>\bin\ounceautod.exe -u username -p password --persist.<br>
  Source: https://help.hcltechsw.com/appscan/Source/10.0.8/topics/ounce_auto_login.html <br>
  <br>

```yaml
variables:
- name: artifactFolder
  value: $(Pipeline.Workspace)\s\AltoroJ 3.1.1\build\libs
- name: artifactName
  value: altoromutual.war
- name: aseAppName
  value: AltoroJ
- name: aseHostname
  value: ase.lab.local
- name: aseToken
  value: C:\ProgramData\HCL\AppScanSource\config\ounceautod.token
- name: sevSecGw
  value: highIssues
- name: maxIssuesAllowed
  value: 10

trigger:
- master

pool: winappscan

steps:
- powershell: |
    cd 'AltoroJ 3.1.1'
    gradle build
  displayName: 'Step 1 - Building artifact'

- powershell: |
    "login_file $(aseHostname) $(aseToken) -acceptssl" | out-file script.scan
    "RUNAS AUTO" | out-file -append script.scan
    "oa `"$(artifactFolder)\$(artifactName)`" -appserver_type Tomcat7 -no_ear_project" | out-file -append script.scan
    "ra `"$(artifactFolder)\$(artifactName).ozasmt`" -scanconfig `"Normal scan`" -name `"$(artifactName)-$(Build.BuildNumber)`"" | out-file -append script.scan
    "report Findings zip `"$(artifactName).zip`" `"$(artifactFolder)\$(artifactName).ozasmt`" -includeSrcBefore:5 -includeSrcAfter:5 -includeTrace:definitive -includeTrace:suspect -includeHowToFix" | out-file -append script.scan
    "pa `"$(artifactFolder)\$(artifactName).ozasmt`"" | out-file -append script.scan
    "publishassessase `"$(artifactFolder)\$(artifactName).ozasmt`" -aseapplication `"$(aseAppName)`" -name `"$(artifactName)-$(Build.BuildNumber)`"" | out-file -append script.scan
    "exit" | out-file -append script.scan
  displayName: 'Step 2 - Creating config scan file'

- powershell: |
    AppScanSrcCli scr script.scan
  displayName: 'Step 3 - Scanning artifact and publishing SAST result'

- publish: $(Pipeline.Workspace)/s/$(artifactName).zip
  artifact: $(artifactName).zip
  continueOnError: On
  displayName: 'Step 4 - Publishing scan result'

- powershell: |
    [XML]$xml=Get-Content "$(artifactFolder)\$(artifactName).ozasmt"
    [int]$highIssues = $xml.AssessmentRun.AssessmentStats.total_high_finding
    [int]$mediumIssues = $xml.AssessmentRun.AssessmentStats.total_med_finding
    [int]$lowIssues = $xml.AssessmentRun.AssessmentStats.total_low_finding
    [int]$totalIssues = $highIssues+$mediumIssues+$lowIssues
    $maxIssuesAllowed = $maxIssuesAllowed -as [int]

    write-host "There is $highIssues high issues, $mediumIssues medium issues and $lowIssues low issues."
    write-host "The company policy permit less than $(maxIssuesAllowed) $(sevSecGw) severity."

    if (( $highIssues -gt $(maxIssuesAllowed) ) -and ( "$(sevSecGw)" -eq "highIssues" )) {
      write-host "Security Gate build failed";
      exit 1
      }
    elseif (( $mediumIssues -gt $(maxIssuesAllowed) ) -and ( "$(sevSecGw)" -eq "mediumIssues" )) {
      write-host "Security Gate build failed";
      exit 1
      }
    elseif (( $lowIssues -gt $(maxIssuesAllowed) ) -and ( "$(sevSecGw)" -eq "lowIssues" )) {
      write-host "Security Gate build failed";
      exit 1
      }
    elseif (( $totalIssues -gt $(maxIssuesAllowed) ) -and ( "($sevSecGw)" -eq "totalIssues" )) {
      write-host "Security Gate build failed";
      exit 1
      }
    else{
      write-host "Security Gate passed"
      }
  displayName: 'Step 5 - Checking Security Gate'
```
<br>Reference project: https://github.com/jayachandradev/Altroz
