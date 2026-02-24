# Initial deployment

## Setup using `just` (recommneded)
```bash
just install_all
```
### Going step by step
```bash
just setup_dependencies
just install_argocd
just install_apps
```

### Retrying an individual dependency
- There's private just recipes for each dependency (they don't show up in `just -l`, but you can still run them)
- You can check the [justfile](./justfile) for all the steps, but they should all follow the naming pattern `setup_{dep name}` and you can view the ordering as the order that they're called in `setup_dependencies`

Example:
```bash
just setup_cluster-config # Deploy some very basic manifests to configure the cluster
just setup_external-secrets # Deploy external-secrets (https://external-secrets.io)
just setup_cert-manager # Deploy cert-manager (https://cert-manager.io)
just setup_ingress # Deploy the cloudflare tunnel ingress controller (https://tunnel.strrl.dev)
```

## Manual setup
```bash
## Install various dependencies of argo first
# Basic cluster configuration
kubectl apply -k cluster-config -n kube-system
# External-secrets (https://external-secrets.io)
helm repo add external-secrets https://charts.external-secrets.io
helm install \
  external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets \
  --wait
# Create the infisical universal auth secret (https://infisical.com/docs/documentation/platform/identities/universal-auth)
kubectl create secret generic infisical-secret --from-literal=clientId='INSERT_CLIENT_ID_HERE' --from-literal=clientSecret='INSERT_CLIENT_SECRET_HERE' -n external-secrets
# Create the external-secret secretstore
kubectl apply -k external-secrets -n external-secrets
# Cert-manager (https://cert-manager.io/)
kubectl apply -k cert-manager -n cert-manager
# Install cloudflare tunnel ingress (https://tunnel.strrl.dev/)
kubectl apply -k cloudflare-ingress -n cloudflare-tunnel-ingress-controller
helm repo add strrl.dev https://helm.strrl.dev
helm install \
  cloudflare-tunnel-ingress-controller \
  strrl.dev/cloudflare-tunnel-ingress-controller \
  --namespace cloudflare-tunnel-ingress-controller \
  --wait \
  --values ./cloudflare-ingress/values.yaml

## Install ArgoCD
kubectl apply --server-side --force-conflicts -k argocd -n argocd

## Install the ArgoCD project that deploys all the apps
kubectl apply -k argoproj -n argocd
```

# Apps used/deployed
## External-secrets (https://external-secrets.io)
- Used to pull secrets from https://infisical.com/ (an open-source secret manager)
- Can automatically sync secrets from a secret manager and create `Secret` objects that other projects can reference
- Can also generate passwords, and can even push `Secret`s to the secret manager
- Can be setup with CloudNativePG to auto-rotate database passwords (https://cloudnative-pg.io/docs/1.28/cncf-projects/external-secrets)

## Cert-manager (https://cert-manager.io)
- Used to get [Let's Encrypt](https://letsencrypt.org/) certificates
- Configured to use DNS challenges with Cloudflare

## Cloudflare Tunnel Ingress Controller (https://tunnel.strrl.dev/)
- Used to automatically create Cloudflare Tunnels which then can forward traffic to `Ingress` objects which then route to apps
- Can be used to make some services publically accessible under your own domain without having your whole cluster be publically accessible

## ArgoCD (https://argo-cd.readthedocs.io/en/stable/)
- Used to automatically deploy apps to my kubernetes cluster

## Tailscale Kubernetes Operator (https://tailscale.com/docs/features/kubernetes-operator)
- Used to expose services on my cluster over my tailnet

## Reloader (https://github.com/stakater/Reloader)
- Used to automatically re-make pods in deployments/statefulsets when the configmaps and secrets they use get updated

## Longhorn (https://longhorn.io/)
- Used for cluster storage
- Configured to automatically backup to backblaze b2

# Acknowledgements
- The repo structure is heavily inspiried by [the argoproj-deployments repo](https://github.com/argoproj/argoproj-deployments) which is what deploys and configures the [ArgoCD live example](https://cd.apps.argoproj.io/) that is mentioned in the docs: https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#manage-argo-cd-using-argo-cd
  - The initial setup instructions are heavily inspiried by [the directions in that repo](https://github.com/argoproj/argoproj-deployments/blob/master/infrastructure/terraform/gcp/README.md#argo-cd-deployment)
