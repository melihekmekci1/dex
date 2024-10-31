
# SSO Login for Kubernetes Dashboard using Dex and LDAP

This guide provides step-by-step instructions to configure SSO (Single Sign-On) for accessing the Kubernetes Dashboard using Dex as an identity provider, with LDAP for authentication.

## Prerequisites

- A running LDAP server in your internal network.
- A Kubernetes cluster with three master nodes on the same network.
- `kubectl` access to the cluster.

## Steps

### Step 1: Set Up Namespace for Dex

1. Create a namespace for Dex in your Kubernetes cluster:

   ```bash
   kubectl create namespace dex
   ```

### Step 2: Create Dex Configuration (ConfigMap)

1. Create a Dex configuration file named `dex-config.yaml` with the following content, adjusting values as needed for your environment:

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: dex-config
     namespace: dex
   data:
     config.yaml: |
       issuer: http://dex.dex:32000
       storage:
         type: kubernetes
         config:
           inCluster: true
       web:
         http: 0.0.0.0:5556
       connectors:
       - type: ldap
         id: ldap
         name: LDAP
         config:
           host: "<ldap-server-address>:389"
           insecureNoSSL: true
           bindDN: "cn=admin,dc=example,dc=com"
           bindPW: "<your-ldap-bind-password>"
           userSearch:
             baseDN: "ou=users,dc=example,dc=com"
             filter: "(objectClass=person)"
             username: uid
             idAttr: uid
             emailAttr: mail
             nameAttr: cn
           groupSearch:
             baseDN: "ou=groups,dc=example,dc=com"
             filter: "(objectClass=groupOfNames)"
             userAttr: uid
             groupAttr: member
             nameAttr: cn
       staticClients:
       - id: kubernetes
         redirectURIs:
         - 'https://<kubernetes-dashboard-url>'
         name: 'Kubernetes'
         secret: '<some-random-secret>'
   ```

   Replace:
   - `<ldap-server-address>` with the address of your LDAP server.
   - `<your-ldap-bind-password>` with your LDAP bind password.
   - `<some-random-secret>` with a randomly generated secret (use `openssl rand -base64 32`).

2. Apply the ConfigMap:

   ```bash
   kubectl apply -f dex-config.yaml
   ```

### Step 3: Deploy Dex

1. Create a file `dex-deployment.yaml` to define the Dex deployment:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: dex
     namespace: dex
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: dex
     template:
       metadata:
         labels:
           app: dex
       spec:
         containers:
         - name: dex
           image: quay.io/dexidp/dex:v2.30.0
           volumeMounts:
           - name: config
             mountPath: /etc/dex/cfg
           args:
           - serve
           - --config
           - /etc/dex/cfg/config.yaml
           ports:
           - containerPort: 5556
         volumes:
         - name: config
           configMap:
             name: dex-config
   ```

2. Deploy Dex with the following command:

   ```bash
   kubectl apply -f dex-deployment.yaml
   ```

3. Next, create a service file `dex-service.yaml` for Dex to expose it within the cluster:

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: dex
     namespace: dex
   spec:
     ports:
     - port: 32000
       targetPort: 5556
     selector:
       app: dex
   ```

4. Apply the service configuration:

   ```bash
   kubectl apply -f dex-service.yaml
   ```

### Step 4: Update the Kubernetes API Server with OIDC Flags

1. Add the following flags to each Kubernetes API server (on all three master nodes) to enable OpenID Connect (OIDC) integration:

   ```yaml
   --oidc-issuer-url=http://dex.dex:32000
   --oidc-client-id=kubernetes
   --oidc-username-claim=email
   --oidc-groups-claim=groups
   ```

   - Ensure the `--oidc-issuer-url` matches the `issuer` field in the Dex configuration file.

2. **Restart the API Server** on each master node for the changes to take effect.

### Step 5: Configure the Kubernetes Dashboard for OIDC Authentication

1. **Update the Kubernetes Dashboard deployment** with the following changes to support OIDC authentication:

   In the Kubernetes Dashboard manifest file, update or add the `args` section to specify OIDC settings:

   ```yaml
   containers:
   - name: kubernetes-dashboard
     args:
       - --authentication-mode=oidc
       - --oidc-issuer-url=http://dex.dex:32000
       - --oidc-client-id=kubernetes
   ```

2. **Apply the updated Dashboard configuration**:

   ```bash
   kubectl apply -f <dashboard-manifest-file>
   ```

### Step 6: Access the Kubernetes Dashboard

1. Open the Kubernetes Dashboard URL in your browser (`https://<kubernetes-dashboard-url>`).
2. You should be redirected to Dex for login.
3. Enter your LDAP credentials. If configured correctly, you’ll authenticate via LDAP through Dex and gain access to the Kubernetes Dashboard.

### Verification Checklist

- **Issuer URL**: Make sure it’s consistently `http://dex.dex:32000` in both Dex configuration and Kubernetes API server flags.
- **Secret Consistency**: The secret in `config.yaml` under `staticClients` should match any OAuth client settings you use.
- **LDAP Connection**: Verify the LDAP configuration (`baseDN`, `bindDN`, etc.) matches your LDAP server settings.
- **API Server Flags**: Ensure OIDC-related flags are set identically across all master nodes.

This setup should allow you to authenticate into the Kubernetes Dashboard via LDAP using Dex as your identity provider.
