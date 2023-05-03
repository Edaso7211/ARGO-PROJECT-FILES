# ARGO-PROJECT-FILES
# Ready Pay Infrastructure Applications

This repo contains the yaml files for ArgoCD as well as the app definitions for things that ArgoCD deploys. When interacting with Kubernetes clusters, you often need to target a specific context/cluster and namespace. To make this easier, it's highly recommended to make use of [`kubens` and `kubectx`](https://github.com/ahmetb/kubectx/).

# Applications

The following applications are deployed in the cluster. What each application is used for, any necessary manual set up steps, and deployment mechanism are documented below.

## ArgoCD

[ArgoCD](https://argo-cd.readthedocs.io/en/stable/) is used for deploying applications using a git-ops workflow. Argo deploys applications by looking at a given git repo, and then deploying any Kubernetes configuration files it can find. The exact repos and paths within those repos can be configured in a number of ways, but in these clusters they are configured in a declarative manner via [`Application`](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#applications) and [`ApplicationSet`](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/) resources.

Once Argo is running, it is used for deploying all other applications listed below. The `Application` and `ApplicationSet` resources that Argo deploys can be found in the `kube/argocd-apps/` directory.

### Manual Steps

Part of Argo's purpose is to deploy applications that are defined in private git repos. To authorize Argo to read from private repos, you need to provide a method of authenticating. Argo can be configured to authenticate in a number of ways to [private repos](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/), but the example below shows a simple SSH key attached to a git repo as a deploy key.

To start, create a key (leave the password blank when prompted):


ssh-keygen -t rsa -b 4096 -f ~/.ssh/my_argo_key -C ''


Copy the public key:


cat ~/.ssh/my_argo_key.pub | pbcopy


And navigate to the git repo you want Argo to be able to read. Under `Settings` -> `Deploy keys`, add a new deploy key. Name it something reasonable and paste the *public* key that you copied above.

Now we need to provide the private key to Argo. To do this, open up the Argo UI (`kubectl port-forward -n argocd svc/argocd-server 8443:443`). Open your browser to [https://localhost:8443](https://localhost:8443), and follow [these instructions](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/#ssh-private-key-credential) to add the repository (you can skip filling out the `Project` field). Be sure to use an SSH-style git URL (`git@github.com:ORG/REPO.git`) and paste the *private* key.

Repeat that for each repo Argo needs access to.

The above approach works if you have a small number of repos that Argo needs access to. If Argo needs access to many repos, it would be worthwhile to use [credential templates](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/#credential-templates), or create a [GitHub app](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/#github-app-credential) so Argo can use a single key to access any repo in the org.

### Deploying

Because Argo deploys all the other applications, Argo itself must be deployed manually. This can be done via:


kubectl apply -k kube/argocd/overlays/<overlayName>


## Cert Manager

[Cert manager](https://cert-manager.io) is used to create self signed certificates for Kubernetes admission webhooks (initially, just the Kong validating admission webhook). Cert manager could be, but is not currently, used for generating SSL certs via Let's Encrypt for the primary entrypoint to the cluster (Kong).

### Manual Steps

Cert manager does not require any manual set up.

### Deploying

Cert manager is deployed via Argo. Apply the application to the cluster and Argo will do the rest:


kubectl apply -k kube/argocd-apps/<overlayName>


## Datadog

[Datadog](https://www.datadoghq.com) is used for monitoring all aspects of the environment from the infrastructure down to the individual application. Applications can also be configured to send custom metrics to datadog.

### Manual Steps

To authorize the Datadog agent in the cluster with the Datadog APIs, an api key and app key must be generated. Login to Datadog and go to the [Organization Settings](https://app.datadoghq.com/organization-settings/api-keys) page. Under `API Keys`, create a new key. Save this value. Now under `App Keys`, create a new key. Save this value.

Now create a secret with those two keys.


kubectl create secret generic -n default --from-literal=api-key=<api-key-value> --from-literal=app-key=<app-key-value> datadog


Now generate a random 32 character string for encrypting communication between the datadog agents and the cluster agent.


openssl rand 32 | shasum -a 256 | head -c 32


Now create a secret with that value.


kubectl create secret generic -n default --from-literal=token=<value> datadog-cluster-agent


Datadog can now be deployed. If it is already deployed, perform a restart:


kubectl rollout restart -n default deploy datadog-cluster-agent
kubectl rollout restart -n default ds datadog


Lastly, Datadog supports monitoring GCP infrastructure directly. This requires logging in to the Datadog UI and pasting a GCP service account key with the appropriate permissions. The terraform code in this repo already creates this service account with the necessary permissions, you just need to login and create a key.

1. Go to the [GCP Service Accounts](https://console.cloud.google.com/iam-admin/serviceaccounts?project=ready-pay-test) page and click on the [datadog service account](https://console.cloud.google.com/iam-admin/serviceaccounts/details/105436289701423848114?project=ready-pay-test)
1. Under `Keys`, `Add key` -> `Create new key`. Download it as JSON
    * If there were any other keys attached to the service account, delete them
1. Now go to the [integrations](https://app.datadoghq.com/integrations) page in Datadog
1. Install or configure the `Google Cloud Platform` integration
1. Upload the private key file you downloaded
1. Click Save/Update

### Deploying

Datadog is deployed via Argo. Apply the application to the cluster and Argo will do the rest:


kubectl apply -k kube/argocd-apps/<overlayName>


## Kong

[Kong](https://docs.konghq.com/kubernetes-ingress-controller/latest/) (or more specifically, the Kong Kubernetes Ingress Controller) is the primary API gateway and ingress controller for the cluster. When deploying Kong, a load balancer will automatically be created with a public IP attached. This load balancer and IP address are the single entry point to the entire cluster. When Kong receives a request, it will perform SSL termination, and forward the request to the appropriate backend service based on the requested hostname and path. Kong is configured through the use of `Ingress` resources. These resources specify which hostname and path should be forwarded to which `Service`, and ultimately `Pod`.

Because Kong performs TLS termination, SSL certificates are required to be placed into the Kubernetes cluster as `Secret` resources.

Kong is capable of responding to any number of domain names, as long as the DNS is set up. When creating DNS records to support new domains, they should be created as A records and point to Kong's IP address. Kong's IP address can be found by examining the kong-proxy service in the cluster.


kubectl get svc -n kong kong-proxy


The IP will be shown under the `EXTERNAL-IP` column.

### Manual Steps

Kong doesn't require any manual steps to be up and running, but you will need to create SSL certificates and place them in the cluster as `Secret` resources. The name of the secret depends on your `Ingress` resources. Take the following `Ingress` resource as an example:


apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-api
  namespace: default
  annotations:
    kubernetes.io/ingress.class: kong
    konghq.com/protocols: http, https
spec:
  tls:
    - hosts:
        - myapp.com
      secretName: myapp.com
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /api
            pathType: ImplementationSpecific
            backend:
              service:
                name: my-api
                port:
                  number: 80


With this resource, you will need a `Secret` named `myapp.com` that contains the SSL certificate for the `myapp.com` domain. Create a secret resource like this:


apiVersion: v1
kind: Secret
metadata:
  name: myapp.com
  namespace: default
type: kubernetes.io/tls
data:
  tls.key: <based64EncodedPrivateKey>
  tls.crt: <based64EncodedPublicCertAndChain>


Then `kubectl apply -f <file>` to the cluster. A couple things to note about this:

* The secret must be defined in the same namespace as the `Ingress` resource that references it (future versions of Kong improve on this)
* If multiple `Ingress` resources reference the same secret from different namespaces, the secret must be duplicated in each namespace (future versions of Kong improve on this)
* The `tls.crt` must be a certificate chain. When purchasing a cert from a certificate provider, they will usually include an intermediate certificate. The `tls.crt` must include the leaf certificate and the intermediate (in that order).

### Deploying

Kong is deployed via Argo. Apply the application to the cluster and Argo will do the rest:


kubectl apply -k kube/argocd-apps/<overlayName>


One thing that is unique about Kong, is that even though it's deployed by Argo, Argo is looking at this repo and applying the kustomize overlays that it finds in `kube/kong/overlays/`. So upgrading Kong or modifying its configuration is not done in `kube/argocd-apps/kong.yaml` like most other apps. It is done by modifying files in `kube/kong/`.

The reason Kong is deployed like this (kustomize instead of helm) is the lack of support for certain configuration options in the Kong helm chart. Some examples of the support that is missing are:

* Cannot deploy custom resources (like Certificates and Issuers). This would require us to deploy a kong helm chart, and kong kustomize overlays.
* Cannot set custom annotations on the validation webhook service. This removes our ability to dynamically configure datadog metrics for the certificate.
* Creation of a metrics service. This removes our ability to pull metrics from the kong metrics endpoints via datadog

## Velero

[Velero](https://velero.io) is used to take automatic backups of all resources in the Kubernetes cluster, as well as take snapshots of all volumes in use in the Kubernetes cluster. Resource backups are stored in a GCS bucket in each project. The bucket is named `<project>-velero`. Volume snapshots are taken using standard GCP APIs. Velero is configured to take daily backups retained for 30 days, and monthly backups retained for 1 year.

The Velero CLI can be used for interacting with the backups, taking more frequent manual backups, or performing a restore.

### Manual Steps

In order to take disk snapshots, upload files to a GCS bucket, and download files from a GCS bucket, Velero needs permissions via a GCP service account. The terraform in this repo already configure a Velero service account with the necessary permissions, but a new key will be needed. To download a key for the Velero service account, go to the [GCP Service Accounts](https://console.cloud.google.com/iam-admin/serviceaccounts?project=ready-pay-test) page. Choose the [`velero` service account](https://console.cloud.google.com/iam-admin/serviceaccounts/details/113723796496189560026/keys?project=ready-pay-test). Under the `Keys` tab, `Add Key` -> `Create new key` (key type `JSON`) to download the key (if there were any other keys attached to the service account, delete them).

Now that key needs to be placed into a secret called `cloud-credentials` in the `velero` namespace.


kubectl create secret generic cloud-credentials -n velero --from-file=cloud=/path/to/svcacct/key.json


If Velero is not yet running, it can now be deployed. If it is running, restart Velero:


kubectl rollout restart -n velero deploy velero


If there are any older keys attached to the service account, delete them.

### Deploying

Velero is deployed via Argo. Apply the application to the cluster and Argo will do the rest:


kubectl apply -k kube/argocd-apps/<overlayName>
