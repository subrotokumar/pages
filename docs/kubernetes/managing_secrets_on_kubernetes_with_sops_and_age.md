---
hide:
    - toc
    - navigation
---

# Managing Kubernetes Secrets with SOPS and Age

> A production blueprint for zero trust GitOps secrets management, envelope encryption cryptography, KSOPS in-memory rendering, and the audit boundaries between SOPS and HashiCorp Vault.

## The Flawed Foundation of Kubernetes Secrets

Most teams discover the Kubernetes secret management problem through a predictable progression. You author a deployment manifest, require database credentials, and inject them into a Kubernetes Secret resource. The manifest is committed to version control. At code review, an engineer points out that base64 encoding is not encryption. A base64 string is merely plaintext represented in an alternative ASCII character set:

```bash
# Decoding a base64 encoded Kubernetes secret requires a single shell command
echo "bXlTM2N1cjNQQHNzd29yZA==" | base64 -d
# Output: myS3cur3P@ssword
```

The conventional advice is straightforward: never commit plaintext secrets to git repositories.

Unfortunately, removing secrets from git breaks GitOps workflows. If secret resources are applied manually through ad hoc `kubectl apply` commands on engineer workstations, version history vanishes, pull request peer reviews disappear, configuration drift accelerates, and rebuilding a disaster recovery cluster from scratch becomes impossible.

Mozilla SOPS (Secrets OPerationS) paired with Age solves this dilemma. It enables committing fully encrypted, tamper evident secret manifests directly into version control alongside standard application code, while allowing automated continuous delivery controllers such as ArgoCD to decrypt them dynamically in memory during deployment.

## The Hybrid Cryptography Model of SOPS

Understanding how SOPS secures data is critical for threat modeling and compliance reviews. SOPS does not encrypt entire files blindly with a public key. It employs envelope encryption with value level granularity.

### Value Level Envelope Encryption

When you run SOPS against a YAML manifest, the tool generates a random 256-bit symmetric Data Encryption Key (DEK) using system entropy. This DEK encrypts each sensitive leaf value individually using AES-256-GCM. 

Field keys remain in plaintext. This architectural decision is vital for operational safety:

```yaml
# Plaintext view during git pull request review
database:
  host: primary-db.internal.net               # Plaintext configuration
  port: 5432                                 # Plaintext configuration
  username: ENC[AES256_GCM,data:dHhz...,type:str]
  password: ENC[AES256_GCM,data:eUts...,type:str]
```

Reviewers can verify configuration changes, hostnames, and key names without exposing confidential credentials.

The DEK itself is subsequently encrypted using each configured recipient key (such as an Age public key or cloud KMS master key) and stored in a metadata block appended to the bottom of the file.

### Cryptographic Integrity via MAC

SOPS computes a Message Authentication Code (MAC) across all keys and values in the document. This MAC is encrypted with the DEK and stored in the metadata header:

```yaml
sops:
  age:
    - recipient: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
      enc: |
        -----BEGIN AGE ENCRYPTED FILE-----
        YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUxOS...
        -----END AGE ENCRYPTED FILE-----
  lastmodified: "2026-03-15T10:30:00Z"
  mac: ENC[AES256_GCM,data:4gY3...,type:str]
  version: 3.9.0
```

If an attacker modifies an encrypted ciphertext value or injects an unencrypted key into the file directly, the MAC verification fails during decryption and the operation terminates immediately. Tamper detection is mathematically enforced.

## Why Age Replaced GPG for Modern Infrastructure

For years, SOPS deployments relied on GPG for asymmetric encryption. GPG carries substantial operational friction: complex key server synchronization, web of trust management, fragile keyring files, and cumbersome passphrase prompts.

Age, designed by Filippo Valsorda, replaces GPG with a minimal, modern cryptographic toolchain. Age key generation produces a lightweight keypair based on audited primitives: X25519 for Diffie-Hellman key exchange and ChaCha20-Poly1305 for symmetric authentication.

An Age public key is an explicit, human readable string:

```
age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
```

The corresponding private key is stored as a single line in a plain text file without external daemon dependencies:

```
AGE-SECRET-KEY-1QQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQ
```

## Step by Step Setup

### 1. Generating Keys and Environment Configuration

Generate an identity file on a secure workstation:

```bash
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt

# Export the key location to shell environment
export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt
```

### 2. Defining Repository Encryption Rules

Place a `.sops.yaml` configuration file at the root of your GitOps repository to declare which public keys encrypt specific file paths:

```yaml
creation_rules:
  # Staging environment secrets
  - path_regex: envs/staging/.*\.yaml
    age: age1staging7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p

  # Production environment secrets requiring dual recipient encryption
  - path_regex: envs/prod/.*\.yaml
    age: >-
      age1prodql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p,
      age1ciops24pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac9z
```

### 3. File Operations Workflow

Encrypting, editing, and rotating credentials is handled via the SOPS CLI:

```bash
# Encrypt an existing plaintext manifest in place
sops --encrypt --in-place envs/prod/secrets.yaml

# Open encrypted file in editor, decrypt in memory, re-encrypt atomically on save
sops envs/prod/secrets.yaml

# Re-encrypt data encryption key when updating recipient lists
sops --rotate --in-place envs/prod/secrets.yaml
```

## In-Cluster Decryption with KSOPS and ArgoCD

In an automated GitOps architecture, secrets must be decrypted inside the cluster without persisting plaintext files to disks or exposing private keys to client developers.

KSOPS is a Kustomize executive generator plugin designed for this exact function. During the Kustomize rendering phase, KSOPS intercepts referenced SOPS files, reads the Age private key from an isolated Kubernetes secret, decrypts payloads in memory, and passes standard Kubernetes Secret objects to the API server.

```
+--------------------------------------------------------------+
| Git Repository (Encrypted YAML)                              |
+--------------------------------------------------------------+
                               |
                               v (git pull)
+--------------------------------------------------------------+
| ArgoCD Repo Server Pod                                       |
|                                                              |
|   +-------------------+       +--------------------------+   |
|   | Kustomize Build   | ----> | KSOPS Exec Plugin        |   |
|   +-------------------+       | (Mounts Age Private Key) |   |
|                               +--------------------------+   |
|                                           |                  |
|                                           v (In Memory)      |
+--------------------------------------------------------------+
                               |
                               v (TLS API Post)
+--------------------------------------------------------------+
| Kubernetes API Server                                        |
| Plaintext Secret stored in memory / etcd encrypted at rest   |
+--------------------------------------------------------------+
```

### Configuring the ArgoCD Repo Server

Mount the Age private key secret and inject the KSOPS binary into the `argocd-repo-server` deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
  namespace: argocd
spec:
  template:
    spec:
      volumes:
        - name: sops-age-key
          secret:
            secretName: sops-age-key
        - name: custom-tools
          emptyDir: {}
      initContainers:
        - name: install-ksops
          image: viaductoss/ksops:v4.3.2
          command: ["/bin/sh", "-c"]
          args:
            - cp /usr/local/bin/ksops /custom-tools/
          volumeMounts:
            - mountPath: /custom-tools
              name: custom-tools
      containers:
        - name: argocd-repo-server
          env:
            - name: SOPS_AGE_KEY_FILE
              value: /home/argocd/.config/sops/age/keys.txt
          volumeMounts:
            - mountPath: /home/argocd/.config/sops/age/keys.txt
              subPath: keys.txt
              name: sops-age-key
              readOnly: true
            - mountPath: /usr/local/bin/ksops
              subPath: ksops
              name: custom-tools
```

### Kustomization Manifest Architecture

Organize your application manifests so that the secret generator integrates seamlessly with standard resources:

```yaml
# apps/payments/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

generators:
  - secret-generator.yaml
```

```yaml
# apps/payments/secret-generator.yaml
apiVersion: viaductoss.github.io/v1alpha1
kind: ksops
metadata:
  name: payments-secret-generator
  annotations:
    config.kubernetes.io/function: |
      exec:
        path: ksops
files:
  - ./secrets.enc.yaml
```

## Architectural Trade-Offs: SOPS vs External Secrets vs Vault

Selecting a secret management strategy requires understanding operational failure domains.

| Dimension | SOPS + Age (KSOPS) | External Secrets Operator (ESO) | HashiCorp Vault Agent Sidecar |
| --- | --- | --- | --- |
| Storage Location | Encrypted inside Git | Cloud Secret Manager (AWS/GCP/Vault) | Vault Cluster |
| Secret Decryption Point | Repo server at render time | In-cluster operator controller | Pod init container / agent sidecar |
| Failure Domain | Static decrypt (no runtime dependencies) | External cloud API availability | Vault cluster availability |
| Dynamic Secret Generation | Not supported | Supported via Vault provider | Supported natively |
| Read Access Auditing | Not supported (git tracks changes only) | Supported via cloud provider IAM logs | Full audit trail on every read |
| Operational Complexity | Minimal (CLI binary and keypair) | Moderate (CRDs and IAM roles) | High (Vault management and unsealing) |

### The Critical Audit Boundary

The primary limitation of SOPS involves access auditing. Version control records who committed a modification to an encrypted file, but it cannot record who decrypted and read the file locally. 

If regulatory compliance (such as PCI-DSS or strict HIPAA boundaries) mandates verifiable logs of every individual credential read operation, SOPS is insufficient on its own. For such environments, HashiCorp Vault or AWS Secrets Manager with CloudTrail is mandatory.

Conversely, for teams seeking a robust GitOps workflow that eliminates plaintext secrets, prevents accidental exposure during code reviews, and requires zero external cloud API dependencies during cluster reconciliation, SOPS with Age provides the most reliable architecture available.