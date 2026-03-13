---
hide:
    - toc
---

# Managing Secrets on Kubernetes with SOPS and Age 
 
Most teams discover the Kubernetes secrets problem the same way. You write a deployment manifest, you need a database password, and you put it in a Secret object. Then someone asks where that Secret came from. Then someone asks how it gets into the cluster. Then someone asks whether it's safe to commit it to git. 
 
The answer to that last question is no — base64 is not encryption. A Kubernetes Secret stored in git is plaintext with extra steps. 
 
This blog covers one specific solution to that problem: SOPS paired with Age, integrated into a Kubernetes GitOps workflow via KSOPS and ArgoCD. Not because it's the only solution, but because it's the right one for teams that want secrets living in git alongside their application manifests without being reckless about it. 
 
--- 
 
## The Problem with Kubernetes Secrets 
 
Before getting into SOPS, it's worth being precise about what the actual problem is. 
 
Kubernetes Secret objects store data as base64-encoded strings. Base64 is an encoding, not encryption. Anyone with read access to the Secret object, etcd, or the git repository where the manifest lives can trivially decode it. 
 
```bash 
# This is all it takes to decode a Kubernetes secret value 
echo "bXlTM2N1cjNQQHNzd29yZA==" | base64 -d 
# myS3cur3P@ssword 
``` 
 
The standard advice is: never commit Secret manifests to git. Which creates a different problem — if your secrets aren't in git, they're outside your GitOps pipeline. Someone applied them manually. Maybe. At some point. From somewhere. 
 
You lose version history. You lose PR review. You lose the ability to audit what changed and when. You lose the single source of truth that GitOps is supposed to give you. 
 
SOPS solves this by letting you commit secrets to git in an encrypted form that is safe to store, safe to review, and automatically decryptable at deploy time by the cluster that holds the right key. 
 
--- 
 
## What SOPS Is 
 
SOPS (Secrets OPerationS) is an open-source tool from Mozilla that encrypts structured files at the value level. It supports YAML, JSON, ENV, and INI formats. 
 
The key design decision: SOPS encrypts values, not the entire file. Field names remain in plaintext. This means a teammate opening an encrypted secrets file in a PR review sees the complete structure of the configuration — every key that exists — without seeing any of the actual secret values. 
 
```yaml 
# What your teammate sees in a PR diff 
database: 
  host: prod-db.internal.company.com        # plaintext 
  port: 5432                                 # plaintext 
  username: ENC[AES256_GCM,data:Hk3p...,type:str] 
  password: ENC[AES256_GCM,data:xK92m...,type:str] 
api_key: ENC[AES256_GCM,data:pL84n...,type:str] 
``` 
 
The review can verify that the right fields exist, that sensitive values are encrypted, and that non-sensitive values (host, port) are what they should be. No credentials are exposed in the diff. 
 
--- 
 
## What Age Is 
 
Age is a modern file encryption tool built as a deliberate replacement for GPG. The design goals are minimal complexity and minimal footprint. 
 
Where GPG requires key signing, key servers, a web of trust, passphrases, and a working knowledge of its configuration model to use correctly, Age requires one thing: a keypair. 
 
```bash 
# Generate a keypair 
age-keygen -o key.txt 
 
# key.txt contains: 
# Public key:  age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p 
# Private key: AGE-SECRET-KEY-1QQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQ 
``` 
 
The public key encrypts. The private key decrypts. No passphrases by default. No key servers. No expiry management. The cryptography underneath is X25519 for key exchange and ChaCha20-Poly1305 for symmetric encryption — modern, audited, and not GPG. 
 
SOPS supports multiple encryption backends: GPG, AWS KMS, GCP KMS, Azure Key Vault, and HashiCorp Vault transit. Age has become the default for teams that don't need cloud KMS integration. For most Kubernetes setups, Age is the right choice. 
 
--- 
 
## How SOPS Encryption Works 
 
SOPS uses a hybrid encryption model. Understanding this matters because it explains why the approach is secure and what the actual threat model looks like. 
 
When you encrypt a file with SOPS: 
 
1. SOPS generates a random 256-bit data encryption key (DEK) 
2. The DEK encrypts each individual value in the file using AES-256-GCM 
3. The DEK itself is encrypted using your Age public key (via X25519 key exchange + ChaCha20-Poly1305) 
4. The encrypted DEK is stored in the SOPS metadata block appended to the file 
5. A MAC (message authentication code) is computed over all keys and values and stored encrypted in the metadata 
 
The MAC is critical. If anyone modifies an encrypted value directly in the file without going through SOPS, the MAC check fails on decryption and the operation is rejected entirely. Tamper detection is built in. 
 
```yaml 
# The sops metadata block at the bottom of every encrypted file 
sops: 
  age: 
    - recipient: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p 
      enc: | 
        -----BEGIN AGE ENCRYPTED FILE----- 
        YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUxOSBrZVBxc3... 
        -----END AGE ENCRYPTED FILE----- 
  lastmodified: "2024-01-15T10:30:00Z" 
  mac: ENC[AES256_GCM,data:abc123...,type:str] 
  version: 3.8.1 
``` 
 
Multiple recipients are supported by encrypting the same DEK to multiple public keys. Each recipient can independently decrypt the file using their private key. Adding a new team member means re-encrypting only the DEK with their public key — not re-encrypting every secret value. 
 
--- 
 
## Before and After: What Encryption Actually Looks Like 
 
**Before encryption — never commit this:** 
 
```yaml 
database: 
  host: prod-db.internal.company.com 
  port: 5432 
  username: app_user 
  password: myS3cur3P@ssword 
api_key: sk-prod-abc123xyz 
jwt_secret: supersecretjwtkey123 
feature_flags: 
  dark_mode: true 
  beta_users: false 
``` 
 
**After encryption — safe to commit:** 
 
```yaml 
database: 
  host: prod-db.internal.company.com   # non-sensitive, left plaintext 
  port: 5432                            # non-sensitive, left plaintext 
  username: ENC[AES256_GCM,data:Hk3p...,type:str] 
  password: ENC[AES256_GCM,data:xK92m...,type:str] 
api_key: ENC[AES256_GCM,data:pL84n...,type:str] 
jwt_secret: ENC[AES256_GCM,data:mN71k...,type:str] 
feature_flags: 
  dark_mode: true                       # non-sensitive, left plaintext 
  beta_users: false                     # non-sensitive, left plaintext 
sops: 
  age: 
    - recipient: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p 
      enc: | 
        -----BEGIN AGE ENCRYPTED FILE----- 
        YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUxOSBrZVBxc3... 
        -----END AGE ENCRYPTED FILE----- 
  lastmodified: "2024-01-15T10:30:00Z" 
  mac: ENC[AES256_GCM,data:integrity_check...,type:str] 
  version: 3.8.1 
``` 
 
You control which fields get encrypted. SOPS has a `--encrypted-regex` flag and supports `encrypted_regex` in `.sops.yaml` if you want to be precise about it. By default, SOPS encrypts all string values. Non-string values (integers, booleans) are left plaintext because encrypting them would lose type information. 
 
--- 
 
## Setting Up SOPS with Age 
 
### Install the tools 
 
```bash 
# Install SOPS 
# Arch / EndeavourOS 
yay -S sops 
 
# macOS 
brew install sops 
 
# Or download binary directly 
curl -LO https://github.com/getsops/sops/releases/latest/download/sops-v3.x.x.linux.amd64 
chmod +x sops-v3.x.x.linux.amd64 
sudo mv sops-v3.x.x.linux.amd64 /usr/local/bin/sops 
 
# Install Age 
# Arch / EndeavourOS 
sudo pacman -S age 
 
# macOS 
brew install age 
``` 
 
### Generate your Age keypair 
 
```bash 
age-keygen -o ~/.config/sops/age/keys.txt 
 
# Output: 
# Public key: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p 
# Secret key written to ~/.config/sops/age/keys.txt 
 
# Set the environment variable so SOPS knows where your key is 
export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt 
``` 
 
Add that export to your shell profile so it persists across sessions. 
 
### Configure SOPS 
 
Create a `.sops.yaml` at the root of your repository. This tells SOPS which keys to use for which files. 
 
```yaml 
# .sops.yaml 
creation_rules: 
  # All secrets files use the team Age key 
  - path_regex: secrets/.*\.yaml 
    age: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p 
 
  # Production uses an additional AWS KMS key for extra control 
  - path_regex: secrets/prod/.*\.yaml 
    age: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p 
    kms: arn:aws:kms:us-east-1:123456789012:key/your-key-id 
 
  # Multiple recipients — both team members can decrypt 
  - path_regex: secrets/shared/.*\.yaml 
    age: >- 
      age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p, 
      age1lggyhqjf6kftyq5sr0a9slt7kcg7kx7y8fqkn4y3xhq2qp9zjqs8r0kvs 
``` 
 
### Encrypt and decrypt files 
 
```bash 
# Encrypt a file in place 
sops --encrypt --in-place secrets/db-credentials.yaml 
 
# Encrypt and write to a new file 
sops --encrypt secrets/db-credentials.yaml > secrets/db-credentials.enc.yaml 
 
# Decrypt to stdout (never write decrypted files to disk if you can avoid it) 
sops --decrypt secrets/db-credentials.yaml 
 
# Edit an encrypted file directly (SOPS decrypts in memory, opens editor, re-encrypts on save) 
sops secrets/db-credentials.yaml 
 
# Rotate the DEK (re-encrypt with current keys, useful after removing a recipient) 
sops --rotate --in-place secrets/db-credentials.yaml 
 
# Update file to add a new recipient 
sops --rotate --add-age age1newrecipientpublickey... --in-place secrets/db-credentials.yaml 
``` 
 
--- 
 
## Kubernetes Integration: KSOPS and ArgoCD 
 
KSOPS is a Kustomize exec plugin that integrates SOPS decryption directly into the ArgoCD rendering pipeline. When ArgoCD renders your Kustomize application, KSOPS intercepts the SOPS-encrypted files, decrypts them in memory, and passes the plaintext to Kubernetes. The private key never leaves the cluster. Decrypted values never touch git. 
 
### Architecture overview 
 
``` 
Git repo (encrypted secrets) 
       │ 
       ▼ 
ArgoCD pulls repo 
       │ 
       ▼ 
Kustomize render 
       │ 
       ▼ 
KSOPS plugin invoked 
       │ 
       ├── reads Age private key from Kubernetes Secret (argocd namespace) 
       ├── decrypts values in memory 
       └── passes plaintext manifest to Kubernetes API 
                │ 
                ▼ 
        Kubernetes Secret created in target namespace 
``` 
 
### Store the Age private key in the cluster 
 
```bash 
# The private key needs to live in the cluster as a Kubernetes Secret 
# This is the one secret you can't put in git — store it securely elsewhere 
# (password manager, printed backup, separate secure storage) 
 
kubectl create secret generic sops-age-key \ 
  --namespace argocd \ 
  --from-file=keys.txt=~/.config/sops/age/keys.txt 
``` 
 
> **This key is the trust anchor for your entire secrets setup.** Restrict access to it with RBAC. Back it up before you need it. If this key is compromised, all secrets encrypted with the corresponding public key are compromised and need to be rotated. 
 
```yaml 
# RBAC: only ArgoCD repo-server needs to read this secret 
apiVersion: rbac.authorization.k8s.io/v1 
kind: Role 
metadata: 
  name: sops-key-reader 
  namespace: argocd 
rules: 
  - apiGroups: [""] 
    resources: ["secrets"] 
    resourceNames: ["sops-age-key"] 
    verbs: ["get"] 
--- 
apiVersion: rbac.authorization.k8s.io/v1 
kind: RoleBinding 
metadata: 
  name: argocd-repo-server-sops-key 
  namespace: argocd 
subjects: 
  - kind: ServiceAccount 
    name: argocd-repo-server 
    namespace: argocd 
roleRef: 
  kind: Role 
  name: sops-key-reader 
  apiGroup: rbac.authorization.k8s.io 
``` 
 
### Configure ArgoCD to use KSOPS 
 
Patch the ArgoCD repo-server to mount the Age key and install the KSOPS plugin: 
 
```yaml 
# argocd-repo-server patch 
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
            - name: XDG_CONFIG_HOME 
              value: /home/argocd/.config 
          volumeMounts: 
            - mountPath: /home/argocd/.config/sops/age/keys.txt 
              subPath: keys.txt 
              name: sops-age-key 
              readOnly: true 
            - mountPath: /usr/local/bin/ksops 
              subPath: ksops 
              name: custom-tools 
``` 
 
Register the plugin in ArgoCD's ConfigManagementPlugin: 
 
```yaml 
apiVersion: v1 
kind: ConfigMap 
metadata: 
  name: argocd-cmp-plugin 
  namespace: argocd 
data: 
  plugin.yaml: | 
    apiVersion: argoproj.io/v1alpha1 
    kind: ConfigManagementPlugin 
    metadata: 
      name: kustomize-with-ksops 
    spec: 
      generate: 
        command: ["kustomize", "build", "--enable-alpha-plugins", "--enable-exec", "."] 
``` 
 
### Repository structure 
 
``` 
your-repo/ 
├── .sops.yaml 
├── apps/ 
│   └── my-app/ 
│       ├── deployment.yaml 
│       ├── service.yaml 
│       ├── kustomization.yaml 
│       └── secrets/ 
│           ├── generator.yaml      # KSOPS generator config 
│           └── db-credentials.yaml # SOPS encrypted 
``` 
 
```yaml 
# kustomization.yaml 
apiVersion: kustomize.config.k8s.io/v1beta1 
kind: Kustomization 
 
resources: 
  - deployment.yaml 
  - service.yaml 
 
generators: 
  - secrets/generator.yaml 
``` 
 
```yaml 
# secrets/generator.yaml 
apiVersion: viaductoss.github.io/v1alpha1 
kind: ksops 
metadata: 
  name: db-credentials-generator 
  annotations: 
    config.kubernetes.io/function: | 
      exec: 
        path: ksops 
files: 
  - ./db-credentials.yaml 
``` 
 
```yaml 
# secrets/db-credentials.yaml (SOPS encrypted) 
apiVersion: v1 
kind: Secret 
metadata: 
  name: db-credentials 
  namespace: my-app 
type: Opaque 
stringData: 
  password: myS3cur3P@ssword     # this value gets encrypted by SOPS 
  username: app_user              # this value gets encrypted by SOPS 
``` 
 
Encrypt it before committing: 
 
```bash 
sops --encrypt --in-place apps/my-app/secrets/db-credentials.yaml 
git add apps/my-app/secrets/db-credentials.yaml 
git commit -m "add encrypted db credentials" 
``` 
 
ArgoCD picks up the commit, KSOPS decrypts at render time, Kubernetes sees a normal Secret object. The whole flow is GitOps — change a secret by editing the encrypted file, opening a PR, getting it reviewed, and merging. No kubectl apply. No manual steps. 
 
--- 
 
## The Audit Limitation You Need to Know 
 
Git history records who committed a change to an encrypted file and when. It does not record who decrypted and read a value. 
 
If a developer runs `sops -d secrets/db-credentials.yaml` locally, that operation produces no audit trail. If someone exports a decrypted value to their environment and uses it, that is untracked. Git shows the change surface, not the access surface. 
 
**This matters for compliance.** SOC 2, PCI-DSS, and HIPAA typically require audit trails that cover secret reads, not just changes. SOPS does not satisfy that requirement. HashiCorp Vault does — every read is logged with identity, timestamp, and path. 
 
If read-access auditing is a hard requirement, SOPS is the wrong tool. If change auditing is sufficient for your environment, SOPS covers you well. 
 
--- 
 
## Secret Rotation 
 
When you need to rotate a secret value: 
 
```bash 
# Decrypt, edit the value, re-encrypt 
sops secrets/db-credentials.yaml 
# (your editor opens, change the value, save and close) 
# SOPS automatically re-encrypts on save 
 
# Commit the change 
git add secrets/db-credentials.yaml 
git commit -m "rotate database password" 
``` 
 
ArgoCD detects the commit, KSOPS decrypts the new value, Kubernetes updates the Secret. If your application is watching for Secret changes (via a reloader like Reloader or stakater), it picks up the new credential automatically. 
 
For rotating the Age key itself (if the private key is compromised or a team member leaves): 
 
```bash 
# Generate a new Age keypair 
age-keygen -o new-key.txt 
 
# Add new recipient and remove old one, then rotate DEK in all secrets files 
sops --rotate \ 
  --add-age age1newpublickey... \ 
  --rm-age age1oldpublickey... \ 
  --in-place secrets/db-credentials.yaml 
 
# Update the Kubernetes Secret with the new private key 
kubectl create secret generic sops-age-key \ 
  --namespace argocd \ 
  --from-file=keys.txt=new-key.txt \ 
  --dry-run=client -o yaml | kubectl apply -f - 
``` 
 
--- 
 
## Multiple Environments 
 
For teams managing multiple environments, the `.sops.yaml` creation rules let you use different keys per environment: 
 
```yaml 
# .sops.yaml 
creation_rules: 
  - path_regex: envs/dev/.*\.yaml 
    age: age1devpublickey... 
 
  - path_regex: envs/staging/.*\.yaml 
    age: age1stagingpublickey... 
 
  - path_regex: envs/prod/.*\.yaml 
    age: age1prodpublickey... 
    # Optionally add KMS for production 
    kms: arn:aws:kms:us-east-1:123456789012:key/prod-key-id 
``` 
 
Each environment cluster holds only its own Age private key. The production key never exists in dev or staging. A compromised dev cluster cannot decrypt production secrets. 
 
--- 
 
## Common Mistakes 
 
**Committing the private key.** The Age private key should never be in git. The public key is safe to commit and reference in `.sops.yaml`. Only the private key needs protecting. 
 
**Not backing up the private key.** If the Kubernetes Secret holding the Age private key is deleted and you have no backup, you cannot decrypt any of your secrets. Back it up to a password manager or secure offline storage on day one. 
 
**Encrypting non-sensitive values.** SOPS encrypts all string values by default. Use `--encrypted-regex` or the `encrypted_regex` option in `.sops.yaml` to be selective. Encrypting things like `host` and `port` makes PR reviews less useful without adding security. 
 
**Forgetting to re-encrypt after adding a recipient.** If you add a new team member's public key to `.sops.yaml` but don't rotate existing files, those files are still only decryptable by the original recipients. Run `sops --rotate` on existing files to add the new recipient to them. 
 
**Using SOPS for compliance-gated environments without understanding the audit gap.** git history is change history, not access history. Know the difference before choosing the tool. 
 
--- 
 
SOPS with Age solves a specific problem well: storing encrypted secrets in git alongside application manifests, making them reviewable in PRs, and decrypting them automatically at deploy time in a Kubernetes GitOps pipeline. 
 
It does not solve dynamic credential generation, automatic rotation, or read-access auditing. For those requirements, Vault is the right tool. 
 
For teams on ArgoCD or Flux who want secrets in git without ceremony, without extra infrastructure, and without manual kubectl apply steps, SOPS is the most straightforward path to get there. 
 
The setup takes an afternoon. The operational overhead after that is minimal. And your secrets finally live where the rest of your configuration does.