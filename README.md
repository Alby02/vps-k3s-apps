# Managing Apps and Databases

This repository uses a combination of `helmfile`, `dotenvx`, and ArgoCD to manage applications and their PostgreSQL databases dynamically. 

The centralized PostgreSQL cluster (CloudNativePG) is deployed in the `pg-clusters` namespace.

## How to add a new app with a database

Follow these steps to securely onboard a new application that requires a database in the shared cluster.

### 1. Set the secure credentials

First, generate and store the database password (and any other secrets) securely in your `.env` file using `dotenvx`.

```bash
# Add a database password
dotenvx set MYAPP_DATABASE_PASSWORD "your-secure-password"

# Add any other required secrets
dotenvx set MYAPP_API_TOKEN "your-api-token"
```

### 2. Configure `helmfile.yaml`

Update `helmfile.yaml` to define your new application. This configuration tells the `bootstrap` chart to create the namespace and the necessary credential secrets.

Add a new block under `apps`:

```yaml
        apps:
          # ... existing apps ...
          my-new-app:
            namespace: my-new-app
            database:
              secretName: myapp-credentials
              user: myapp_user
              password: "{{ requiredEnv \"MYAPP_DATABASE_PASSWORD\" }}"
            env:
              # Inject the database password for the app to consume
              DATABASE_PASSWORD: "{{ requiredEnv \"MYAPP_DATABASE_PASSWORD\" }}"
              # Add any other environment variables you want in the secret
              API_TOKEN: "{{ requiredEnv \"MYAPP_API_TOKEN\" }}"
```

### 3. Register the database in the PostgreSQL Cluster

Tell the shared PostgreSQL cluster to create the database and the role (user) for your application. Open `charts/pg-shared-cluster/values.yaml` and add your app to the `apps` list:

```yaml
apps:
  # ... existing apps ...
  - name: my-new-app
    database:
      name: myapp_db
      owner: myapp_user
```

> **Note:** The `owner` must exactly match the `database.user` you defined in `helmfile.yaml`. The `name` must exactly match the app key in `helmfile.yaml`.

### 4. Create the ArgoCD Application

Finally, create a new ArgoCD application manifest in `argocd/application/myapp.yaml` to deploy your actual application's Helm chart. 

When configuring your Helm chart values in the ArgoCD application, be sure to:
1. Set the database host to the cross-namespace service: `shared-postgres-rw.pg-clusters.svc.cluster.local`
2. Configure your app to consume the existing secret you named in `helmfile.yaml` (e.g., `myapp-credentials`) rather than having the Helm chart create one.

### 5. Apply the infrastructure changes

Run `helmfile` to apply the namespaces and secrets, then let ArgoCD handle the rest.

```bash
dotenvx run -- helmfile apply
```

Once this finishes, ArgoCD will synchronize, CloudNativePG will create your database/role, and your new application will spin up connected to it!
