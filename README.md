Tag based VM Automatic Startup and Shutdown WITH dependency rules
=================================================================

            

This script can be used as Azure Runbook (scheduled / on demand) to stop/start VMs. It uses tags in an intelligent way to automate this.


You'll give your VMs a startup and shutdown tag in hh:mm format, which defines the window in which the VM should be running.


Optionally, you can give your VMs a dependsOn tag, to specify which VMs this VM depends on, the script will ensure that those are started first.


Script home: [http://www.lieben.nu](http://www.lieben.nu)




PowerShell
Edit|Remove
powershell
#Author: Jos Lieben (OGD), Peter Takacs (OGD)
#Date: 01-06-2017
#Script home: http://www.lieben.nu
#Copyright: MIT
#Purpose: automatically start and stop VM's based on their tags
#Requires –Version 5
#Name: azureTagbasedAutoStartupShutdown

<#Notes / features

>> Each target VM should have a 'Startup' and 'Shutdown' tag in hh:mm format
>> Each target VM should have a depensOn tag IF it depends on other VM's that should first be in a running state
>> Each vm with a dependsOn tag should have this tag filled in JSON format with the names of the vm's it depends on, example: ['vname1','vname2']
>> A PS Credential with valid permissions should be present to log in to Azure
>> Subscription ID in which to run
>> Powershell V5

#>

Param(
    [Parameter(Mandatory=$true)]$subscriptionId, #ID of subscription which houses the target VM's
    $automationPSCredentialName='AzureStartUpShutdownCreds',#name of the Azure credential in your credential store
    $maxRunTimeInSeconds=3600, #maximum time this script will wait for all VM's to start
    [Switch]$testMode #if specified, won't actually start or stop a VM
)

try{
    $azureCreds = Get-AutomationPSCredential -Name $automationPSCredentialName
}catch{
    Write-Error 'Failed to load azure credentials from credential object'
    Exit
}

try{
    Write-Output 'Logging in to Azure'
    $res = Login-AzureRmAccount -Credential $azureCreds -ErrorAction Stop
    Write-Output 'Logged in to Azure'
}catch{
    Throw
    Exit
}

try{
    Write-Output 'Retrieving virtual machines with a Startup and Shutdown tag'
    $targetVMs = Find-AzureRmResource -TagName 'Startup'| Where-Object {$_.Resourcetype -eq 'Microsoft.Compute/virtualMachines'} | Get-AzureRmVM | where {$_.Tags.Keys -contains 'Shutdown' -and $_.Tags.Keys -contains 'Startup'}
    $targetVMs | add-member -MemberType NoteProperty -Name StartupShutdownProcessed -Value $False
    $count = 0
    #check current status
    foreach($VM in $targetVMs){
        $currentVMState = (Get-AzureRmVM -Name $VM.Name -ResourceGroupName $VM.ResourceGroupName -Status -ErrorAction Stop).Statuses[1].Code
        if($currentVMState -eq 'Powerstate/Stopped' -or $currentVMState -eq 'Powerstate/Deallocated'){
            $targetVMs[$count] | add-member -MemberType NoteProperty -Name currentlyRunning -Value $False   
        }else{
            $targetVMs[$count] | add-member -MemberType NoteProperty -Name currentlyRunning -Value $True
        }
        $count++
    }
    if($targetVMs.Count -gt 0){
        Write-Output '$($targetVMs.Count) VMs retrieved'
    }else{
        Throw 'No virtual machines retrieved, script will not continue'
    }
}catch{
    Throw
    Exit
}

$runTimeInSeconds = 0
$firstLoop = $True
while($runTimeInSeconds -le $maxRunTimeInSeconds){
    [DateTime]$dateTime = Get-Date
    Write-Verbose 'Current time: $(Get-Date)'
    #exit loop when all VM's have been started/stopped
    [Array]$unprocessedVMs = @($targetVMs | where {$_.StartupShutdownProcessed -eq $False -and $_})
    if($unprocessedVMs.Count -eq 0){
        Write-Output 'No more virtual machines in unprocessed state, job completed'
        break
    }
    $count = -1
    foreach($VM in $targetVMs){
        $count++
        #skip if already processed
        if($VM.StartupShutdownProcessed){
            continue
        }
        Write-Verbose 'VM $($VM.Name) start processing, VM window $($VM.Tags['Startup']) - $($VM.Tags['Shutdown'])'

        #first, retrieve current VM status
        try{
            $currentVMState = (Get-AzureRmVM -Name $VM.Name -ResourceGroupName $VM.ResourceGroupName -Status -ErrorAction Stop).Statuses[1].Code
            if($currentVMState -eq 'Powerstate/Stopped' -or $currentVMState -eq 'Powerstate/Deallocated'){
                $targetVMs[$count].currentlyRunning = $False  
                $VM.currentlyRunning = $False 
            }else{
                $targetVMs[$count].currentlyRunning = $True 
                $VM.currentlyRunning = $True 
            }
            Write-Verbose 'VM $($VM.Name) state: $currentVMState'
        }catch{
            Write-Error 'Failed to retrieve Power State for VM $($VM.Name) because of $($Error[0])'
            continue #skip to next VM to avoid processing without valid data
        }

        #determine if this VM should be started or stopped by its time tags and the current datetime
        if($dateTime -gt [DateTime]$VM.Tags['Startup'] -and $dateTime -lt [DateTime]$VM.Tags['Shutdown']){
            #VM should be started according to time tags
            if($VM.currentlyRunning -eq $False){
                #VM is stopped, check if it depends on any other VM's or can be started safely
                $allowedToStart = $True
                if($VM.Tags['dependsOn']){
                    #skip on the first loop as we're still evaluating states
                    if($firstLoop){
                        Write-Verbose 'VM $($VM.Name) not starting yet, VM's without dependencies are processed first'
                        continue
                    }
                    try{
                        [Array]$dependsOnList = $VM.Tags['dependsOn'] | ConvertFrom-Json -ErrorAction Stop
                        Write-Verbose 'VM $($VM.Name) has $($dependsOnList.Count) dependencies, checking their status:'
                        foreach($dependency in $dependsOnList){
                            $dependentVM = $targetVMs | where {$_.Name -eq $dependency -and $_.currentlyRunning -eq $False -and $_}
                            if($dependentVM){
                                $allowedToStart = $False
                                Write-Verbose '$dependency ---> NOT STARTED YET'
                            }else{
                                Write-Verbose '$dependency ---> STARTED'
                            }
                        }
                    }catch{
                        Write-Error 'VM $($VM.Name) has a dependsOn tag, but it does not contain valid JSON data in the [`'vmname1`',`'vmname2`'] JSON format: $($Error[0])'
                        $allowedToStart = $False
                    }
                }else{
                    Write-Verbose 'VM $($VM.Name) should be started, it has no dependencies'
                }
                if($allowedToStart){
                    try{
                        Write-Output '$($VM.Name) starting...'
						if(!$testMode) {$results = Start-AzureRmVM -Name $VM.Name -ResourceGroupName $VM.ResourceGroupName -ErrorAction Stop}
						Write-Output '$($VM.Name) started: $($results.status) command duration: $($results.Starttime) - $($results.Endtime)'
                        $targetVMs[$count].StartupShutdownProcessed = $True 
                        $targetVMs[$count].currentlyRunning = $True 
                    }catch{
                        Write-Error 'Failed to start VM $($VM.Name) because of $($Error[0])'
                    }
                }else{
                    Write-Verbose 'VM $($VM.Name) not allowed to start yet'
                }
            }else{
                #VM is already started and should be removed from the loop
                Write-Verbose 'VM $($VM.Name) has been started'
                $targetVMs[$count].StartupShutdownProcessed = $True
                $targetVMs[$count].currentlyRunning = $True 
            }
        }else{
            #VM should be stopped according to time tags
            if($VM.currentlyRunning){
                #VM is started, check if any other VMs depend on it or if it can be stopped safely
                Write-Verbose 'VM $($VM.Name) should be stopped, checking any VMs that depend on it'
                $allowedToStop = $True
                foreach($reliant in $targetVMs){
                    if($reliant.currentlyRunning){
                        if($reliant.Tags['dependsOn']){
                            try{
                                [Array]$dependsOnList = $reliant.Tags['dependsOn'] | ConvertFrom-Json -ErrorAction Stop
                                foreach($dependency in $dependsOnList){
                                    if($dependency -eq $VM.Name){
                                        $allowedToStop = $False
                                        Write-Verbose '$($reliant.Name) ---> NOT STOPPED YET'
                                    }
                                }
                            }catch{
                                Write-Error 'VM $($VM.Name) has a dependsOn tag, but it does not contain valid JSON data in the [`'vmname1`',`'vmname2`'] JSON format: $($Error[0])'
                                $allowedToStop = $False
                            }
                        }                        
                    }
                }
                if($allowedToStop){
                    try{
                        Write-Output '$($VM.Name) stopping...'
						if(!$testMode) {$results = Stop-AzureRmVM -Name $VM.Name -ResourceGroupName $VM.ResourceGroupName -Force -Confirm:$False -ErrorAction Stop}
						Write-Output '$($VM.Name) stopped: $($results.status) command duration: $($results.Starttime) - $($results.Endtime)'
                        $targetVMs[$count].StartupShutdownProcessed = $True 
                        $targetVMs[$count].currentlyRunning = $False
                    }catch{
                        Write-Error 'Failed to start VM $($VM.Name) because of $($Error[0])'
                    }                    
                }else{
                    Write-Verbose 'VM $($VM.Name) not allowed to stop yet'
                }
            }else{
                #VM is already stopped
                Write-Verbose 'VM $($VM.Name) has been stopped'
                $targetVMs[$count].StartupShutdownProcessed = $True
                $targetVMs[$count].currentlyRunning = $False
            }
        }
    }

    $firstLoop = $False
    #wait a while before looping again
    Sleep -s 10
    $runtimeInSeconds += 10
}

#Author: Jos Lieben (OGD), Peter Takacs (OGD) 
#Date: 01-06-2017 
#Script home: http://www.lieben.nu 
#Copyright: MIT 
#Purpose: automatically start and stop VM's based on their tags 
#Requires –Version 5 
#Name: azureTagbasedAutoStartupShutdown 
 
<#Notes / features 
 
>> Each target VM should have a 'Startup' and 'Shutdown' tag in hh:mm format 
>> Each target VM should have a depensOn tag IF it depends on other VM's that should first be in a running state 
>> Each vm with a dependsOn tag should have this tag filled in JSON format with the names of the vm's it depends on, example: ['vname1','vname2'] 
>> A PS Credential with valid permissions should be present to log in to Azure 
>> Subscription ID in which to run 
>> Powershell V5 
 
#> 
 
Param( 
    [Parameter(Mandatory=$true)]$subscriptionId, #ID of subscription which houses the target VM's 
    $automationPSCredentialName='AzureStartUpShutdownCreds',#name of the Azure credential in your credential store 
    $maxRunTimeInSeconds=3600, #maximum time this script will wait for all VM's to start 
    [Switch]$testMode #if specified, won't actually start or stop a VM 
) 
 
try{ 
    $azureCreds = Get-AutomationPSCredential -Name $automationPSCredentialName 
}catch{ 
    Write-Error 'Failed to load azure credentials from credential object' 
    Exit 
} 
 
try{ 
    Write-Output 'Logging in to Azure' 
    $res = Login-AzureRmAccount -Credential $azureCreds -ErrorAction Stop 
    Write-Output 'Logged in to Azure' 
}catch{ 
    Throw 
    Exit 
} 
 
try{ 
    Write-Output 'Retrieving virtual machines with a Startup and Shutdown tag' 
    $targetVMs = Find-AzureRmResource -TagName 'Startup'| Where-Object {$_.Resourcetype -eq 'Microsoft.Compute/virtualMachines'} | Get-AzureRmVM | where {$_.Tags.Keys -contains 'Shutdown' -and $_.Tags.Keys -contains 'Startup'} 
    $targetVMs | add-member -MemberType NoteProperty -Name StartupShutdownProcessed -Value $False 
    $count = 0 
    #check current status 
    foreach($VM in $targetVMs){ 
        $currentVMState = (Get-AzureRmVM -Name $VM.Name -ResourceGroupName $VM.ResourceGroupName -Status -ErrorAction Stop).Statuses[1].Code 
        if($currentVMState -eq 'Powerstate/Stopped' -or $currentVMState -eq 'Powerstate/Deallocated'){ 
            $targetVMs[$count] | add-member -MemberType NoteProperty -Name currentlyRunning -Value $False    
        }else{ 
            $targetVMs[$count] | add-member -MemberType NoteProperty -Name currentlyRunning -Value $True 
        } 
        $count++ 
    } 
    if($targetVMs.Count -gt 0){ 
        Write-Output '$($targetVMs.Count) VMs retrieved' 
    }else{ 
        Throw 'No virtual machines retrieved, script will not continue' 
    } 
}catch{ 
    Throw 
    Exit 
} 
 
$runTimeInSeconds = 0 
$firstLoop = $True 
while($runTimeInSeconds -le $maxRunTimeInSeconds){ 
    [DateTime]$dateTime = Get-Date 
    Write-Verbose 'Current time: $(Get-Date)' 
    #exit loop when all VM's have been started/stopped 
    [Array]$unprocessedVMs = @($targetVMs | where {$_.StartupShutdownProcessed -eq $False -and $_}) 
    if($unprocessedVMs.Count -eq 0){ 
        Write-Output 'No more virtual machines in unprocessed state, job completed' 
        break 
    } 
    $count = -1 
    foreach($VM in $targetVMs){ 
        $count++ 
        #skip if already processed 
        if($VM.StartupShutdownProcessed){ 
            continue 
        } 
        Write-Verbose 'VM $($VM.Name) start processing, VM window $($VM.Tags['Startup']) - $($VM.Tags['Shutdown'])' 
 
        #first, retrieve current VM status 
        try{ 
            $currentVMState = (Get-AzureRmVM -Name $VM.Name -ResourceGroupName $VM.ResourceGroupName -Status -ErrorAction Stop).Statuses[1].Code 
            if($currentVMState -eq 'Powerstate/Stopped' -or $currentVMState -eq 'Powerstate/Deallocated'){ 
                $targetVMs[$count].currentlyRunning = $False   
                $VM.currentlyRunning = $False  
            }else{ 
                $targetVMs[$count].currentlyRunning = $True  
                $VM.currentlyRunning = $True  
            } 
            Write-Verbose 'VM $($VM.Name) state: $currentVMState' 
        }catch{ 
            Write-Error 'Failed to retrieve Power State for VM $($VM.Name) because of $($Error[0])' 
            continue #skip to next VM to avoid processing without valid data 
        } 
 
        #determine if this VM should be started or stopped by its time tags and the current datetime 
        if($dateTime -gt [DateTime]$VM.Tags['Startup'] -and $dateTime -lt [DateTime]$VM.Tags['Shutdown']){ 
            #VM should be started according to time tags 
            if($VM.currentlyRunning -eq $False){ 
                #VM is stopped, check if it depends on any other VM's or can be started safely 
                $allowedToStart = $True 
                if($VM.Tags['dependsOn']){ 
                    #skip on the first loop as we're still evaluating states 
                    if($firstLoop){ 
                        Write-Verbose 'VM $($VM.Name) not starting yet, VM's without dependencies are processed first' 
                        continue 
                    } 
                    try{ 
                        [Array]$dependsOnList = $VM.Tags['dependsOn'] | ConvertFrom-Json -ErrorAction Stop 
                        Write-Verbose 'VM $($VM.Name) has $($dependsOnList.Count) dependencies, checking their status:' 
                        foreach($dependency in $dependsOnList){ 
                            $dependentVM = $targetVMs | where {$_.Name -eq $dependency -and $_.currentlyRunning -eq $False -and $_} 
                            if($dependentVM){ 
                                $allowedToStart = $False 
                                Write-Verbose '$dependency ---> NOT STARTED YET' 
                            }else{ 
                                Write-Verbose '$dependency ---> STARTED' 
                            } 
                        } 
                    }catch{ 
                        Write-Error 'VM $($VM.Name) has a dependsOn tag, but it does not contain valid JSON data in the [`'vmname1`',`'vmname2`'] JSON format: $($Error[0])' 
                        $allowedToStart = $False 
                    } 
                }else{ 
                    Write-Verbose 'VM $($VM.Name) should be started, it has no dependencies' 
                } 
                if($allowedToStart){ 
                    try{ 
                        Write-Output '$($VM.Name) starting...' 
                        if(!$testMode) {$results = Start-AzureRmVM -Name $VM.Name -ResourceGroupName $VM.ResourceGroupName -ErrorAction Stop} 
                        Write-Output '$($VM.Name) started: $($results.status) command duration: $($results.Starttime) - $($results.Endtime)' 
                        $targetVMs[$count].StartupShutdownProcessed = $True  
                        $targetVMs[$count].currentlyRunning = $True  
                    }catch{ 
                        Write-Error 'Failed to start VM $($VM.Name) because of $($Error[0])' 
                    } 
                }else{ 
                    Write-Verbose 'VM $($VM.Name) not allowed to start yet' 
                } 
            }else{ 
                #VM is already started and should be removed from the loop 
                Write-Verbose 'VM $($VM.Name) has been started' 
                $targetVMs[$count].StartupShutdownProcessed = $True 
                $targetVMs[$count].currentlyRunning = $True  
            } 
        }else{ 
            #VM should be stopped according to time tags 
            if($VM.currentlyRunning){ 
                #VM is started, check if any other VMs depend on it or if it can be stopped safely 
                Write-Verbose 'VM $($VM.Name) should be stopped, checking any VMs that depend on it' 
                $allowedToStop = $True 
                foreach($reliant in $targetVMs){ 
                    if($reliant.currentlyRunning){ 
                        if($reliant.Tags['dependsOn']){ 
                            try{ 
                                [Array]$dependsOnList = $reliant.Tags['dependsOn'] | ConvertFrom-Json -ErrorAction Stop 
                                foreach($dependency in $dependsOnList){ 
                                    if($dependency -eq $VM.Name){ 
                                        $allowedToStop = $False 
                                        Write-Verbose '$($reliant.Name) ---> NOT STOPPED YET' 
                                    } 
                                } 
                            }catch{ 
                                Write-Error 'VM $($VM.Name) has a dependsOn tag, but it does not contain valid JSON data in the [`'vmname1`',`'vmname2`'] JSON format: $($Error[0])' 
                                $allowedToStop = $False 
                            } 
                        }                         
                    } 
                } 
                if($allowedToStop){ 
                    try{ 
                        Write-Output '$($VM.Name) stopping...' 
                        if(!$testMode) {$results = Stop-AzureRmVM -Name $VM.Name -ResourceGroupName $VM.ResourceGroupName -Force -Confirm:$False -ErrorAction Stop} 
                        Write-Output '$($VM.Name) stopped: $($results.status) command duration: $($results.Starttime) - $($results.Endtime)' 
                        $targetVMs[$count].StartupShutdownProcessed = $True  
                        $targetVMs[$count].currentlyRunning = $False 
                    }catch{ 
                        Write-Error 'Failed to start VM $($VM.Name) because of $($Error[0])' 
                    }                     
                }else{ 
                    Write-Verbose 'VM $($VM.Name) not allowed to stop yet' 
                } 
            }else{ 
                #VM is already stopped 
                Write-Verbose 'VM $($VM.Name) has been stopped' 
                $targetVMs[$count].StartupShutdownProcessed = $True 
                $targetVMs[$count].currentlyRunning = $False 
            } 
        } 
    } 
 
    $firstLoop = $False 
    #wait a while before looping again 
    Sleep -s 10 
    $runtimeInSeconds += 10 
}





        
    
TechNet gallery is retiring! This script was migrated from TechNet script center to GitHub by Microsoft Azure Automation product group. All the Script Center fields like Rating, RatingCount and DownloadCount have been carried over to Github as-is for the migrated scripts only. Note : The Script Center fields will not be applicable for the new repositories created in Github & hence those fields will not show up for new Github repositories.
