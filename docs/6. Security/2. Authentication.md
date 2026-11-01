# Authentication

Every request to the Kubernetes API server must be authenticated before it can be authorized and processed.

---

## 1. Types of Accounts: Users vs. ServiceAccounts

Kubernetes distinguishes between two types of accounts:

| Account Type | Target Identity | Management in Kubernetes |
| :--- | :--- | :--- |
| **Normal Users** | Human operators, developers, administrators. | **Not managed by Kubernetes API objects.** Kubernetes does not have a `User` kind; you cannot run `kubectl create user`. It relies on external identity providers, X.509 certificates, or OIDC tokens. |
| **ServiceAccounts** | Applications, Pods, CI/CD pipelines, controllers. | **Managed natively by the Kubernetes API.** Can be created and managed via `kubectl create serviceaccount <name>`. |

```sh
# Create a service account
kubectl create serviceaccount sa1

# List service accounts
kubectl get serviceaccount
```

---

## 2. What `kube-apiserver` Extracts from Authentication

Regardless of the authentication mechanism used, once identity is verified, the `kube-apiserver` extracts four standard attributes for subsequent authorization:
1. **Username:** Unique identifier (e.g., `jane`, `system:node:node01`).
2. **UID:** Unique string identifying the user.
3. **Groups:** List of logical memberships (e.g., `system:authenticated`, `developers`, `system:masters`).
4. **Extra Fields:** Key-value metadata used by authorizers or audit logging.

---

## 3. Supported Authentication Mechanisms

### A. X.509 Client Certificates (Most Common for Admins & Components)
- Standard mechanism for cluster administrators and core components (`kubelet`, `kube-scheduler`, `kube-controller-manager`).
- Certificates are signed by the cluster CA (`--client-ca-file`).
- Identity is mapped directly from the certificate's subject:
  - **`CN` (Common Name):** Mapped to the **Username** (e.g. `CN=jane`).
  - **`O` (Organization):** Mapped to **Groups** (e.g. `O=developers`, `O=system:masters`).
### B. OpenID Connect (OIDC) Tokens (Modern Enterprise Standard)
- Connects Kubernetes to external identity providers (Google, Keycloak, Azure AD, Okta).
- Relies on OAuth2 and JWT (`id_token`) standards.
- The user authenticates with the external provider, receives a JWT, and sends it via `Authorization: Bearer <id_token>`.
- The API server verifies the token signature using the provider's public key and extracts claims.
- User authenticates with the external provider, hets a JWT, and sends it via Bearer token: `Authentication: Bearer <jwt-token>`
### C. Webhook Token Authentication
- A mechanism to verify bearer tokens using a custom external HTTP service.
- When a Bearer token arrives, `kube-apiserver` sends a `TokenReview` request via HTTP POST to an external `webhook` service.
- The external service validates the token and responds with user details: `{ "authenticated": true, "user": { "username": "omer", "groups": ["devs"] } }`
### D. Authenticating Proxy
- A reverse proxy (e.g., NGINX, Envoy) sits in front of the `kube-apiserver`.
- The proxy handles user authentication and forwards the request to `kube-apiserver` with custom HTTP headers:
  - `X-Remote-User: jane`
  - `X-Remote-Group: developers`
- `kube-apiserver` trusts these headers because it validates the proxy's client certificate (`--requestheader-client-ca-file`).

### E. Static Token File (Legacy / Lab Only)
- A plain text CSV file configured via `--token-auth-file`.
- Format: `token,user,uid,"group1,group2"`
- Passed via: `Authorization: Bearer <token>`
- **Not recommended for production:** Tokens are stored in clear text, and updating/revoking tokens requires restarting `kube-apiserver`. *(Note: Static password file `--basic-auth-file` was completely removed in v1.19).*

---

## 4. Anonymous Requests

When enabled (`--anonymous-auth=true`, default in most clusters), requests that fail or omit credentials are assigned the username `system:anonymous` and group `system:unauthenticated`. RBAC then governs whether anonymous users can access specific public endpoints (like `/healthz`, `/livez`, `/readyz`).