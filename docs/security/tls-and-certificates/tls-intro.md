# TLS & Public Key Infrastructure (PKI) in Kubernetes

Transport Layer Security (TLS) is the foundation of cluster communication security in Kubernetes. Every interaction between users, the control plane, worker nodes, and `etcd` is encrypted and authenticated using digital certificates.

---

## Why TLS Matters in Kubernetes

Kubernetes uses **mutual TLS (mTLS)**:
- **Server Authentication:** Clients verify that the API server or `etcd` node is genuine and not an imposter (Man-in-the-Middle protection).
- **Client Authentication:** Servers verify the caller's identity (authenticating administrators, controllers, and kubelets without traditional passwords).
- **Data Encryption:** All payloads in transit (secrets, pod definitions, cluster state) are encrypted against eavesdropping.

---

## Roadmap of the TLS Series

In the following notes, we break down TLS in Kubernetes systematically:

1. **TLS Basics:** Symmetric vs. asymmetric encryption, public/private keys, and the certificate authority (CA) concept.
2. **TLS in Kubernetes:** Complete architecture map of CAs, server certificates, and client certificates used across the cluster.
3. **Certificate Generation:** Creating private keys, Certificate Signing Requests (CSR), and signing with OpenSSL.
4. **Certificate Details:** Dissecting certificate fields (CN, Organization, Subject Alternative Names - SANs, Validity).
5. **Certificates in CKA:** Common exam tasks (renewing certificates with `kubeadm`, approving CSRs via the Kubernetes API, configuring Kubeconfigs).
6. **Troubleshooting TLS:** Diagnosing and resolving `"connection refused"` errors, expired certificates, and hostname mismatches.
