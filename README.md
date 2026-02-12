# Dolibarr Multi-Tenant K3s Deployment

Deploy isolated Dolibarr ERP instances per customer on a K3s cluster via GitHub Actions.

Each customer gets their own namespace, MariaDB database, and Dolibarr instance accessible at `https://{customer}.moyocorps.com`.

## Prerequisites

1. **K3s cluster** with Traefik ingress controller
2. **Cloudflare DNS**: Wildcard `*.moyocorps.com` A record pointing to VPS IP (proxied)
3. **Cloudflare SSL/TLS**: Mode set to "Full" for `moyocorps.com`
4. **GitHub Secret**: `KUBE_CONFIG` — base64-encoded kubeconfig for the K3s cluster:
   ```bash
   cat /etc/rancher/k3s/k3s.yaml | base64 -w 0
   ```
   Update the `server:` URL to your VPS public IP before encoding.

## Usage

### Deploy a customer

1. Go to **Actions** > **Deploy/Destroy Customer Dolibarr Instance**
2. Click **Run workflow**
3. Fill in:
   - `customer_name`: e.g., `acme` (lowercase, 3-20 chars, alphanumeric + hyphens)
   - `dolibarr_version`: e.g., `19` (default)
   - `action`: `deploy`
4. Access at `https://acme.moyocorps.com`

### Get admin password

```bash
kubectl get secret dolibarr-secrets -n dolibarr-acme \
  -o jsonpath='{.data.DOLI_ADMIN_PASSWORD}' | base64 -d
```

Default admin username: `admin`

### Update a customer (re-deploy)

Run the workflow again with the same `customer_name` and a new `dolibarr_version`. Secrets and data are preserved — only the deployment is updated.

### Destroy a customer

Run the workflow with `action: destroy`. This deletes the entire namespace including all data (PVCs).

## Architecture

```
Internet → Cloudflare (TLS) → VPS:80 → Traefik → Ingress → Dolibarr Pod
                                                          → MariaDB Pod
```

Each customer runs in namespace `dolibarr-{name}` with:
- **MariaDB 11** — 5Gi persistent storage
- **Dolibarr** (tuxgasy/dolibarr) — 5Gi document storage
- **Traefik Ingress** — routes `{name}.moyocorps.com` to the Dolibarr service

## Project Structure

```
├── .github/workflows/
│   └── deploy-customer.yml    # CI/CD pipeline
├── k8s/templates/
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── mariadb-pvc.yaml
│   ├── mariadb-deployment.yaml
│   ├── mariadb-service.yaml
│   ├── dolibarr-pvc.yaml
│   ├── dolibarr-deployment.yaml
│   ├── dolibarr-service.yaml
│   └── ingress.yaml
└── README.md
```

Templates use `__PLACEHOLDER__` values that are substituted during CI:
- `__CUSTOMER_NAME__` — the customer identifier
- `__CUSTOMER_NAMESPACE__` — `dolibarr-{name}`
- `__CUSTOMER_DOMAIN__` — `{name}.moyocorps.com`
- `__DOLIBARR_VERSION__` — image tag

## Troubleshooting

```bash
# Check pod status
kubectl get pods -n dolibarr-acme

# View Dolibarr logs
kubectl logs -n dolibarr-acme deployment/dolibarr

# View MariaDB logs
kubectl logs -n dolibarr-acme deployment/mariadb

# Check ingress
kubectl get ingress -n dolibarr-acme

# List all customer namespaces
kubectl get namespaces -l app=dolibarr
```
