# Kubernetes-Active-Directory-SSO
Kubernetes cluster where kubectl authenticates once against Windows Active Directory using Dex as an OIDC broker. Built and documented on a 4-node (2 control planes and 2 workers) Ubuntu 22.04 home lab.


###

### 1. Create the two AD Groups to verify RBAC

```powershell
New-ADGroup -Name "k8s-admins" -GroupScope Global -GroupCategory Security -Path "CN=Users,DC=lab,DC=local"
New-ADGroup -Name "k8s-developers" -GroupScope Global -GroupCategory Security -Path "CN=Users,DC=lab,DC=local"
```

### 2. Create the users

`-EmailAddress` is required, not cosmetic. The API server is configured with
`--oidc-username-claim=email`, so a user without a mail attribute has no
username and authentication fails.

```powershell
New-ADUser -Name "Taha Admin" -SamAccountName "taha.admin" `
  -UserPrincipalName "taha.admin@lab.local" `
  -EmailAddress "taha.admin@lab.local" `
  -Path "CN=Users,DC=lab,DC=local" `
  -AccountPassword (Read-Host -AsSecureString "Password") -Enabled $true

New-ADUser -Name "Dev User" -SamAccountName "dev.user" `
  -UserPrincipalName "dev.user@lab.local" `
  -EmailAddress "dev.user@lab.local" `
  -Path "CN=Users,DC=lab,DC=local" `
  -AccountPassword (Read-Host -AsSecureString "Password") -Enabled $true
```
![Users created in Active Directory](docs/screenshots/02-ad-users.png)

### 3. Assign group membership

```powershell
Add-ADGroupMember -Identity "k8s-admins" -Members "taha.admin"
Add-ADGroupMember -Identity "k8s-developers" -Members "dev.user"
```

Verify:

```powershell
Get-ADGroupMember -Identity "k8s-admins" | Select Name,SamAccountName
Get-ADGroupMember -Identity "k8s-developers" | Select Name,SamAccountName
```

![Group membership](docs/screenshots/03-group-membership.png)
