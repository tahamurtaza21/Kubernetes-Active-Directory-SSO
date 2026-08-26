# Kubernetes-Active-Directory-SSO
Kubernetes cluster where kubectl authenticates once against Windows Active Directory using Dex as an OIDC broker. Built and documented on a 4-node (2 control planes and 2 workers) Ubuntu 22.04 home lab.

This setup assumes that you already have a working Kubernetes Cluster. 

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

Run this from your Kubernetes Control Plane Server, not the domain controller. It proves the account
works from where Dex will actually use it.

You will be prompted for the administrator password of the domain controller. 

```bash
ldapsearch -x -H ldap://<YOUR_IP>:389 \
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

### 6. Point `dex.lab.local` at a node

Dex is exposed as a NodePort service on port 32000, which means the port is
open on *every* node in the cluster — traffic to any of them reaches the Dex
pod wherever it happens to be scheduled. So `dex.lab.local` just needs to
resolve to one node consistently.

**Consistently is the important word.** The API server resolves this name to
call Dex's OIDC discovery endpoint, and it does so on both control planes. If
they resolve it differently — or if the name has multiple A records and
round-robins — authentication succeeds or fails depending on which API server
the request lands on, with no obvious pattern.

I pointed it at my first control plane node: a control plane is the node
least likely to be powered off in a lab, and it keeps the entry point on the
machine I work from. Any node would function.

Add the entry on **all the control planes** — they're the hosts that must have
it, since the API server is what resolves the name:

```bash
# On <CONTROL-PLANE-NODE-1> AND <CONTROL-PLANE-NODE-2>
echo "<CONTROL-PLANE-NODE-1-IP>  dex.lab.local" | sudo tee -a /etc/hosts
```

Verify on each node. This must return exactly one address, identical on both:

```bash
getent hosts dex.lab.local
```

```
<CONTROL-PLANE-NODE-1-IP>  dex.lab.local
```

### 7. Generate the Dex certificate

Dex serves HTTPS, so it needs a certificate before it will start.

The OpenSSL config and command below are taken from the
[upstream Dex guide](https://dexidp.io/docs/guides/kubelogin-activedirectory/),
with two changes explained underneath.

A certificate is an ID card that says "I am `dex.lab.local`." When something
connects, TLS checks whether the name it typed matches a name on the card. The
`[alt_names]` section is that list of names.

**Change 1 — added the node's IP as a second name. This is primarily for debugging purposes** 
The guide lists the hostname alone... 

When login breaks, you need to know whether the name is the problem or Dex is the problem. Hitting the IP directly skips name resolution entirely — if that works, the name is broken; if it doesn't, Dex is broken.

```bash
cat > req.cnf <<'EOF'
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name

[req_distinguished_name]

[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = dex.lab.local
IP.1 = <CONTROL-PLANE-NODE-1-IP>
EOF

openssl req -new -x509 -sha256 -days 3650 -newkey rsa:4096 \
  -extensions v3_req -out tls.crt -keyout tls.key \
  -config req.cnf -subj "/CN=dex.lab.local" -nodes
```

Check before moving on:

```bash
openssl x509 -in tls.crt -noout -text | grep -A2 "Subject Alternative Name"
```

Expected output:

```
X509v3 Subject Alternative Name:
    DNS:dex.lab.local, IP Address:192.168.79.141
```
Both names must be listed (dex.lab.local and your ***CONTROL-PLANE-NODE-1-IP***. 
If the section is empty or absent, regenerate the certificate — the config file wasn't picked up.

![Group membership](docs/screenshots/05-certificate-verification.png)

### 9. Copy the certificate to both control planes

Ensure that you copy the certificate to /etc/kubernetes/pki/***name-of-certificate.crt***

The API server has to verify Dex's certificate when it calls the OIDC
discovery endpoint. Dex's certificate is self-signed, so nothing trusts it by
default — the API server needs to be handed the certificate explicitly and
told to trust it via `--oidc-ca-file`.

### 10. Create the namespace and secrets

Dex needs two secrets to exist before it starts. It mounts both, and a missing
one leaves the pod stuck in `ContainerCreating`.

```bash
kubectl create namespace dex
```

**TLS secret** — the certificate and key Dex serves HTTPS with.

```bash
kubectl create secret tls dex-tls \
  --cert=$HOME/dex-certs/tls.crt \
  --key=$HOME/dex-certs/tls.key \
  -n dex
```

**LDAP bind password** — for the `dex-service` account from step 4.

```bash
kubectl create secret generic dex-ldap-bind \
  --from-literal=bindPW='<dex-service password>' \
  -n dex
```

Use single quotes, or bash will mangle any `!`, `$`, or `&` in the password.

The password goes in a Secret rather than in the Dex config file so the config
can be committed to this repo as-is. The upstream guide puts it straight in the
config, which is fine for a local demo but not for anything in git.

Verify:

```bash
kubectl get secrets -n dex
```

```
NAME             TYPE                DATA   AGE
dex-ldap-bind    Opaque              1      10s
dex-tls          kubernetes.io/tls   2      30s
```

`dex-tls` should be type `kubernetes.io/tls` with 2 keys. If it says `Opaque`,
it was created with the wrong command.

![Secrets created](docs/screenshots/06-dex-secrets.png)

### 11. Create the Dex config

This tells Dex where Active Directory is, how to find a user, and how to find
which groups they're in.

Save as `manifests/dex/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dex-config
  namespace: dex
data:
  config.yaml: |
    # Dex's own address. Must be identical to --oidc-issuer-url on the
    # API server, or the API server won't trust the tokens Dex issues.
    issuer: https://dex.lab.local:32000/dex

    # Where Dex saves login sessions. "kubernetes" = store them in the
    # cluster, so Dex keeps nothing on disk and can restart safely.
    storage:
      type: kubernetes
      config:
        inCluster: true

    # Dex serves HTTPS. These files come from the dex-tls secret,
    # mounted at /etc/dex/tls by the Deployment in step 13.
    web:
      https: 0.0.0.0:5556
      tlsCert: /etc/dex/tls/tls.crt
      tlsKey: /etc/dex/tls/tls.key

    oauth2:
      # Skip the "allow this app?" page. One less click.
      skipApprovalScreen: true

    # Which app is allowed to ask Dex for a login. Here, that's kubectl.
    staticClients:
    - id: kubernetes
      name: Kubernetes
      secret: ZXhhbXBsZS1hcHAtc2VjcmV0
      # After login, Dex sends the browser back here. kubelogin is
      # listening on one of these ports on your own machine.
      redirectURIs:
      - http://localhost:8000
      - http://localhost:18000
      # Needed only for the browser-less login flow — see step 17.
      - urn:ietf:wg:oauth:2.0:oob

    connectors:
    - type: ldap
      id: ldap
      name: Active Directory
      config:
        host: 192.168.79.132:389
        insecureNoSSL: true    # plain LDAP, not LDAPS

        # The account Dex logs in as to search the directory.
        # Password comes from the dex-ldap-bind secret that we did earlier, not from here.
        bindDN: CN=Dex Service,CN=Users,DC=lab,DC=local
        bindPW: $LDAP_BIND_PW

        # Finding the person logging in.
        userSearch:
          baseDN: CN=Users,DC=lab,DC=local   # where to look
          filter: "(objectClass=user)"       # only look at users
          username: sAMAccountName           # what they type to log in
          idAttr: distinguishedName          # their unique id
          emailAttr: mail                    # becomes their k8s username
          nameAttr: cn                       # their display name

        # Finding their groups.
        groupSearch:
          baseDN: CN=Users,DC=lab,DC=local
          filter: "(objectClass=group)"      # only look at groups
          # Match a group if its member list contains this user.
          userMatchers:
          - userAttr: distinguishedName
            groupAttr: member
          nameAttr: cn                       # token says "k8s-admins",
                                             # not the full long DN
```

Apply it:

```bash
kubectl apply -f manifests/dex/configmap.yaml
```

**What is that `secret` under staticClients?**

It's a shared password between Dex and kubectl — not an AD password, and
nothing to do with users.

Dex only issues tokens to applications it recognises. `staticClients` is that
list, and it has one entry: kubectl. When kubectl exchanges its login code for
a token, it sends this string to prove it's the app Dex expects rather than
something else pointed at the same endpoint. The value here must match the
`--oidc-client-secret` kubectl is configured with in step 17.

`ZXhhbXBsZS1hcHAtc2VjcmV0` is the placeholder from the upstream guide (base64
of "example-app-secret"). Since it only has to match on both sides, any string
works — a real deployment would generate a random one.

### 12. Give Dex permission to manage its own storage

Step 11 set `storage: type: kubernetes`, which means Dex keeps login sessions
and tokens in the cluster as custom resources rather than in a database. To do
that it has to create those resource types and read and write them — so it
needs a ServiceAccount with permissions of its own.

**This has nothing to do with user access.** It's Dex's own housekeeping.
Mapping AD groups to cluster permissions is a separate step (16), and the two
are easy to confuse because both involve ClusterRoles.

Save as `manifests/dex/rbac.yaml`:

```yaml
# The identity the Dex pod runs as.
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dex
  namespace: dex
---
# What that identity is allowed to do.
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: dex
rules:
# Full access to Dex's own custom resources — where it stores auth
# codes, refresh tokens and sessions.
- apiGroups: ["dex.coreos.com"]
  resources: ["*"]
  verbs: ["*"]
# Permission to create those resource types in the first place.
# Dex registers them itself on first start.
- apiGroups: ["apiextensions.k8s.io"]
  resources: ["customresourcedefinitions"]
  verbs: ["create"]
---
# Tie the two together.
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dex
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: dex
subjects:
- kind: ServiceAccount
  name: dex
  namespace: dex
```

Apply it **before** the Deployment. Without it the pod starts, tries to
register its CRDs, gets refused, and crash-loops with a permissions error that
looks like a config problem:

```bash
kubectl apply -f manifests/dex/rbac.yaml
```


