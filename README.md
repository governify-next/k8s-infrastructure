# Governify Next on Kubernetes

This repository deploys the Governify Next services, MongoDB, Redis,
InfluxDB 3, and Grafana to Kubernetes. The default deployment target is a k3s
cluster with the bundled Traefik ingress controller enabled, but the manifests
also include a standard Kubernetes overlay that uses native `Ingress`.

The default k3s path uses k3s's `local-path` storage, which is best suited for
a single-node installation. For a standard Kubernetes cluster, make sure the
cluster has a default `StorageClass` or patch the PVCs for your storage class.

## What you need

- A Linux server, k3s cluster, or Kubernetes cluster with Internet access and
  enough persistent storage for 85 GiB of persistent-volume requests.
- A public IP address and DNS records for the service hostnames.
- For k3s: the bundled Traefik ingress controller enabled. Let’s Encrypt must
  be able to reach the cluster on TCP 443; HTTP traffic on TCP 80 is redirected
  to HTTPS by the Traefik configuration in this repository.
- For standard Kubernetes: an ingress controller and a TLS certificate secret,
  or an equivalent certificate-management setup, for the configured hostnames.
- `kubectl` configured to access the cluster. The commands below assume it is
  run by a cluster administrator.
- Credentials for the Google OpenID Connect application used by Scope Manager.
  Its redirect URI must match the configured public hostname.

The images currently use the `develop` tag. Pin image tags in `base/services/` to
an immutable release before a production deployment.

## 1. Create a k3s cluster

Before installing, review the [k3s operating-system requirements](https://docs.k3s.io/installation/requirements).
Then install k3s with its default components by
running the following command:

```sh
curl -sfL https://get.k3s.io | sh -
```

## 2. Configure public hostnames and OIDC

The checked-in configuration uses these hostnames:

| Service | Public URL |
| --- | --- |
| Scope Manager | `https://scope-manager.k8s.next.governify.io` |
| Registry | `https://registry.k8s.next.governify.io` |
| Computer | `https://computer.k8s.next.governify.io` |
| Fetcher | `https://fetcher.k8s.next.governify.io` |
| Reporter | `https://reporter.k8s.next.governify.io` |
| Director | `https://director.k8s.next.governify.io` |
| Join (backend) | `https://join-backend.k8s.next.governify.io` |
| Join | `https://join.k8s.next.governify.io` |
| Grafana | `https://grafana.k8s.next.governify.io` |
| Kubernetes Dashboard | `https://headlamp.k8s.next.governify.io` |

Create A/AAAA records for every hostname above, pointing to the ingress
controller entrypoint. If using another domain, replace the host rules in
`base/ingress.yaml` and update these related public URLs before deployment:

- `OIDC_REDIRECT_URI` in `base/services/scope-manager.yaml`
- `GRAFANA_PUBLIC_URL` in `base/services/reporter.yaml`
- `GF_SERVER_ROOT_URL` in `base/infra/grafana.yaml`

Headlamp is exposed as the Kubernetes dashboard. It creates a
`governify-headlamp-admin` service account bound to `cluster-admin` for
dashboard login. After deployment, read its token with:

```sh
kubectl -n governify-next get secret governify-headlamp-admin-token -o jsonpath='{.data.token}' | base64 -d
```

Register the resulting Scope Manager redirect URI with the OIDC provider. The
current manifest uses Google as its issuer (`https://accounts.google.com`).

## 3. Create the runtime secrets

Secrets are deliberately not stored in Git. Copy the example and replace every
placeholder value:

```sh
cp secrets.example.yaml secrets.yaml
```

Generate strong values, for example:

```sh
openssl rand -hex 32   # jwt_secret
openssl rand -hex 24   # Grafana password or token material
```

Important details:

- `jwt_secret` should be a random value of at least 32 bytes.
- `influx_token` and the `token` value in `influx_admin_token.json` must be
  exactly the same value.
- Set `oidc_client_id` and `oidc_client_secret` to the OIDC application's
  credentials.
- Keep `secrets.yaml` private. It is ignored by Git; use a secret manager or
  sealed-secret workflow if your deployment process requires GitOps.

## 4. Deploy to k3s

Run these commands from the repository root after DNS is in place. Creating the
namespace first lets the Secret be applied before workloads are created. The
Traefik configuration enables HTTPS redirects, ACME TLS certificates, and
persistent ACME state.

```sh
kubectl apply -f base/core.yaml
kubectl apply -f secrets.yaml
kubectl apply -k .
```

The repository root points to the k3s overlay by default.

## 5. Deploy to standard Kubernetes

Use this path for clusters that provide a standard ingress controller instead
of k3s's bundled Traefik installation.

Before applying the overlay, either create a TLS secret named
`governify-next-tls` in the `governify-next` namespace or adapt
`platform/kubernetes/ingress-patch.yaml` to your certificate manager. If your
ingress controller is not the cluster default, add `spec.ingressClassName` with
an overlay patch.

```sh
kubectl apply -f base/core.yaml
kubectl apply -f secrets.yaml
kubectl apply -k platform/kubernetes
```

## 6. Deploy with Argo CD

Use this path when the cluster should continuously reconcile the manifests from
the Git repository instead of relying on repeated local `kubectl apply`
commands. The repository is hosted at
[`governify-next/k8s-infrastructure`](https://github.com/governify-next/k8s-infrastructure).

Install Argo CD in the cluster:

```sh
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Install Argo CD Image Updater in the same namespace. The default Image Updater
installation watches its own namespace, which is where the `Application` and
`ImageUpdater` resources in this repository are created:

```sh
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/config/install.yaml
```

Create the runtime secrets before the first sync. This keeps secret material out
of Git while still allowing Argo CD to manage the rest of the deployment.
For this you will previously need to create the corresponding namespace.

```sh
kubectl create namespace governify-next
kubectl apply -f secrets.yaml
```

Then apply the Argo CD bootstrap resources from this repository:

```sh
kubectl apply -k argocd
```

The checked-in Argo CD `Application` tracks the `develop` branch and uses
`path: .`, so the default deployment is the k3s overlay. For a standard
Kubernetes cluster, change `spec.source.path` in `argocd/governify-next.yaml`
to `platform/kubernetes` before applying it.

The checked-in `ImageUpdater` tracks the eight Governify service images that use
the `develop` tag. It uses the `digest` strategy so a new image pushed to the
same mutable tag causes Argo CD to deploy the new image digest without requiring
manual Kubernetes YAML edits for each commit.

To access the Argo CD UI locally:

```sh
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Then open `https://localhost:8080`. The initial admin password can be read from
the bootstrap secret:

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

For a fully GitOps-managed production setup, replace the manually applied
`secrets.yaml` step with a sealed-secret, SOPS, or external-secret workflow and
commit only encrypted or external secret references.
