# Configuration to Access Kubernetes using Keycloak

Simple demonstration on how to manage access to kubernetes cluster using keycloak SSO.

The demonstration will implement access for Devs with limited privileges to view only **deployments** and **pods**, and Admin with clusterwide admin privileges.

This approach will reduce the overhead of manually granting access to the cluster. When integrated properly, the same user group and realm can be used to manage access across multiple application e.g ArgoCD, Grafana, etc.

## Table of Contents

1. [Requirements](#requirements)
2. [Setup keycloak user and group](#1-setup-keycloak-user-and-group)
3. [Create and configure Client in Keycloak](#2-create-and-configure-client-in-keycloak)
4. [Edit Kube-ApiServer config](#3-edit-kube-apiserver-config)
5. [Create Clusterroles](#4-create-clusterroles)
6. [Update kubeconfig to use oidc](#5-update-kubeconfig-to-use-oidc)
7. [Test access to the cluster](#6-test-access-to-the-cluster)
8. [Troubleshooting](#7-troubleshooting)
9. [Cleanup / Rollback](#8-cleanup--rollback)
10. [Next Steps](#next-steps)

## Requirements

1. Kubernetes cluster (kubeadm cluster setup)
2. Keycloak
3. kubectl-oidc-login plugin (kubelogin)
4. kubectl

For this demo, I will be using:

- kubernetes (v1.31.1)
- keycloak (26.4.6)
- kubectl (v1.34.1)

> I will not cover how to install and setup kubernetes and keycloak in this demo.
> You will need ssh access to the controlplane to complete this demo.

### Installing kubectl-oidc-login (kubelogin)

**MacOS:**
```bash
brew install kubelogin
```

**Linux:**
```bash
# Using Homebrew on Linux
brew install kubelogin

# Or download binary directly
VERSION=v1.28.0
curl -LO https://github.com/int128/kubelogin/releases/download/${VERSION}/kubelogin_linux_amd64.zip
unzip kubelogin_linux_amd64.zip
sudo mv kubelogin /usr/local/bin/
```

**Windows:**
```powershell
# Using Chocolatey
choco install kubelogin

# Or using Scoop
scoop install kubelogin
```

## 1. Setup keycloak user and group

In this section, I will create two users and two groups to represent Devs and Admins.

Be sure to create a new Realm in keycloak for the demo. If you do not know how to do so, please see keycloak documentation

### Create two users `(john.doe & jane.doe)`

1. After login, click on `Users` -> `Add user`
2. Fill out `username` with `john.doe` field and `Email` set to `john.doe@localhost`
3. Click on create and navigate to `Credentials` tab
4. Set desired password and save. Temporary can be turned off for this demo
5. Repeat the same steps for `jane.doe`

### Create two groups `k8s-cluster-admin & k8s-cluster-dev`

1. Click on `Groups` -> `Create group` button
2. Enter group name `k8s-cluster-admin` and create
3. Click on the newly created group -> `Members` tab
4. Add member `john.doe`
5. Repeat step 1-4 for `jane.doe` and add user to `k8s-cluster-dev` group

## 2. Create and configure Client in Keycloak

1. Click on Clients -> `Create client`
2. Set Client ID to `kubernetes` and Name to `kubernetes`
3. Turn on the `Client authentication`
4. Select `Standard flow` and `Service accounts roles`
5. On the next page, set `Valid redirect URIs` to `http://localhost:18000/*` and `http://localhost:8000/*`
   > These ports (8000 and 18000) are used by kubelogin for the OAuth callback after authentication
6. Set `Web origins` to `+`
   > The `+` allows all origins defined in Valid redirect URIs
7. In the `Credentials` tab, copy the client secret and save it somewhere. We will need it later

### Configure Client Mapper for Groups

This step is **critical** - without it, group membership won't be included in the token and RBAC won't work.

1. Navigate to the `kubernetes` client -> `Client scopes` tab
2. Click on `kubernetes-dedicated`
3. Click `Add mapper` -> `By configuration` -> `Group Membership`
4. Set the following:
   - Name: `groups`
   - Token Claim Name: `groups`
   - Full group path: `OFF`
   - Add to ID token: `ON`
   - Add to access token: `ON`
   - Add to userinfo: `ON`
5. Click `Save`

> This mapper ensures that group membership is included in the token claims so Kubernetes can perform RBAC based on groups.

## 3. Edit Kube-ApiServer config

1. ssh into controlplane
2. Edit kube-apiserver with editor of choice `vi /etc/kubernetes/manifests/kube-apiserver.yaml`
3. Add the following commandline parameters

```yaml
spec:
  containers:
    - command:
        - kube-apiserver
        # ... existing flags ...
        - --oidc-issuer-url=https://<YOUR-KEYCLOAK-ENDPOINT-HERE>/realms/<REALM-HERE>
        - "--oidc-username-prefix=oidc:"
        - "--oidc-groups-prefix=oidc:"
        - --oidc-client-id=kubernetes
        - --oidc-username-claim=preferred_username
        - --oidc-groups-claim=groups
```

**OIDC Flags Explained:**
- `--oidc-issuer-url`: The base URL of your Keycloak realm (must be HTTPS in production)
- `--oidc-username-prefix`: Prefix added to usernames to avoid conflicts (e.g., `oidc:john.doe`)
- `--oidc-groups-prefix`: Prefix added to group names (e.g., `oidc:k8s-cluster-admin`)
- `--oidc-client-id`: The client ID created in Keycloak (`kubernetes`)
- `--oidc-username-claim`: Which claim from the token to use as username (`preferred_username`)
- `--oidc-groups-claim`: Which claim contains group membership information (`groups`)

> Wait for ApiServer to be available again before proceeding. This typically takes 30-60 seconds. You can check with `kubectl get pods -n kube-system | grep kube-apiserver`

## 4. Create Clusterroles

The clusterrole represent the permission the user / group will assume after authentication.

### Create `k8s-cluster-dev` clusterrole

```bash
kubectl create clusterrole k8s-cluster-dev --verb=create,get,list --resource=pods,deployments
```

> I will not create a new clusterrole for k8s-cluster-admin because I will bind it to kubernetes cluster admin role.

### Create clusterrolebindings

```bash
# dev clusterrolebinding
kubectl create clusterrolebinding k8s-cluster-dev --clusterrole k8s-cluster-dev --group=oidc:k8s-cluster-dev
```

```bash
# admin clusterrolebinding
kubectl create clusterrolebinding k8s-cluster-admin --clusterrole cluster-admin --group=oidc:k8s-cluster-admin
```

## 5. Update kubeconfig to use oidc

Use editor of choice `vi ~/.kube/config` and add the following lines

```yaml
contexts:
  - context:
      cluster: kubernetes
      user: keycloak
    name: keycloak-context
users:
  - name: keycloak
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1beta1
        args:
          - oidc-login
          - get-token
          - --oidc-issuer-url=https://<YOUR-KEYCLOAK-ENDPOINT-HERE>/realms/<REALM-HERE>
          - --oidc-client-id=kubernetes
          - --oidc-client-secret=<CLIENT-SECRET-HERE>
          - --oidc-extra-scope=email,profile,groups
        command: kubectl
        env: null
        interactiveMode: IfAvailable
        provideClusterInfo: false
```

## 6. Test access to the cluster

We have to test the connectivity.

- Execute

```bash
kubectl config use-context keycloak-context
kubectl get pods -A
```

> This will open up the browser. Login using `john.doe` and your password to keycloak. You should be able to view cluster pods resource

- Invalidate the current user login by going to keycloak -> clients -> kubernetes -> Session. Click on the 3 dots on and sign out `john.doe` or use `rm -r ~/.kube/cache/oidc-login`

- Try again `kubectl get pods -A` and login with the user `jane.doe`. The user should be able to view pods

- Execute `kubectl get secrets -A`. You will get an error like `Error from server (Forbidden): secrets is forbidden: User "oidc:jane.doe" cannot list resource "secrets" in API group "" in the namespace "default"`

This way, you can manage access to the cluster via Keycloak.

## 7. Troubleshooting

### Browser doesn't open for authentication

**Problem:** When running `kubectl get pods -A`, the browser doesn't open automatically.

**Solution:**
- Manually copy the URL from the terminal and open it in your browser
- Check if port 8000 or 18000 is already in use: `lsof -i :8000` and `lsof -i :18000`
- Ensure kubelogin is properly installed: `kubectl-oidc_login --version`

### API Server not restarting

**Problem:** After editing kube-apiserver.yaml, the API server doesn't come back up.

**Solution:**
- Check API server logs: `sudo tail -f /var/log/pods/kube-system_kube-apiserver-*/kube-apiserver/*.log`
- Verify YAML syntax is correct (no extra spaces or tabs)
- Ensure the OIDC issuer URL is accessible from the control plane: `curl -k https://<YOUR-KEYCLOAK-ENDPOINT>/realms/<REALM>/.well-known/openid-configuration`

### "Error: oidc: verify token: failed to verify signature"

**Problem:** Token signature verification fails.

**Solution:**
- Ensure the `--oidc-issuer-url` exactly matches your Keycloak realm URL
- Verify the issuer URL is accessible from the API server (not just localhost)
- Check if Keycloak is using self-signed certificates; you may need to configure CA certificates

### Groups not working / Permission denied

**Problem:** User can authenticate but RBAC doesn't work based on groups.

**Solution:**
- Verify the groups mapper is configured correctly in Keycloak (see section 2)
- Check that `Full group path` is set to `OFF` in the mapper
- Decode your token at https://jwt.io and verify the `groups` claim is present
- Ensure the `--oidc-groups-prefix=oidc:` matches your clusterrolebinding group names

### Token expiration issues

**Problem:** Constantly having to re-authenticate.

**Solution:**
- Tokens expire based on Keycloak realm settings
- Go to Keycloak -> Realm Settings -> Tokens tab
- Increase `Access Token Lifespan` (default is usually 5 minutes)
- Consider increasing `SSO Session Idle` and `SSO Session Max` as well

### Clear cached credentials

If you need to force re-authentication:
```bash
rm -rf ~/.kube/cache/oidc-login
```

## 8. Cleanup / Rollback

To remove the OIDC configuration and revert to the original setup:

### Remove kubeconfig OIDC context

```bash
kubectl config delete-context keycloak-context
kubectl config delete-user keycloak
```

### Remove ClusterRoleBindings

```bash
kubectl delete clusterrolebinding k8s-cluster-dev
kubectl delete clusterrolebinding k8s-cluster-admin
```

### Remove ClusterRole

```bash
kubectl delete clusterrole k8s-cluster-dev
```

### Revert API Server configuration

1. SSH into the control plane
2. Edit `/etc/kubernetes/manifests/kube-apiserver.yaml`
3. Remove all lines starting with `--oidc-`
4. Wait for API server to restart

### Delete Keycloak resources

1. Delete the `kubernetes` client from Keycloak
2. Delete the groups `k8s-cluster-admin` and `k8s-cluster-dev`
3. Delete users `john.doe` and `jane.doe` (if created only for this demo)

## Next Steps

**_Furthermore, next step will be to integrate Identity Provider, manage users and group via active directory_**
