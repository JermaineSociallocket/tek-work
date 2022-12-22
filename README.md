# AWS Nginx INgress COntroller and Cert Manager

## Nginx Ingress Controller

### Setup

Visit <https://kubernetes.github.io/ingress-nginx/deploy/#quick-start> and find the charts URL and visit <https://github.com/kubernetes/ingress-nginx/releases> and get the charts release version.

Currently charts URL is `https://kubernetes.github.io/ingress-nginx` and latest charts release as of 06/12/2022 is `4.4.0`

For easier management and upgrade I design the charts as dependency chart as shown in `ingress-nginx` directory.

After creating and defining the `Chart.yaml` do `helm dependency update` to download the charts. After successful dependency update, you will get following directory structure.

```tree
ingress-nginx
├── Chart.lock
├── Chart.yaml
├── charts
│   └── ingress-nginx-4.4.0.tgz
└── values.yaml
```

The `Chart.yaml` contains charts URL and version to use, `charts` directory contains the actual pulled charts and `values.yaml` that we define to override default `values.yaml` file and to add additional information such as custom annotations.

Here in `values.yaml` we define custom annotations to spin a AWS Network Load Balancer.

### Installation

After the setup is done `cd` into the `ingress-nginx` directory, but before that do not forget to create a namespace for IC (Ingress Controller) as

```bash
kubectl create ns ingress-nginx
# Best to install in ingress-nginx namespace
```

```bash
cd ingress-nginx
```

Install the IC, do not forget to dry-run and debug

```bash
helm install ingress-nginx . -n ingress-nginx --dry-run --debug
# If everything looks good, do the actual installation
helm install ingress-nginx . -n ingress-nginx
```

Now watch for events `kubectl get events -n ingress-nginx -w` and make sure pods are running correctly and Load Balance is in `Ensured` state
If everything looks good then installation is successful.

### Changing configurations

#### ConfigMaps

To add config maps from <https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/> `config:` section of `values.yaml` file and do helm upgrade as

```bash
helm upgrade ingress-nginx . -n ingress-nginx
```

from inside the `ingress-nginx` directory.

#### Annotations

Nginx Ingress Controller provides lots of annotations to modify its behavior. <https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/>

To add annotations modify `values.yaml` file accordingly and do helm upgrade.

#### Upgrading the IC

To upgrade the IC check for new releases and change the version in `Chart.yaml` and do helm upgrade as

```bash
helm dependency update
helm upgrade ingress-nginx . -n ingress-nginx
```

Be aware of breaking changes while upgrading. Always check for change log and handle breaking changes accordingly.

## Cert Manager

All `Chart.yaml` settings are similar to IC setup including the directory structure and installation methods.
Get installation instruction for hem at <https://cert-manager.io/docs/installation/helm/>

### Install

Create a new namespace for cert-manager as

```bash
kubectl create ns cert-manager
```

Install cert manager as

```bash
cd cert-manager
helm install cert-manager . -n cert-manager --dry-run --debug
# If everything looks good do the actual installation
helm install cert-manager . -n cert-manager
```

### Upgrades

To upgrade check the release page at <https://cert-manager.io/docs/release-notes/> and change `Chart.yaml` file do

```bash
cd cert-manager
helm dependency update
helm upgrade cert-manager . -n cert-manager
```

Before upgrading check for release notes and breaking changes.

### Cluster Issuer

For to cert-manager to actually issuer certificates when requested we have to add Issuer or ClusterIssuer.

Here we create ClusterIssuer which works across the cluster where as Issuer works only in the namespace it is defined.

Here ClusterIssuer for HTTP01 Challenge is defined in file `http01-cluster-issuer.yaml` as

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: http01-cluster-issuer
  namespace: cert-manager
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: jermaine.anderson@sociallocket.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: http01-cluster-issuer-private-key
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: nginx

```

Above definition is explained in the cert-manager documentations <https://cert-manager.io/docs/configuration/acme/http01/>

### Getting certificates

To get the certificate

Point your domain or subdomain to your Ingress Controllers Load Balancers IP address. Like `A example.com 1.1.1.1`

When DNS is properly propagated (check here <https://www.whatsmydns.net/>)  

Annotate your Ingress Resource  
Add `cert-manager.io/cluster-issuer: YOUR_CLUSTER_ISSUER_NAME`  annotation to your Ingress resource and apply accordingly to trigger cert-manager to generate certificate. Check statue with `kubectl get certificate` or `kubectl get cert`

OR  
Get certificate independently as in docs <https://cert-manager.io/docs/usage/certificate/>
