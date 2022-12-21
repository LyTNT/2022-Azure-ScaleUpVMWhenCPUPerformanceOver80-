# 2022-Azure-ScaleUpVMWhenCPUPerformanceOver80
# A. Overview
1. Purpose of this presentation:
- If CPU Performance >10% within 1 min, VM will automatically scale up to higher level of configuration
- Based on this flow, User can set up other automation cases for VM on their own such as Scale down, Restart, Stop or Start 
2. Scale up Rule: VM sizes scaling pair 
Ex: | | Basic_A0 | Basic_A4 | | Standard_A0 | Standard_A4 | | Standard_A5 | Standard_A7 | | Standard_A8 | Standard_A9 | | Standard_A10 | Standard_A11 | | Standard_D1 | Standard_D4 | | Standard_D11 | Standard_D14 | | Standard_DS1 | Standard_DS4 | | Standard_DS11 | Standard_DS14 | | Standard_D1v2 | Standard_D5v2 | | Standard_D11v2 | Standard_D14v2 | | Standard_G1 | Standard_G5 | | Standard_GS1 | Standard_GS5 |
3. Issue I faced on this task:
- Select the Runbook version that is not compatible with the Module in Automation Account
- Add Role for User-assigned managed identity in the wrong way, so running fails

# B. Summary of flow
## I> Create User-assigned managed identity

Ref: https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/how-manage-user-assigned-managed-identities?pivots=identity-mi-methods-azp

**1. There are 2 ways approaching Automation Account**
- system-assigned managed identity
- User assign managed identity

I follow the second one

![image](https://user-images.githubusercontent.com/103499666/208809739-befd445c-9600-4d8f-a660-7f27a76f785c.png)

**2 Assign permissions to managed identities** 

Ref: https://learn.microsoft.com/en-us/azure/automation/automation-create-alert-triggered-runbook

Follow steps on the above link, summary as follows:

**2.1 Provide variables**
```
$resourceGroup = "resourceGroup"
$automationAccount = "AutomationAccount"
$userAssignedManagedIdentity = "userAssignedManagedIdentity"
```
```
$resourceGroup = "testscaleuprg"
$automationAccount = "lytntautoaccount"
$userAssignedManagedIdentity = "UAI1"
```
**2.2 Use PowerShell cmdlet New-AzRoleAssignment to assign a role to the system-assigned managed identity.**
```
$SAMI = (Get-AzAutomationAccount -ResourceGroupName $resourceGroup -Name $automationAccount).Identity.PrincipalId
New-AzRoleAssignment `
    -ObjectId $SAMI `
    -ResourceGroupName $resourceGroup `
    -RoleDefinitionName "Contributor"
```
##Role **"Virtual Machine Contributor"** may be okay

##Wait result this step

**2.3 Assign a role to a user-assigned managed identity.**
```
$UAMI = (Get-AzUserAssignedIdentity -ResourceGroupName $resourceGroup -Name $userAssignedManagedIdentity)
New-AzRoleAssignment `
    -ObjectId $UAMI.PrincipalId `
    -ResourceGroupName $resourceGroup `
    -RoleDefinitionName "Contributor"
```
##Wait result this step

**2.4 final**

>$UAMI.ClientId

##save id: 6132e277-8327-41e5-9671-105033dad25d

##img2: the result as line3 in the image

![image](https://user-images.githubusercontent.com/103499666/208814115-3daa48cf-f5a7-4563-ada4-e22db0c90c8e.png)


## II> Tạo Virtual VM- Ubuntu
The Cheapest VM config: **Standard B1s (1 vcpu, 1 GiB memory)**

upgraded vm=> **_Standard B2s (2 vcpus, 4 GiB memory)_**
- Networking: open port 22
- Tick Delete Public ip when VM is deleted
- Enable System assigned managed identity
- Login with Azure AD
- Download key and create vm

## III> Create Automation Account and Runbook
- Tab Advanced, tick System assigned và User assigned
- Add user assigned identities, ex **UAI1**
- Networking: choose Public access

![image](https://user-images.githubusercontent.com/103499666/208814425-9a53d6f6-52d0-4cc1-a9cb-91f6d7bc6543.png)

## III.2 Create Runbook for scale up VM
- Name:
- Runbook Type: PowerShell
- Runtime version: 5.1
**Note:This version will compatible with Module (in Automation Account-vd:lytntautoaccount=> Module)**
- Code của Runbook có thể Import từ Gallery

EX: **ScaleUp-Azure-VM-On-Alert**

author: azureautomation
- Save=> Publish

![image](https://user-images.githubusercontent.com/103499666/208814972-5671588d-7641-4fd8-810b-c19f4c95b13f.png)

## IV. Create Alert

Ref: https://learn.microsoft.com/en-us/azure/automation/automation-create-alert-triggered-runbook

**1 Scope**
- Filter by resource type: Virtual machines
- Click choose vm you want to scale up

##img5: Interface of Alert rule scope

![image](https://user-images.githubusercontent.com/103499666/208816651-04c70f1a-1b79-418f-a347-bb0dfed9d2cf.png)

**2. Condition**
- choose CPU Percentage

#If CPU percentage >10% within 1 min => alert fired
- Threshold value: 10
- Lookback period: 1

![image](https://user-images.githubusercontent.com/103499666/208816743-9518c92a-bd87-4ef7-b8ad-1cb9329698fc.png)

**3. Action**

Create action group

- Send email when alert fired
- Action type: Automation Runbook

 + Runbook Source: User
 + Subscription -> select Automation account created above -> select runbook of scaleup published
 + Enable Alert Schema
 
 ![image](https://user-images.githubusercontent.com/103499666/208816862-af624980-8641-4d7b-8149-dd5497b2909b.png)

**4. Name for alert and create**

**5. Check alert rule attached on VM**

![image](https://user-images.githubusercontent.com/103499666/208817005-b3551a3d-16f6-4262-8f0e-4ad24d9c511c.png)

## V> Access VM and test
#git bash
```
cd ~
cd Desktop/
```
#Connect to VM
>ssh -i misasinglevm_key.pem azureuser@20.234.37.207

#Download stress module to push CPU upto 100%

```
sudo apt-get update
sudo apt-get -y install stress
sudo stress --cpu 1 --timeout 2000
```
#To observe, open new git window, type command:
>htop

#Result: Check mail and Refresh VM
-	orgiginal vm: **Standard B1s (1 vcpu, 1 GiB memory)**
-	upgraded vm: **_Standard B2s (2 vcpus, 4 GiB memory)_**










