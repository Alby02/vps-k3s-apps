# VPS K3s Apps & Infrastructure

This repository defines the core infrastructure, applications, and database configurations for my k3s cluster. It uses a combination of **ArgoCD** for GitOps, **Helmfile** for dynamic secret injection, and **dotenvx** for encrypted secret management.

## Current Infrastructure & Apps

- **CloudNativePG (`pg-clusters`)**: A centralized PostgreSQL cluster that securely serves databases to applications across different namespaces.
- **Kingshot Role Manager**: A Discord bot and OCR-based roster management system.
- **Tailscale Operator & Exit Node**: Integrates the cluster securely into a Tailnet, providing an exit node for network routing.

---

## Onboarding a New Application

Follow these steps to securely onboard a new application to the cluster. You can onboard apps that require a database, or simple apps that only need environment variables.

### 1. Store Credentials Securely

First, generate and store any required secrets (database passwords, API tokens, OAuth credentials) securely in your `.env` file using `dotenvx`.

```bash
# Example for an app requiring a database password
dotenvx set MYAPP_DATABASE_PASSWORD "your-secure-password"

# Example for other tokens
dotenvx set MYAPP_API_TOKEN "your-api-token"
```

### 2. Configure `values/bootstrap.yaml.gotmpl`

The `bootstrap` Helm chart dynamically provisions Namespaces and Kubernetes Secrets for your applications before ArgoCD deploys them. Open `values/bootstrap.yaml.gotmpl` and add your application under the `apps` dictionary.

#### Example A: App with a Database
```yaml
  my-new-app:
    namespace: my-new-app
    envSecretName: myapp-credentials
    database:
      user: myapp_user
      password: {{ requiredEnv "MYAPP_DATABASE_PASSWORD" | quote }}
    env:
      DATABASE_PASSWORD: {{ requiredEnv "MYAPP_DATABASE_PASSWORD" | quote }}
      API_TOKEN: {{ requiredEnv "MYAPP_API_TOKEN" | quote }}
```

#### Example B: App without a Database (e.g., Tailscale)
If your app doesn't need a PostgreSQL database, simply omit the `database` block.
```yaml
  my-simple-app:
    namespace: my-simple-app
    envSecretName: simple-app-env
    env:
      API_TOKEN: {{ requiredEnv "MYAPP_API_TOKEN" | quote }}
```

### 3. Register the Database (If applicable)

If you configured a `database` block in Step 2, you must tell the shared PostgreSQL cluster to create the actual database and role (user). Open `charts/pg-shared-cluster/values.yaml` and add your app to the list:

```yaml
apps:
  # ... existing apps ...
  - name: my-new-app
    database:
      name: myapp_db
      owner: myapp_user
```

> **Note:** The `owner` must exactly match the `database.user` you defined in the bootstrap values. The `name` must exactly match the app key.

### 4. Create the ArgoCD Application

Create a new ArgoCD application manifest in the `argocd/application/` directory (e.g., `myapp.yaml`). This manifest tells ArgoCD where to find the actual Kubernetes manifests or Helm charts for your application.

When configuring your app to run in the cluster:
1. **Database Host**: Use the cross-namespace service: `shared-postgres-rw.pg-clusters.svc.cluster.local`
2. **Secrets**: Configure your app to consume the existing secret you named in Step 2 (`envSecretName`) rather than having the application's Helm chart generate a new one.

### 5. Apply the Infrastructure Changes

Run `helmfile` to apply the bootstrap namespaces and encrypted secrets.

```bash
dotenvx run -- helmfile apply
```

Once this finishes, ArgoCD will synchronize the state, CloudNativePG will spin up your database, and your new application will deploy successfully!

## License

This project is licensed under the [MIT License](LICENSE).
