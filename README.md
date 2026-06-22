# Governify Next on k3s

This repository deploys the Governify Next services, MongoDB, Redis,
InfluxDB 3, and Grafana to Kubernetes. It is intended for a k3s cluster with
the bundled Traefik ingress controller enabled.

The base configuration is production-oriented but uses k3s's default
`local-path` storage, which is best suited for a single-node installation, but
the persistent data will not survive failure of that node.

## What you need

- A Linux server (or k3s cluster) with Internet access and enough local disk
  for 85 GiB of persistent-volume requests.
- A public IP address and DNS records for the service hostnames. Let’s Encrypt
  must be able to reach the cluster on TCP 443; HTTP traffic on TCP 80 is
  redirected to HTTPS.
- `kubectl` configured to access the cluster. The commands below assume it is
  run by a cluster administrator.
- Credentials for the Google OpenID Connect application used by Scope Manager.
  Its redirect URI must match the configured public hostname.

The images currently use the `develop` tag. Pin image tags in `base/apps/` to
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
| Collector | `https://collector.k8s.next.governify.io` |
| Reporter | `https://reporter.k8s.next.governify.io` |
| Director | `https://director.k8s.next.governify.io` |
| Grafana | `https://grafana.k8s.next.governify.io` |

Create A/AAAA records for every hostname above, pointing to the Traefik
entrypoint (normally the k3s server's public address). If using another domain,
replace the host rules in `base/ingress.yaml` and update these related public
URLs before deployment:

- `OIDC_REDIRECT_URI` in `base/apps/scope-manager.yaml`
- `GRAFANA_PUBLIC_URL` in `base/apps/reporter.yaml`
- `GF_SERVER_ROOT_URL` in `base/data/grafana.yaml`

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

## 4. Deploy

Run these commands from the repository root after DNS is in place. Creating the
namespace first lets the Secret be applied before workloads are created. The
Traefik configuration enables HTTPS redirects, ACME TLS certificates, and
persistent ACME state.

```sh
kubectl apply -f base/core.yaml
kubectl apply -f secrets.yaml
kubectl apply -f platform/k3s/traefik.yaml
kubectl apply -k base
```