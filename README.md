# kubernetes-keycloak-sso

Simple demonstration on how to manage access to kubernetes cluster using keycloak SSO.

The demonstration will implement access for Devs with limited privileges and Admin with cluster admin privileges.

The advantage will reduce the overhead of manually granting access to the cluster and when integrated properly, the same user group and realm can be used to manage access across multiple application e.g ArgoCD, Grafana, etc.


## Requirements

1. Kubernetes cluster
2. Keycloak 
3. kubectl-oidc-login plugin (kubelogin)
4. kubectl 

For the demonstration, I will be using:

- kubernetes (v1.31.1)
- keycloak (26.4.6)
- kubectl (v1.34.1)

> I will not cover how to install and setup kubernetes and keycloak in this demo.
You will need ssh access to the controlplane to complete this demo.

For kubectl-oidc-login, run `brew install kubelogin` on MacOS.

## 1. Setup keycloak user and group

In this section, I will create two users and two groups to represent Devs and Admins.

Be sure to create a new Realm in keycloak for the demo. If you do not know how to do so, please see keycloak documentation

### Create two users `(john.doe & jane.doe)`

1. After login, click on `Users` -> `Add user`
2. Fill out `username` field and `Email`
3. Click on create and navigate to `Credentials` tab 
4. Set desired password and save. Temporary can be turned off for this demo 
5. Repeat the same steps for jane.doe 

### Create two groups `k8s-cluster-admin & k8s-cluster-dev`

1. Click on `Groups` -> `Create group` button
2. Enter group name `k8s-cluster-admin` and create 
3. Click on the newly created group -> `Members` tab
4. Add member `john.doe` 
5. Repeat step 1-4 for jane.doe

## 2. Create and config Client in Keycloak

1. Click on Clients -> `Create client`
2. Set Client ID to `kubernetes` and Name to `kubernetes`
3. Turn on the `Client authentication` 
4. Select `Standard flow` and `Service accounts roles`
5. On the next page, set `Valid redirect URIs` to `http://localhost:18000/*`and `http://localhost:8000/*`
6. Set `Web origins` to `+`
7. In the `Credentials` tab, copy the client secret and save it somewhere. We will need it later 

## 3. Config Kube-ApiServer

1. ssh into controlplane
2. Edit kube-apisever with editor of choice `vi /etc/kubernetes/manifests/kube-apiserver.yaml`
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

> Wait for ApiServer to be available again before proceeding.

## 4. Create Clusterroles

The clusterrole represent the permission the user / group will assume after authentication. 

### Create `k8s-cluster-dev` clusterrole 

```bash
kubectl create clusterrole k8s-cluster-dev --verb=create,get,list --resource=pods,deploy 
```

> I will not create a new clusterrole for k8s-cluster-admin because I will bind it to kubernetes cluster admin role. 

### Create clusterrolebindings.

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
      - --oidc-issuer-url=https://<YOUR-KEYCLOAK-ENDPOINT-HERE/realms/<REALM-HERE>
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

> This open up the browser. Login using `john.doe` and your password to keycloak. You should be able to view cluster pods resource 

- Invalid the current user login by going to keycloak -> clients -> kubernetes -> Session. Click on the 3 dots on and sign out `john.doe` or use `rm -r ~/.kube/cache/oidc-login`

- Try to again `kubectl get pods -A` and login with the user `jane.doe`. The user should be able to view pods

- Execute `kubectl get secrets -A`. You will get an error like `Error from server (Forbidden): secrets is forbidden: User "oidc:jane.doe" cannot list resource "secrets" in API group "" in the namespace "default"`

This way, you can manage access to the cluster via Keycloak.
