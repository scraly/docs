# Installing Keycloak on OVHcloud Managed Kubernetes

## Goals of this tutorial

+ Easily secure applications and services deployed in a Kubernetes cluster
+ Be able to configure a working Keycloak deployment over the top of a Managed Kubernetes Service provided by OVHcloud
+ Configure the `OpenIdConnect` flags available for the `kube-apiserver` component of a Managed Kubernetes Service through the OVHcloud APIv6
+ Be able to use the `kubectl` command line with the Keycloak OpenIdConnect provider configured
+ Be able to use the `kubectl` command line with Keycloak and a token generated by GitHub configured as `Identity Provider`

## Before you begin

This tutorial presupposes that you already have a working OVHcloud Managed Kubernetes cluster, and some basic knowledge of how to operate it. If you want to know more on those topics, please look at the [deploying a Hello World application](../deploying-hello-world/) documentation.

## What is OIDC

OIDC stands for __[OpenID Connect](https://en.wikipedia.org/wiki/OpenID)__.  
It is an open standard and decentralized authentication protocol.  
This protocol allows verifying the user identity when a user is trying to access a protected HTTPs endpoint.

## What is Keycloak

__[Keycloak](https://www.keycloak.org/about)__ is an open source Identity and Access Management solution aimed at modern applications and services.  
It makes it easy to secure applications and services with little to no code.  
More information could be found here: [Official Keycloak documentation](https://www.keycloak.org/documentation.html)

## Dependencies and requirements

This tutorial has been written to be fully compliant with the release `v1.22` of Kubernetes.  
You may need to adapt it to be able to deploy a functional Keycloak instance in Kubernetes release prior to the `v1.22`. 

### A cert-manager to enable HTTPS connexion through keycloak

* More information could be found here: [Official cert-manager documentation](https://cert-manager.io/docs/)
* Helm chart description: [cert-manager Helm chart](https://artifacthub.io/packages/helm/cert-manager/cert-manager)
* Used Helm Chart for the deployment: `jetstack/cert-manager`

How to add the `cert-manager` Helm repository:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

Then install the `cert-manager` operator from its Helm chart:

```bash
helm install ovh-cert-lab jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.6.1 -f https://raw.githubusercontent.com/ovh/docs/develop/pages/platform/kubernetes-k8s/installing-keycloak/files/01-cert-manager/01-cert-manager-definition.yaml
```

You should have new Deployments, Service, ReplicaSet and Pods runing in your cluster:

```
$ kubectl get all -n cert-manager
NAME                                                        READY   STATUS    RESTARTS   AGE
pod/ovh-cert-lab-cert-manager-5df67445d5-h89zb              1/1     Running   0          25s
pod/ovh-cert-lab-cert-manager-cainjector-5b7bfc69b7-w78hp   1/1     Running   0          25s
pod/ovh-cert-lab-cert-manager-webhook-58585dd956-4bxgm      1/1     Running   0          25s

NAME                                        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/ovh-cert-lab-cert-manager-webhook   ClusterIP   10.3.181.202   <none>        443/TCP   46d

NAME                                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ovh-cert-lab-cert-manager              1/1     1            1           25s
deployment.apps/ovh-cert-lab-cert-manager-cainjector   1/1     1            1           25s
deployment.apps/ovh-cert-lab-cert-manager-webhook      1/1     1            1           25s

NAME                                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/ovh-cert-lab-cert-manager-5df67445d5              1         1         1       25s
replicaset.apps/ovh-cert-lab-cert-manager-cainjector-5b7bfc69b7   1         1         1       25s
replicaset.apps/ovh-cert-lab-cert-manager-webhook-58585dd956      1         1         1       25s
```

Then, we will create an `ACME ClusterIssuer` used by the `cert-manager` operator to requesting certificates from ACME servers, including from Let’s Encrypt.

During this lab, we will use the Let's Encrypt `production` environment to generate all our testing certificates.  

__/!\ Warning /!\\__  
Use of the Let's Encrypt Staging environment is not recommended and not compliant with this tutorial.  

```bash
kubectl --namespace cert-manager apply -f https://raw.githubusercontent.com/ovh/docs/develop/pages/platform/kubernetes-k8s/installing-keycloak/files/01-cert-manager/02-acme-production-issuer-http01.yaml
```

You should have a new ClusterIssuer deployed in your cluster:

```
$ kubectl get clusterissuer letsencrypt-production -o yaml -n cert-manager | kubectl neat
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    email: lab@ovhcloud.com
    preferredChain: ""
    privateKeySecretRef:
      name: acme-production-issuer-http01-account-key
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
    - http01:
        ingress:
          class: nginx
```

__/!\ Warning /!\\__  
You can use [neat](https://github.com/itaysk/kubectl-neat) kubectl plugin in order to remove useless informations in your Kubernetes manifests files.

### An Ingress Nginx to publicly expose Keycloak

* More information could be found here: [Official ingress-nginx documentation](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/)
* Helm chart description: [ingress-nginx Helm chart](https://artifacthub.io/packages/helm/ingress-nginx/ingress-nginx)
* Helm Chart used for the deployment: `ingress-nginx/ingress-nginx`

How to add `ingress-nginx` Helm repository:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Then install the `nginx-ingress` controller:

```bash
helm install ovh-ingress-lab ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace --version 4.0.6
```

You should have new resources in ìngress-nginx` namespace:

```
$ kubectl get all -n ingress-nginx
NAME                                                            READY   STATUS    RESTARTS   AGE
pod/ovh-ingress-lab-ingress-nginx-controller-6f94f9ff8c-w4fqs   1/1     Running   0          6m14s

NAME                                                         TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
service/ovh-ingress-lab-ingress-nginx-controller             LoadBalancer   10.3.166.138   135.125.83.97   80:30026/TCP,443:31963/TCP   46d
service/ovh-ingress-lab-ingress-nginx-controller-admission   ClusterIP      10.3.180.230   <none>          443/TCP                      46d

NAME                                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ovh-ingress-lab-ingress-nginx-controller   1/1     1            1           46d

NAME                                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/ovh-ingress-lab-ingress-nginx-controller-6f94f9ff8c   1         1         1       6m14s
replicaset.apps/ovh-ingress-lab-ingress-nginx-controller-8466446f66   0         0         0       46d
```

If you need to customize your `ingress-nginx` configuration, please refer to the following documentation: [ingress-nginx values](https://github.com/kubernetes/ingress-nginx/blob/main/charts/ingress-nginx/values.yaml)

__For information:__  
Installing this `ingress-nginx` controller will order a LoadBalancer provides by OVHcloud *(this load balancer will be monthly billed)*.  
For more information, please refer to the following documentation: [Using the OVHcloud Managed Kubernetes LoadBalancer](https://docs.ovh.com/gb/en/kubernetes/using-lb/)

To check if the `LoadBalancer` is up and running, execute the following CLI in a console:

```bash
kubectl --namespace ingress-nginx get services ovh-ingress-lab-ingress-nginx-controller -o wide
```

Once your LoadBalancer is up and running, get its IP address to configure your domain name zone:

```bash
kubectl -n ingress-nginx get service ovh-ingress-lab-ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}'
```

In my side, I have this new Service and the following external IP:
```
$ kubectl --namespace ingress-nginx get services ovh-ingress-lab-ingress-nginx-controller -o wide

NAME                                       TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE   SELECTOR
ovh-ingress-lab-ingress-nginx-controller   LoadBalancer   10.3.166.138   135.125.83.97   80:30026/TCP,443:31963/TCP   46d   app.kubernetes.io/component=controller,app.kubernetes.io/instance=ovh-ingress-lab,app.kubernetes.io/name=ingress-nginx

$ kubectl -n ingress-nginx get service ovh-ingress-lab-ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}'
135.125.83.97
```

If you are using the [OVHcloud Domain name product](https://www.ovhcloud.com/en-ie/domains/), you can follow this documentation to configure your DNS record to link it to the public IPv4 address associated to your LoadBalancer: [Editing an OVHcloud DNS zone](https://docs.ovh.com/ie/en/domains/web_hosting_how_to_edit_my_dns_zone/)  
If you are using an external DNS provider, I let you configure your domain and come back here for the next step of this tutorial :)

In my case, the `nslookup` command output of my domain entry shows the following information:

```bash
➱ nslookup keycloak.lab.ovh
Server:         10.15.25.129
Address:        10.15.25.129#53

Non-authoritative answer:
Name:   keycloak.lab.ovh
Address: 51.90.64.27 # where this IP address is the IP used by my Kubernetes service of `LoadBalancer` type
```

## How to configure and deploy the Codecentric Keycloak provider

* More information could be found here: [Official Keycloak documentation](https://www.keycloak.org/documentation.html)
* Helm chart description: [codecentric Keycloak Helm chart](https://artifacthub.io/packages/helm/codecentric/keycloak)
* Helm Chart used for the deployment: `codecentric/keycloak`

__For information:__  
A `PersistentVolume` will be created to host all PostgreSQL data.  
This `PersistentVolume` will be provided through the Cinder storage class which is the default storage class used by Managed Kubernetes Service at OVHcloud *(this volume will be billed)*.  
For more information, please refer to the following documentation: [Setting-up a Persistent Volume OVHcloud Managed Kubernetes](https://docs.ovh.com/ie/en/kubernetes/setting-up-a-persistent-volume/#persistent-volumes-pv-and-persistent-volume-claims-pvc)

### Keycloak installation

How to add the `codeCentric` repository:

```bash
helm repo add codecentric https://codecentric.github.io/helm-charts
helm repo update
```

Then install the `codecentric/keycloak` helm chart:

```bash
helm install ovh-keycloak-lab codecentric/keycloak -n keycloak --create-namespace --version 15.1.0 -f https://raw.githubusercontent.com/ovh/docs/develop/pages/platform/kubernetes-k8s/installing-keycloak/files/02-keycloak/01-keycloak-definition.yaml
```

Check if the Keycloak `statefulSet` is `Ready`:

```bash
kubectl -n keycloak get statefulsets.apps -o wide
```

If yes, we could edit the `files/02-keycloak/02-nginx-ingress-definition.yaml` file and configure the ingress route required to expose Keycloak on the Internet:

```bash
kubectl apply -f https://raw.githubusercontent.com/ovh/docs/develop/pages/platform/kubernetes-k8s/installing-keycloak/files/02-keycloak/02-nginx-ingress-definition.yaml
```

### Keycloak configuration

If you are reading this chapter, it indicates that your Keycloak is now up and running.  
Let's go and open the Keycloak Web console: `https://keycloak.your-domain-name.tld/`  
Log in to the `Administration Console` with the `username` and `password` configured in the `files/02-keycloak/01-keycloak-definition.yaml` file.

#### Create a REALM

A __realm__ in Keycloak is the equivalent of a tenant or a namespace. It allows creating isolated groups of applications and users.  
By default, there is a single realm in Keycloak called `Master`. It is dedicated to manage Keycloak and should not be used for your own applications.

Let's create a realm dedicated to our lab:

1) Open the Keycloak Admin console (https://keycloak.your-domain-name.tld/)
2) With your mouse, display the dropdown menu in the top-left corner where it is indicated `Master`, then click on `Add realm`
3) Fill in the form with this name: `ovh-lab-k8s-oidc-authentication`, then click on the `Create` button

#### Create a CLIENT

A __client__ in Keycloak is an entity that can request a Keycloak server to authenticate a user.

1) From the previously created realm, click on the left-hand menu `Clients` under the `Configure` category
2) Click on `Create` in the top-right corner of the table
3) Fill in the form with the following parameters:

```text
Client ID: k8s-oidc-auth
Client Protocol: openid-connect
Root URL: https: https://keycloak.your-domain-name.tld/
```

4) Then click on the `Save` button
5) Find the `Access Type` field and set its value to `confidential` to require a secret to initiate the login protocol and save the change to display the `Credentials` tab
6) Find the `Valid Redirect URIs` field and set the following value: `*`
7) Find the `Admin URL` and the `Web Origins` fields and set their values to your defined domain name if it is not already done  
In my example: `https://keycloak.lab.ovh/`. __\/!\ Be careful to use the HTTPS schema only /!\\__
8) Save your changes

#### Create a USER

1) From the previously created realm, click on the left-hand menu `Users` under the `Manage` category
2) Click on `Add user` in the top-right corner of the table
3) Fill in the form. Only the `Username` field is required, it's enough for this lab.
4) Then click on the `Save` button

The first user connection required an initial password, so let's create it:

1) Click on the `Credentials` tab
2) Fill in the `Set Password` form
3) Disable the `Temporary` flag to prevent having to update the password on first login
4) Then click on the `Set Password` button and confirm your choice

In my example, I created the following user:

```text
USERNAME: ovhcloud-keycloak-lab
PASSWORD: ovhcloud-keycloak-lab-awesome-password
```

### Kube-apiserver configuration

Some endpoints, to operate your Managed Kubernetes Service provided by OVHcloud, are available through the APIv6.  
In our case, we will configure two required flags associated to the `openIdConnect` configuration.

Go to the [OVHcloud API console](https://api.ovh.com/console/) and log in.  
Then, find the endpoints to use to update the `openIdConnect` configuration of your Managed Kubernetes Service.

#### Available openIdConnect endpoints exposed by the OVHcloud API to manage your Managed Kubernetes Service

* Get `openIdConnect` integration parameters configured on the managed `kube-apiserver`:

```text
GET /kube/service_id/openIdConnect
```

* Configure `openIdConnect` flags on the managed `kube-apiserver`:

```text
POST /kube/service_id/openIdConnect
{
"issuerUrl": "",
"clientId": ""
}
```

* Update `openIdConnect` parameters and reconfigure the managed `kube-apiserver`:

```text
PUT /kube/service_id/openIdConnect
{
"issuerUrl": "",
"clientId": ""
}
```

* Remove `openIdConnect` integration parameters from the managed `kube-apiserver`:

```text
DEL /kube/service_id/openIdConnect
```

__Where:__

* `{serviceName}` is your Openstack tenant ID
* `{kubeId}` is your cluster ID

#### openIdConnect integration and configuration

Here, we will execute the `POST` query to add a new `openIdConnect` configuration to our cluster with the following values:

```
# The serviceName can be found through the OVHcloud Manager
serviceName: ${your-openstack-tenant-id}
# The kubeId can be found through the OVHcloud Manager
kubeId: ${your-managed-kubernetes-id}
# The name of the Keycloak client previously defined: `k8s-oidc-auth` in my example
clientId: k8s-oidc-auth
# The URL to access to the previsouly defined realm: https://${your-configured-root-url}/auth/realms/${your-configured-realm-name}
# In my example, I use the `ovh-lab-k8s-oidc-authentication` realm
issuerUrl: https://keycloak.lab.ovh/auth/realms/ovh-lab-k8s-oidc-authentication
```

Then execute the query and wait a few minutes during the reconfiguration of the `kube-apiserver` of your Managed Kubernetes Service.

During the reconfiguration step, if it is not already done, you must install the `kubectl` plugin manager named [Krew](https://krew.sigs.k8s.io/).  
Then, install the [kubelogin plugin](https://github.com/int128/kubelogin) to extend the capacity of the `kubectl` command line and easily configure your environment to be able to use your Keycloak server.

```bash
kubectl krew install oidc-login
```

Once `kubelogin` installed, go back to the Keycloak web interface to get your client secret.  

You can find this information here:

- Click on the `Clients` menu in the left column
- Click on your previously created client (`k8s-oidc-auth` in my example)
- Go to the `Credentials` tab ang get your `secret` value

Then, customize the following command line with your information and execute it to be able to:

- Log in to your Keycloak provider through your browser
- Generate a token from it
- Configure your Kubectl context to access to Kubernetes APIs with the freshly generated token

Here with the information related to my example:

```bash
kubectl oidc-login setup \
--oidc-issuer-url="https://keycloak.lab.ovh/auth/realms/ovh-lab-k8s-oidc-authentication" \
--oidc-client-id="k8s-oidc-auth" \
--oidc-client-secret="f6ed439e-2dcc-4812-bbff-282cf10bef59"
```

Your favorite browser will display an authentication page to your Keycloak server.  
Here log in with the credentials defined during the `user creation` step of this tutorial.

Once, the authentication succeeded, you can close your browser tab on go back to your console where a message have been displayed.

The `kubelogin` plugin has given you some instructions to follow to finalize the configuration of your Kubectl environment.

Now, you must create a `ClusterRoleBinding` (step 3 of the kubelogin output):

```bash
# Example of output generated on my environment, please customize it with your information
kubectl create clusterrolebinding oidc-cluster-admin --clusterrole=cluster-admin --user='https://keycloak.lab.ovh/auth/realms/ovh-lab-k8s-oidc-authentication#fdb220d7-ad75-4486-9866-b8f59bd6e661'
```

You can ignore the `step 4` if you have already configured the `kube-apiserver` through the OVHcloud APIv6 console.  
Then, configure your `kubeconfig` (step 5 of the `kubelogin` output):

```bash
# Example of output generated on my environment, please customize it with your information
kubectl config set-credentials oidc \
  --exec-api-version=client.authentication.k8s.io/v1beta1 \
  --exec-command=kubectl \
  --exec-arg=oidc-login \
  --exec-arg=get-token \
  --exec-arg=--oidc-issuer-url=https://keycloak.lab.ovh/auth/realms/ovh-lab-k8s-oidc-authentication \
  --exec-arg=--oidc-client-id=k8s-oidc-auth \
  --exec-arg=--oidc-client-secret="f6ed439e-2dcc-4812-bbff-282cf10bef59"
```

And verify your cluster access (step 6 of the `kubelogin` output):

```bash
kubectl --user=oidc get nodes
```

If you could see the nodes of your Managed Kubernetes Service, congrats your Keycloak instance is up and running!

## Configure GitHub as an identity provider in Keycloak

Here, we will configure GitHub as an `Identity Provider` in our Keycloak server hosted in a Managed Kubernetes Service.

### GitHub configuration

1) Once logged in to your GitHub account, go to the [Developers settings](https://github.com/settings/developers)
2) Click on the `OAuth Apps` menu, then click on `Register a new application`
3) Fulfil the following fields:

```text
Application name: ovhcloud-keycloak-lab
Homepage URL: ${your-keycloak-url} # Im my example: https://keycloak.lab.ovh/
Application description: An awesome description like `OVHcloud keycloak lab`
Authorization callback URL: anything you want, we will come back to update this filed if few minutes!
```

4) Then click on the `Register application` button
5) Click on `Generate a new client secret` and keep the secret somewhere, we will use it in a few minutes

### Keycloak configuration

1) Go back to your Keycloak instance
2) In the `ovh-lab-k8s-oidc-authentication` realm, under the `Configure` menu, click on the `Identity Provider` link located in the left column
3) Select the `Social > Github` provider
4) Fulfil the following fields:

```text
Client ID: use the clientId given by Github. In my example: "5vabf3c16570ec9d3da7"
Client Secret: use the secret given by Github. In my example: "92dd205300b0186597395543cfc0fa3c9927aab3"
```

5) Then save your changes
6) In the `ovh-lab-k8s-oidc-authentication` realm, under the `Configure` menu, click on the `Clients` link located in the left column
7) Then, click on `Create` in the top-right corner of the table
8) Fulfill the following parameters:

```text
Client ID: k8s-github-auth
Client Protocol: openid-connect
Root URL: https: https://keycloak.your-domain-name.tld/
```

9) Find the `Access Type` field and set its value to `confidential` to require a secret to initiate login protocol and save the change to display the `Credentials` tab
10) Find the `Valid Redirect URIs` field and set the following value: `*`
11) Find the `Admin URL` and `Web Origins` fields and set their values to your defined domain name. 
In my example: `https://keycloak.lab.ovh/`. __\/!\ Be careful to use the HTTPS schema only /!\\__
12) Save your changes

### Kube-apiserver configuration

Now, it's the time to update the `openIdConnect` parameters of our `kube-apiserver` component.  
Through the [OVHcloud APIv6 console](https://api.ovh.com/console/#/cloud/project/%7BserviceName%7D/kube/%7BkubeId%7D/openIdConnect#PUT), find the endpoint to use to update the OIDC configuration of your Managed Kubernetes Service.

We will use the `PUT` query to update the `clientId` parameter with the new one created to use GitHub as an `Indentity Provider` for the K8s cluster.  

In my example:

```text
{
    issuerUrl: https://keycloak.lab.ovh/auth/realms/ovh-lab-k8s-oidc-authentication
    clientId: "k8s-github-auth"
}
```

### Finalize the configuration

Before to configure the cluster to get an identity from GitHub, we must update the `Authorization callback URL` previously configured in the GitHub OAuth application.

1) Go to the GitHub provider configured in Keycloak under the `Configure > Identity Provider` menu
2) Find the `github` provider then click on the `Edit` button
3) Copy the content of the `Redirect URI`
4) Go back to the previously created `ovhcloud-keycloak-lab` OAuth application in GitHub
5) Paste the `Redirect URI` content in the `Authorization callback URL`
6) Then update the application

### kubelogin configuration

As already done in this tutorial, we will use the [kubelogin plugin](https://github.com/int128/kubelogin) to configure the Managed Kubernetes Service to use our OIDC provider.

With the following information found in the Keycloak web interface, customize your command line to execute the `kubelogin` plugin through `kubectl`:

* The OIDC issuer URL
* The OIDC client ID associated to the `k8s-github-auth` client created in Keycloak
* The OIDC client secret associated to the `k8s-github-auth` client created in Keycloak

Here with the information related to my example:

```bash
kubectl oidc-login setup \
--oidc-issuer-url="https://keycloak.lab.ovh/auth/realms/ovh-lab-k8s-oidc-authentication" \
--oidc-client-id="k8s-github-auth" \
--oidc-client-secret="0de2a792-38c1-488f-8f54-f2afdb97f5d6"
```

Your browser will display an authentication page to your Keycloak server.  
Here log in using the `Github` button available under the form `Sign in to your account`.  

You will be redirected to the GitHub website to authorize your GitHub account to log in to the OAuth application created earlier.  
Once, authenticated, you will be redirected to Keycloak to update your account information.  

Fulfil the required information then click on the `Submit` button.  
If your identity has been provided by GitHub, you will be redirected to a page displaying: `Authenticated`

To finalize the configuration executed the instructions shown by `kubelogin`:

1) Cluster role binding creation

```bash
# In my example
kubectl create clusterrolebinding oidc-cluster-admin --clusterrole=cluster-admin --user='https://keycloak.lab.ovh/auth/realms/ovh-lab-k8s-oidc-authentication#1851f838-f493-410e-aafe-d20291b7ab9b'
```

If a `clusterrolebinding` already exists with the same name, you could delete it before to create the new one:

```bash
kubectl delete clusterrolebinding oidc-cluster-admin
```

2) Creation of the required credentials

```bash
# In my example
kubectl config set-credentials oidc \
--exec-api-version=client.authentication.k8s.io/v1beta1 \
--exec-command=kubectl \
--exec-arg=oidc-login \
--exec-arg=get-token \
--exec-arg=--oidc-issuer-url=https://keycloak.lab.ovh/auth/realms/ovh-lab-k8s-oidc-authentication \
--exec-arg=--oidc-client-id=k8s-github-auth \
--exec-arg=--oidc-client-secret=0de2a792-38c1-488f-8f54-f2afdb97f5d6
```

3) Ignore the `step 4` because we have already configured the `kube-apiserver` through the OVHcloud APIv6 console
4) Test your configuration

```text
kubectl --user=oidc get nodes
```

If you can see your Kubernetes nodes list, your GitHub identity provider is properly working.  
Enjoy!

## Upgrade the keycloak deployment if needed

```bash
helm upgrade ovhcloud-keycloak-lab codecentric/keycloak -n keycloak -f files/02-keycloak/01-keycloak-definition.yaml
```

## Rollout restarts the Keycloak statefulsets if needed

```bash
kubectl -n keycloak rollout restart statefulset ovh-keycloak-lab
```

## Various troubleshooting

- If the `cert-manager` namespace is stuck in deleting state, see the following documentation: [namespace-stuck-in-terminating-state](https://cert-manager.io/docs/installation/helm/#namespace-stuck-in-terminating-state)
- If the `POST` query executed to create your OIDC `kube-apiserver` flags returns you a `400` error, please check if the used `REALM` name is properly configured in the given `issuerUrl` parameter.  

Example of results returned by the OVHcloud API:

```bash
Bad Request (400)
{ "class": "Client::BadRequest", "message": "[InvalidDataError] 400: Failed to reach issuer URL for validation
```

## How to clean up your Managed Kubernetes Service

To clean up all existing resources related to the Keycloak Helm chart, execute the following command lines:

```bash
# Delete the Keycloak statefulsets
helm -n keycloak uninstall ovh-keycloak-lab
# Delete the data related to the Keycloak deployment
kubectl -n keycloak delete pvc data-ovh-keycloak-lab-postgresql-0
# Delete the Keycloak namespace
kubectl delete namespaces keycloak
```

To clean up all existing resources related to the `ingress-nginx` Helm chart, execute the following command lines:

```bash
# Delete the ingress-nginx deployments
helm -n ingress-nginx uninstall ovh-ingress-lab 
# Delete the ingress-nginx namespace
kubectl delete namespaces ingress-nginx
```

To clean up all existing resources related to the `cert-manager` Helm chart, execute the following command lines:

```bash
# Delete the cert-manager deployments
helm -n cert-manager uninstall ovh-cert-lab 
# Delete the cert-manager namespace
kubectl delete namespaces cert-manager
```

# Useful resources

https://www.keycloak.org/documentation  
https://artifacthub.io/packages/helm/codecentric/keycloak  
https://cert-manager.io/docs/usage/ingress/  
https://www.keycloak.org/getting-started/getting-started-kube  
https://kubernetes.io/docs/reference/access-authn-authz/authentication/#option-1-oidc-authenticator  