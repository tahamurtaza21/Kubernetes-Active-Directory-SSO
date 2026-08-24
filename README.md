# Kubernetes-Active-Directory-SSO
Kubernetes cluster where kubectl authenticates once against Windows Active Directory using Dex as an OIDC broker. Built and documented on a 4-node (2 control planes and 2 workers) Ubuntu 22.04 home lab.

###
The lab uses a single Active Directory domain (Windows Server 2019), `lab.local`, running on a
Windows Server VM. Every command below uses `DC=lab,DC=local` as the base DN —
substitute your own domain throughout.

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
![Users created in Active Directory](docs/screenshots/01-ad-users.png)

### 3. Create the groups and assign group membership

```powershell
New-ADGroup -Name "k8s-admins" -GroupScope Global -GroupCategory Security -Path "CN=Users,DC=lab,DC=local"
New-ADGroup -Name "k8s-developers" -GroupScope Global -GroupCategory Security -Path "CN=Users,DC=lab,DC=local"
```

```powershell
Add-ADGroupMember -Identity "k8s-admins" -Members "taha.admin"
Add-ADGroupMember -Identity "k8s-developers" -Members "dev.user"
```

Verify:

```powershell
Get-ADGroupMember -Identity "k8s-admins" | Select Name,SamAccountName
Get-ADGroupMember -Identity "k8s-developers" | Select Name,SamAccountName
```

![Group membership](docs/screenshots/02-ad-groups.png)

### 4. Create the Dex service account

This account is different from the two above. It isn't a person, it never logs
into the cluster, and it belongs to no groups.

**What it does.** Dex needs to look users up in Active Directory before it can
authenticate them — find the account matching a given username, and read which
groups that account belongs to. LDAP requires an authenticated connection to
run those searches, so Dex needs credentials of its own. That's this account.

**What it does not do.** It does not authenticate anyone. When a user logs in,
two separate LDAP binds happen:

1. Dex binds as `dex-service` and searches for the user
2. Dex binds again *as that user*, using the password they just typed —
   if AD accepts it, the password is correct

So the service account is a **directory reader**, not an authenticator. The
user's password is verified by AD itself, and Dex never stores it.

This is why it needs no privileges beyond directory reads. The upstream Dex
guide binds as `cn=Administrator`, which hands full domain rights to a
component that only needs to run searches. A dedicated account means that if
the Dex config leaks, the credential exposed can read the directory and
nothing else.

```powershell
New-ADUser -Name "Dex Service" -SamAccountName "dex-service" `
  -UserPrincipalName "dex-service@lab.local" `
  -Path "CN=Users,DC=lab,DC=local" `
  -AccountPassword (Read-Host -AsSecureString "Password") `
  -Enabled $true -PasswordNeverExpires $true -CannotChangePassword $true
```

`-PasswordNeverExpires` and `-CannotChangePassword` are deliberate: this is a
non-interactive account, and an expired password would silently break every
cluster login at once. No group membership is added — authenticated domain
users can read the directory by default, which is all Dex requires.

![Group membership](docs/screenshots/03-ad-dex.png)

### 5. Verify the service account can read the directory

Run this from `k8s-control`, not the domain controller. It proves the account
works from where Dex will actually use it.

You will be prompted for the administrator password of the domain controller. 

```bash
ldapsearch -x -H ldap://192.168.x.132:389 \
  -D "CN=Dex Service,CN=Users,DC=lab,DC=local" \
  -W \
  -b "CN=Users,DC=lab,DC=local" \
  "(sAMAccountName=taha.admin)" dn mail memberOf
```

Expected output:

```
dn: CN=Taha Admin,CN=Users,DC=lab,DC=local
mail: taha.admin@lab.local
memberOf: CN=k8s-admins,CN=Users,DC=lab,DC=local
```

![Group membership](docs/screenshots/04-ldapsearch.png)


