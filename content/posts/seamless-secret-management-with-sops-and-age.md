+++
date = '2026-06-28T16:10:00+07:00'
draft = false
title = 'Seamless Secret Management in GitOps With SOPS and age'
tags = ['sops', 'age', 'secrets', 'gitops', 'encryption', 'devops', 'kubernetes', 'mise', 'security']
+++
GitOps has a famously awkward edge case: you want *everything* in Git, but you can't commit a plaintext database password to a repository. The usual workarounds — a separate secrets manager, a wall of `kubectl create secret` commands, a shared password vault that nobody keeps in sync — all break the "Git is the source of truth" promise.

[SOPS](https://github.com/getsops/sops) (Secrets OPerationS) and [age](https://github.com/FiloSottile/age) solve this neatly. Together they let you commit **encrypted** secrets straight into Git, review them in pull requests, and decrypt them only where and when they're actually needed. The plaintext never touches the repo; the ciphertext lives right next to the code it belongs to.

In this post I'll cover what each tool is, how the encryption actually works under the hood (including the role of public keys), why the combination fits GitOps so well, how it enables team collaboration, a fully runnable example using [`mise`](https://mise.jdx.dev/) to install both tools, and the limitations you should know before adopting it.

## What are SOPS and age?

They solve two different halves of the same problem.

**age** is a modern, opinionated file-encryption tool — think "GPG without the footguns". It has no configuration knobs, no cipher negotiation, and tiny keys. A public key looks like `age1nwhnh2qv6yealq4npum4tlzl0uyev5haa7y355znqjhwuxu8l3qsc4h8mc` and a private key is a single line you can paste anywhere. age encrypts a whole file (or stdin) for one or more recipients. That's it.

**SOPS** is an *editor* for structured secret files — YAML, JSON, ENV, INI, or binary. Instead of encrypting the whole file into an opaque blob, SOPS encrypts only the **values**, leaving keys, structure, and comments readable. It delegates the actual cryptography to a backend: AWS KMS, GCP KMS, Azure Key Vault, HashiCorp Vault, PGP — or **age**.

The combination is powerful precisely because each tool stays in its lane: age provides simple, auditable encryption with portable keys, and SOPS provides a structure-aware, Git-friendly workflow on top of it. No cloud account required.

## How it works internally

This is the part worth understanding, because it explains every design decision that follows.

### age: envelope encryption with X25519

age uses a classic **envelope (hybrid) encryption** scheme. When you encrypt a file for a recipient's public key:

1. age generates a random 128-bit (16-byte) symmetric **file key** for this one file.
2. The file body is encrypted with a **payload key** derived from the file key (via `HKDF-SHA-256`, salted with a random nonce) using **ChaCha20-Poly1305**, an authenticated cipher (confidentiality *and* integrity). The body is split into 64 KiB chunks so it can be streamed.
3. For each recipient, age **wraps** (encrypts) the file key so only that recipient's private key can unwrap it. This wrapped copy is stored in a per-recipient *stanza* in the file header.

The wrapping for an `age1...` recipient uses **X25519**, an Elliptic-Curve Diffie–Hellman function over Curve25519:

* age generates an **ephemeral keypair** just for this encryption.
* It combines the ephemeral private key with the recipient's public key via Diffie–Hellman to derive a shared secret.
* It runs that shared secret through `HKDF-SHA-256` to get a *wrap key*, which encrypts the file key into the recipient's stanza. The stanza also stores the ephemeral *public* key.

To decrypt, the recipient combines their **private** key with the stored ephemeral public key, derives the *same* shared secret, unwraps the file key, and decrypts the body. The math of Diffie–Hellman guarantees both sides arrive at the same secret without it ever crossing the wire.

The key insight: **the public key only lets you wrap (encrypt) the file key; it cannot unwrap it.** So a public key is safe to share, commit, and put in a PR. You can encrypt *for* someone without being able to decrypt what you just wrote — and that's exactly what makes multi-recipient, GitOps-friendly secrets possible. Because the file key is wrapped once per recipient, encrypting for ten teammates just means ten small stanzas in front of one shared ciphertext body.

An age file header is plainly visible:

```text
-----BEGIN AGE ENCRYPTED FILE-----
-> X25519 kR7t8qV5zwYtVRhb...       # ephemeral pubkey + wrapped file key (recipient 1)
-> X25519 9aB2cD...                 # recipient 2
--- <header MAC, HMAC-SHA-256 over the header>
<ciphertext body, ChaCha20-Poly1305 in 64 KiB chunks>
-----END AGE ENCRYPTED FILE-----
```

### SOPS: a data key on top of age

SOPS adds one more layer so it can encrypt values *individually* while still only doing public-key crypto once:

1. SOPS generates a random **data key** (an AES-256 key) for the file.
2. It encrypts **each secret value** in place with that data key using **AES256-GCM** — so `password: hunter2` becomes `password: ENC[AES256_GCM,data:...,iv:...,tag:...,type:str]`. Keys and structure stay in cleartext, which is what makes diffs reviewable.
3. The data key itself is then encrypted **for every configured recipient**. With the age backend, that means handing the data key to age, which wraps it for each `age1...` public key and stores the resulting age blob in the file's `sops:` metadata block.
4. SOPS computes a **MAC** — a `SHA-512` hash over all the plaintext values — and stores it *encrypted with the data key* (AES256-GCM), which is the `mac: ENC[AES256_GCM,...]` field you'll see in the file. On decryption SOPS recomputes the hash over the decrypted values and compares; if they differ (e.g. someone added, removed, or swapped a ciphertext value), decryption fails.

So there are two nested envelopes: **age wraps the SOPS data key** (per recipient, via X25519), and **the data key encrypts each value** (via AES-GCM). To decrypt, SOPS asks age to unwrap the data key with your private key, then decrypts every `ENC[...]` value and verifies the MAC.

You'll see this structure directly in an encrypted file's footer:

```yaml
sops:
    age:
        - enc: |
            -----BEGIN AGE ENCRYPTED FILE-----   # the data key, wrapped by age
            ...
            -----END AGE ENCRYPTED FILE-----
          recipient: age1nwhnh2qv6yealq4npum4tlzl0uyev5haa7y355znqjhwuxu8l3qsc4h8mc
    encrypted_regex: ^(data|stringData)$
    mac: ENC[AES256_GCM,data:...]                 # integrity check
    version: 3.13.1
```

## Why this combination, and common use cases

A few properties fall out of the design above:

* **Secrets live in Git.** The ciphertext is committed alongside the manifests it configures. One repo, one source of truth — the GitOps ideal.
* **Diffs stay meaningful.** Because only values are encrypted, a PR shows *which* keys changed, even if it can't show the new plaintext. Reviewers see structure and intent.
* **No central secret server required.** Decryption needs only a private key file. Great for homelabs, edge, air-gapped, or "I don't want to pay for a KMS" setups. (You *can* still use KMS — SOPS supports mixing backends.)
* **Asymmetric trust.** Anyone with the public key can add or update secrets; only holders of a private key can read them. CI can encrypt without being able to decrypt.

Typical use cases:

* **Kubernetes secrets in GitOps.** Commit encrypted `Secret` manifests and let [Flux's SOPS integration](https://fluxcd.io/flux/guides/mozilla-sops/) or the [SOPS operator / `ksops` for Argo CD](https://argo-cd.readthedocs.io/en/stable/operator-manual/secret-management/) decrypt them in-cluster using a private key stored once as a cluster secret.
* **App config / `.env` files.** Encrypt `config.prod.yaml` or `.env.production` and decrypt at deploy time.
* **Terraform / Ansible variables.** Keep `secrets.auto.tfvars` or Ansible vault-style data encrypted in the repo.
* **CI/CD pipelines.** Store the age private key as a single CI secret; pipelines decrypt everything else from the repo on demand.

## How it enables team collaboration

This is age's quiet superpower. Because you encrypt for a *list* of public keys, onboarding a teammate is a metadata change, not a secret re-share.

1. Each engineer (and each environment, and CI) generates their own age keypair and **publishes only the public key** — in the repo, a wiki, or chat. Private keys never leave their owner's machine.
2. A `.sops.yaml` file in the repo lists which public keys may decrypt which paths.
3. To add a new member, you append their public key to `.sops.yaml` and run `sops updatekeys` on the affected files. SOPS unwraps the data key, re-wraps it for the new recipient list, and writes the file back — **without ever exposing the plaintext values**. The change is a reviewable diff in the `sops:` block.
4. To off-board someone, remove their key and run `updatekeys` again, then rotate the underlying secrets.

You can even encrypt to *different* recipient sets per path — say, dev secrets readable by the whole team but prod secrets restricted to the CI key and two leads — all declared in one `.sops.yaml`:

```yaml
creation_rules:
  - path_regex: secrets/dev/.*\.ya?ml$
    age: "age1dev...,age1alice...,age1bob..."
  - path_regex: secrets/prod/.*\.ya?ml$
    age: "age1ci...,age1lead..."
```

No shared master password, no "DM me the prod creds", no secret that's only in one person's head.

## A runnable example

Let's encrypt a Kubernetes `Secret` end to end. We'll use [`mise`](https://mise.jdx.dev/) to install `sops` and `age` so the versions are pinned and reproducible. (New to `mise`? See my [getting started post](../getting-started-with-mise/).)

### 1. Create the project and install the tools

```bash
mkdir sops-age-demo && cd sops-age-demo

# install both tools, pinned in mise.toml
mise use sops@latest age@latest
```

This writes a `mise.toml` and installs the binaries:

```toml
# mise.toml
[tools]
age = "latest"
sops = "latest"
```

Confirm they're active:

```bash
mise exec -- sops --version    # sops 3.13.1 (latest)
mise exec -- age --version     # v1.3.1
```

> Pin real versions (e.g. `sops@3.13.1`, `age@1.3.1`) and commit a `mise.lock` if you want byte-for-byte reproducibility — see the [mise lockfile docs](https://mise.jdx.dev/configuration/settings.html#lockfile).

### 2. Generate an age keypair

```bash
mise exec -- age-keygen -o keys.txt
# Public key: age1nwhnh2qv6yealq4npum4tlzl0uyev5haa7y355znqjhwuxu8l3qsc4h8mc
```

`keys.txt` holds your **private** key — never commit it. The public key is printed and also stored as a comment inside the file. You can re-derive the public key from the private file at any time:

```bash
mise exec -- age-keygen -y keys.txt
# age1nwhnh2qv6yealq4npum4tlzl0uyev5haa7y355znqjhwuxu8l3qsc4h8mc
```

Tell SOPS where your private key lives (SOPS reads this env var when decrypting):

```bash
export SOPS_AGE_KEY_FILE=$PWD/keys.txt
```

### 3. Declare encryption rules in `.sops.yaml`

Put your **public** key here. The `encrypted_regex` tells SOPS to only encrypt values under `data`/`stringData`, leaving the rest of the manifest readable:

```yaml
# .sops.yaml
creation_rules:
  - path_regex: secrets/.*\.ya?ml$
    encrypted_regex: '^(data|stringData)$'
    age: >-
      age1nwhnh2qv6yealq4npum4tlzl0uyev5haa7y355znqjhwuxu8l3qsc4h8mc
```

### 4. Write and encrypt a secret

```bash
mkdir -p secrets
cat > secrets/db.yaml <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
stringData:
  username: app
  password: s3cr3t-p@ssw0rd
EOF

mise exec -- sops encrypt --in-place secrets/db.yaml
```

The result is safe to commit. Note that `metadata` and the keys stay readable — only the values are `ENC[...]`:

```yaml
apiVersion: v1
kind: Secret
metadata:
    name: db-credentials
stringData:
    username: ENC[AES256_GCM,data:/UNE,iv:JpHY...,tag:pEdX...,type:str]
    password: ENC[AES256_GCM,data:lk6v7...,iv:NflK...,tag:d9sz...,type:str]
sops:
    age:
        - enc: |
            -----BEGIN AGE ENCRYPTED FILE-----
            ...the data key, wrapped for your public key...
            -----END AGE ENCRYPTED FILE-----
          recipient: age1nwhnh2qv6yealq4npum4tlzl0uyev5haa7y355znqjhwuxu8l3qsc4h8mc
    encrypted_regex: ^(data|stringData)$
    mac: ENC[AES256_GCM,data:...]
    version: 3.13.1
```

### 5. Decrypt, edit, and extract

Decrypt the whole file (needs your private key via `SOPS_AGE_KEY_FILE`):

```bash
mise exec -- sops decrypt secrets/db.yaml
# ...stringData.password: s3cr3t-p@ssw0rd
```

Edit in place — SOPS decrypts into a temp editor session and re-encrypts on save:

```bash
mise exec -- sops edit secrets/db.yaml
```

Pull out a single value (handy in deploy scripts):

```bash
mise exec -- sops decrypt --extract '["stringData"]["password"]' secrets/db.yaml
# s3cr3t-p@ssw0rd
```

Pipe a decrypted manifest straight to your cluster, so plaintext never lands on disk:

```bash
mise exec -- sops decrypt secrets/db.yaml | kubectl apply -f -
```

### 6. Add a teammate (key rotation)

Append a second public key to the `age:` line in `.sops.yaml`, then re-wrap the data key for the new recipient list — no plaintext exposure:

```bash
mise exec -- sops updatekeys secrets/db.yaml
```

The only change in the diff is a new stanza in the `sops:` block. The teammate can now decrypt with *their* private key, and the original `ENC[...]` values are untouched.

### 7. What to commit (and what not to)

```bash
echo 'keys.txt' >> .gitignore   # NEVER commit private keys

git add mise.toml .sops.yaml secrets/db.yaml .gitignore
git commit -m "Add encrypted db credentials"
```

Commit the encrypted secret, the `.sops.yaml`, and `mise.toml`. Keep `keys.txt` out of Git — distribute private keys out-of-band (a password manager, your CI's secret store, or a sealed cluster secret).

### 8. Clean up (optional)

```bash
cd .. && rm -rf sops-age-demo
```

## Limitations and gotchas

No tool is free of trade-offs. Know these before you commit:

* **Key distribution is on you.** age has no PKI, no web of trust, no revocation lists. Getting private keys safely onto machines/CI and proving a public key really belongs to a teammate is a manual, out-of-band process.
* **Removing a recipient doesn't un-leak a secret.** `sops updatekeys` stops *future* access, but anyone who already decrypted (or kept an old Git revision) still has the plaintext. Off-boarding means **rotating the actual secret values**, not just the keys.
* **Git history is forever.** If you ever commit a plaintext secret by mistake, it lives in history until you rewrite it. Encrypt *before* the first commit.
* **Metadata isn't secret.** Keys, structure, and comments stay in cleartext by design. Don't put sensitive data in a *key name* like `aws_root_password_for_acme_corp`. The value lengths are also roughly observable.
* **Per-value encryption only suits structured files.** SOPS shines on YAML/JSON/ENV. For arbitrary binaries it falls back to whole-file encryption, losing the nice diffs — at which point plain `age` may be simpler.
* **Decryption needs the private key present at runtime.** In Kubernetes that usually means storing the age key as an in-cluster secret for Flux/Argo to use — so your cluster's secret store becomes the new root of trust you must protect.
* **No built-in auditing or rotation policy.** Unlike a managed KMS, there's no access log, automatic rotation, or fine-grained IAM. You build process around it yourself. For high-compliance environments, SOPS can target KMS backends instead of (or alongside) age.
* **Whitespace/formatting churn.** SOPS rewrites files on encrypt and may normalize formatting, which can produce noisier diffs than expected.

None of these are dealbreakers — they're the cost of a serverless, Git-native model. For most teams, homelabs, and GitOps setups, the simplicity is well worth it.

## Wrapping up

SOPS + age turns "secrets in Git" from an oxymoron into a clean workflow. age gives you small, portable keys and a dead-simple envelope-encryption scheme where public keys safely wrap a file key for any number of recipients; SOPS layers a structure-aware, per-value editor on top so your encrypted secrets diff and review like normal config. Pin both with `mise`, declare recipients in `.sops.yaml`, and your secrets become just another reviewable, version-controlled, GitOps-friendly file — with the plaintext staying exactly where it belongs: nowhere near the repo.

## Useful links

* [SOPS — getsops/sops](https://github.com/getsops/sops)
* [age — FiloSottile/age](https://github.com/FiloSottile/age)
* [age design & specification](https://age-encryption.org/v1)
* [Flux: Manage Kubernetes secrets with SOPS](https://fluxcd.io/flux/guides/mozilla-sops/)
* [Argo CD secret management](https://argo-cd.readthedocs.io/en/stable/operator-manual/secret-management/)
* [Getting started with mise](../getting-started-with-mise/)
