# Andr Cluster

## TODO
- Terraform to create all

## Ensure Correct Cluster
Ensure you're the correct cluster using:
```
kubectl config current-context
```

To change the cluster you're pointing to:
```
kubectl config get-contexts
```

Set one of the clusters, such as `gke_gcp-project_europe-west2-c_cluster-name`:
```
kubectl config use-context gke_gcp-project_europe-west2-c_cluster-name
```

## Install Helm
Helm is used to install tooling such as `cert-manager`.
```
kubectl create -f kubernetes/manifests/auth/tiller-rbac-config.yaml
```
```
helm init --service-account tiller
```

## Create Ingress Controller
The ingress controller is the entry point into the Kubernetes cluster over port
80 (HTTP) and 443 (HTTPS), the latter using the certificate created by the
`cert-manager` which is installed later. To use the certificate you need the
annotation:
```
certmanager.k8s.io/cluster-issuer: letsencrypt-issuer
```

__ACTION:__ Initially (for a brand new cluster) you will need to
create the ingress controller *without* the references to the certificates. This
is because you need to create the ingress controller, to get the IP address, to
create the DNS A-record (see later), to get the certificate.

So comment out the lines such as:

```
# annotations:
#  certmanager.k8s.io/cluster-issuer: letsencrypt-issuer
```
...and:

```
# tls:
# - secretName: andr-tls
```
...then run:
```
kubectl apply -f kubernetes/manifests/ingress/ingress.yaml
```
Obtain the external IP address of the ingress (this will some time):
```
kubectl get ing andr-ingress
```

When the external IP is available, visit
GCP Cloud DNS and update `andr.io.` A record with the new IP (NB: you should probably
only do this when you are sure that you want to receive traffic on this cluster).

Visiting `andr.io` will, for some time, either point to the previous
address or return a Google 404. It will take some time to update the
load-balancers and is normal.

__NOTE:__ The cert-manager won't be able to create the certificate until you're
actually accepting traffic to the cluster so if you're migrating from an old
cluster you may need to create the `andr-tls` secret manually or accept
an amount of outage as you update DNS records and receive a new certificate on
your new cluster.

## Install and Configure Certificate Manager
`cert-manager` is used to manage certificates.
```
helm install --name cert-manager --namespace kube-system stable/cert-manager
```
```
kubectl apply -f kubernetes/manifests/ingress/acme-issuer.yaml
```
```
kubectl apply -f kubernetes/manifests/ingress/certificate.yaml
```

__ACTION:__ At this point you should set the ingress controller back to using TLS so
uncomment the lines in `kubernetes/manifests/ingress/ingress.yaml` and re-run the `apply`
command.

## Create Environment-specific Config
There is currently no environment-specific config, although they would live at:
_kubernetes/manifests/environment/(production|stage)/configmap-environment.yaml_

## Create Config for all Environments
There is currently no all-environment-specific secrets, although they would live at:
_kubernetes/manifests/environment/all/secrets-environment.yaml_

## Create Environment-specific Secrets
There are currently no environment-specific secrets, although they would live at:
_kubernetes/manifests/environment/(production|stage)/secrets-environment.yaml_

## Create Secrets For All Environments
Currently we only have UAS secrets:
```
kubectl apply -f kubernetes/manifests/environment/all/secrets-uas.yaml
```

## Create Components
Config specific to the component, for example _wiring_ such as other services
like `API_HOST: "foo:50051"`.

__NOTE:__ services should not use environment naming such as `foo-stage` but
naming such as `foo`. This is because an environment is defined by the cluster
the manifest are applied to rather than the service names.

The following commands should be run for all component_names in components:
```
kubectl apply -f kubernetes/manifests/components/(component_name)/configmap-(component_name).yaml
```
Create Kubernetes deployment which maps the container image to container ports,
configmaps, and health-checks etc.
```
kubectl apply -f kubernetes/manifests/components/(component_name)/deployment-(component_name).yaml
```
Create Horizontal Pod Autoscaler which encapsulates the autoscaling rules for
this service.
```
kubectl apply -f kubernetes/manifests/components/(component_name)/hpa-(component_name).yaml
```
Create Kubernetes service which exposes the component pod to traffic. Services
should not have external access directly but instead run through an ingress
controller.
```
kubectl apply -f kubernetes/manifests/components/(component_name)/service-(component_name).yaml
```
