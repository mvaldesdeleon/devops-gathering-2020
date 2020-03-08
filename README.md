# EKS Workshop @ DevOps Gathering 2020

## Agenda

* [Introduction](https://eksworkshop.com/010_introduction/)
* [Start the workshop...](https://eksworkshop.com/020_prerequisites/)
* [Launch using eksctl](https://eksworkshop.com/030_eksctl/)
* [Deploy the Kubernetes Dashboard](https://eksworkshop.com/beginner/040_dashboard/)
* [Helm](https://eksworkshop.com/beginner/060_helm/)
* [Autoscaling our Applications and Clusters](https://eksworkshop.com/beginner/080_scaling/)
* [Intro to RBAC](https://eksworkshop.com/beginner/090_rbac/)
* [IAM Roles for Service Accounts](https://eksworkshop.com/beginner/110_irsa/)
* [GitOps with Weave Flux](https://eksworkshop.com/intermediate/260_weave_flux/)
* [Tracing with X-Ray](https://eksworkshop.com/intermediate/245_x-ray/)

## AWS Workshop Portal

Log into our [AWS Workshop Portal](https://dashboard.eventengine.run/login) with the participant hashcode provided, and set your name. You can get information on how to access the AWS Console by clicking on the "AWS Console" button.

## Deploy the Kubernetes Dashboard

### Deploy the Kubernetes Metrics Server

Download and deploy the `metrics-server`:

```sh
DOWNLOAD_URL=$(curl -Ls "https://api.github.com/repos/kubernetes-sigs/metrics-server/releases/latest" | jq -r .tarball_url)
DOWNLOAD_VERSION=$(grep -o '[^/v]*$' <<< $DOWNLOAD_URL)
cd ~/environment
curl -Ls $DOWNLOAD_URL -o metrics-server-$DOWNLOAD_VERSION.tar.gz
mkdir metrics-server-$DOWNLOAD_VERSION
tar -xzf metrics-server-$DOWNLOAD_VERSION.tar.gz --directory metrics-server-$DOWNLOAD_VERSION --strip-components 1
kubectl apply -f metrics-server-$DOWNLOAD_VERSION/deploy/1.8+/
```

Verify the deployment:

```sh
kubectl get deployment metrics-server -n kube-system
```

Expected output:

```sh
NAME             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
metrics-server   1         1         1            1           56m
```

### Deploy the Dashboard

Use the following command to deploy the Kubernetes dashboard:

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
```

Output:

```sh
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
```

### Create an eks-admin Service Account and Cluster Role Binding

By default, the Kubernetes dashboard user has limited permissions. In this section, you create an eks-admin service account and cluster role binding that you can use to securely connect to the dashboard with admin-level permissions. For more information, see [Managing Service Accounts](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) in the Kubernetes documentation.

Create a file called `eks-admin-service-account.yaml` with the text below. This manifest defines a service account and cluster role binding called `eks-admin`.

```sh
cat <<EoF > ~/environment/eks-admin-service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eks-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: eks-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: eks-admin
  namespace: kube-system
EoF
```

Apply the service account and cluster role binding to your cluster.

```sh
kubectl apply -f ~/environment/eks-admin-service-account.yaml
```

Output:

```sh
serviceaccount "eks-admin" created
clusterrolebinding.rbac.authorization.k8s.io "eks-admin" created
```

### Connect to the Dashboard

Now that the Kubernetes dashboard is deployed to your cluster, and you have an administrator service account that you can use to view and control your cluster, you can connect to the dashboard with that service account.

Retrieve an authentication token for the eks-admin service account. Copy the `<authentication_token>` value from the output. You use this token to connect to the dashboard.

```sh
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
```

Output:

```sh
Name:         eks-admin-token-b5zv4
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=eks-admin
              kubernetes.io/service-account.uid=bcfe66ac-39be-11e8-97e8-026dce96b6e8

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      <authentication_token>
```

**Tip: Keep this authentication token handy, as the Dashboard session will expire after a while and you will need to log in again.**

Since this is deployed to our private cluster, we need to access it via a proxy. Kube-proxy is available to proxy our requests to the dashboard service. In your workspace, run the following command:

```sh
kubectl proxy --port=8080 --address='0.0.0.0' --disable-filter=true &
```

This will start the proxy, listen on port 8080, listen on all interfaces, and will disable the filtering of non-localhost requests.

This command will continue to run in the background of the current terminal’s session.

**Warning: We are disabling request filtering, a security feature that guards against XSRF attacks. This isn’t recommended for a production environment, but is useful for our dev environment.**

Now we can access the Kubernetes Dashboard. In your Cloud9 environment, click **Tools / Preview / Preview Running Application**. Scroll to the end of the URL and append:

```
api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#!/login
```

Select "Token" and paste the authentication token retrieved previously.

If you want to see the dashboard in a full tab, click the Pop Out button, like below:

![Screenshot of the "Popout" button](https://eksworkshop.com/images/popout.png)

### References

* [Amazon EKS User Guide - Tutorial: Deploy the Kubernetes Web UI (Dashboard)](https://docs.aws.amazon.com/eks/latest/userguide/dashboard-tutorial.html)
* [Kubernetes Documentation - Web UI (Dashboard)](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)

## Helm

Note that the output of `helm install` and `helm status` changed in Helm 3.x, and no longer lists the resources created.

Also keep in mind that `helm` respects namespaces. You can specify the namespace with `-n <namespace>` or use `-a` to specify all namespaces.

## Autoscaling our Applications and Clusters

The [Deploy the Metrics Server](https://eksworkshop.com/beginner/080_scaling/deploy_hpa/#deploy-the-metrics-server) section should be skipped, as the Metrics Server was installed while deploying the Dashboard.

You can either remove the last commands from the Cleanup step or leave them, as they will fail silently:

```sh
helm -n metrics uninstall metrics-server

kubectl delete ns metrics
```

### References
* [Kubernetes Documentation - Horizontal Pod Autoscaler Walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)

## Intro to RBAC

Make sure you change your working directory to `~/environment` before starting this module.

## IAM Roles for Service Accounts

### References
* [Kubernetes Documentation - Managing Service Accounts](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)

## GitOps with Weave Flux

### GitHub Setup

**Tip: Keep the Personal Access Token handy. Later in this module you will need to push changes back to GitHub, and this Personal Access Token can be used instead of your password. This will also bypass 2FA, if you have enabled it.**

### Install Weave Flux

Now we will use Helm to install Weave Flux into our cluster and enable it to interact with our Kubernetes configuration GitHub repo.

First, install the Flux Custom Resource Definition:

```sh
kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/master/deploy/crds.yaml
```

In the following steps, your Git user name will be required. Without this information, the resulting pipeline will not function as expected. Set this as an environment variable to reuse in the next commands. Update this to your git username.

```sh
YOURUSER=yourgitusername
```

First, create the flux Kubernetes namespace

```sh
kubectl create namespace flux
```

Next, add the Flux chart repository to Helm and install Flux, along with the Helm Operator. Update the Git URL below to match your Kubernetes configuration manifest repository, if you used a different name than the one suggested.

```sh
helm repo add fluxcd https://charts.fluxcd.io

helm upgrade -i flux fluxcd/flux \
--set git.url=git@github.com:${YOURUSER}/k8s-config \
--namespace flux

helm upgrade -i helm-operator fluxcd/helm-operator \
--set helm.versions=v3 \
--set git.ssh.secretName=flux-git-deploy \
--namespace flux
```

Watch the install and confirm everything starts. There should be 3 pods.

```sh
kubectl get pods -n flux
```

Install fluxctl in order to get the SSH key to allow GitHub write access. This allows Flux to keep the configuration in GitHub in sync with the configuration deployed in the cluster.

```sh
sudo wget -O /usr/local/bin/fluxctl https://github.com/fluxcd/flux/releases/download/1.14.1/fluxctl_linux_amd64
sudo chmod 755 /usr/local/bin/fluxctl

fluxctl version
fluxctl identity --k8s-fwd-ns flux
```

Copy the provided key and add that as a deploy key in the GitHub repository.

* In GitHub, select your `k8s-config` GitHub repo. Go to **Settings** and click **Deploy Keys**.
* Click on **Add Deploy Key**
  * Name: **Flux Deploy Key**
  * Paste the key output from fluxctl
  * Click **Allow Write Access**. This allows Flux to keep the repo in sync with the real state of the cluster
  * Click **Add Key**

Now Flux is configured and should be ready to pull configuration.

### Deploy from Manifests

After creating the workloads manifest file, `k8s-config/workloads/eks-example-dep.yaml`, in addition to replacing `YOURACCOUNT` and `YOURTAG` in the image URI, the current region (`us-east-1`) will also need to be updated to the workshop region, `eu-central-1`. If you fetch the URL directly from the Amazon ECR Console, then it will already have the correct value.

Also, when making a change to the eks-example source code, there is no need to use `vi` and make the changes from the termina. You can navigate to the file within Cloud9 and edit it using the integrated editor.

### Deploy from Helm

After creating the Helm release manifest file, `k8s-config/releases/nginx.yaml`, the `apiVersion` will need to be updated to `helm.fluxcd.io/v1`.

### Cleanup

To properly delete the kubernetes resources created, use the following commands instead:

```sh
helm uninstall helm-operator --namespace flux
helm uninstall flux --namespace flux
kubectl delete namespace flux 
kubectl delete crd helmreleases.helm.fluxcd.io

helm uninstall mywebserver -n nginx
kubectl delete namespace nginx
kubectl delete svc eks-example -n eks-example
kubectl delete deployment eks-example -n eks-example
kubectl delete namespace eks-example
```
