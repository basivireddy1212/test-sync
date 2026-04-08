Using a **GitHub App token in GitHub Actions** is the best practice for secure automation (better than PATs). Here’s a **clear, production-ready setup** for GitHub.

---

# 🚀 1. Create & Configure GitHub App

Go to:

```
https://github.com/settings/apps
```

### Required settings:

* ✅ Repository permissions:

  * Contents → **Read & Write**
  * Metadata → **Read**
* ✅ Install App:

  * Install on your account/org
  * Select repo(s) OR “All repositories”

---

# 🔐 2. Store Secrets in Repo

Go to:

```
Repo → Settings → Secrets → Actions
```

Add:

| Secret Name       | Value                           |
| ----------------- | ------------------------------- |
| `APP_ID`          | GitHub App ID                   |
| `APP_PRIVATE_KEY` | Private key (.pem file content) |

👉 Paste full `.pem` content:

```
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
```

---

# ⚙️ 3. GitHub Actions Workflow (Working Example)

```yaml
name: Use GitHub App Token

on:
  workflow_dispatch:

jobs:
  app-auth:
    runs-on: ubuntu-latest

    steps:
      - name: Generate GitHub App Token
        id: app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          owner: basivireddy1212
          repositories: test-sync

      - name: Checkout using App Token
        uses: actions/checkout@v4
        with:
          token: ${{ steps.app-token.outputs.token }}

      - name: Test API
        run: |
          curl -H "Authorization: Bearer ${{ steps.app-token.outputs.token }}" \
          https://api.github.com/repos/basivireddy1212/test-sync
```

---

# ✅ 4. Use Token for Git Operations

### Clone / Push

```bash
git clone https://x-access-token:${TOKEN}@github.com/OWNER/REPO.git
```

### Example in workflow:

```yaml
- name: Push changes
  run: |
    git config user.name "github-app[bot]"
    git config user.email "github-app[bot]@users.noreply.github.com"
    
    echo "test" >> file.txt
    git add .
    git commit -m "Update via GitHub App"
    
    git push https://x-access-token:${{ steps.app-token.outputs.token }}@github.com/basivireddy1212/test-sync.git
```

---

# 🔁 5. Multi-Repo Access (Important)

If syncing across repos:

```yaml
with:
  owner: basivireddy1212
  repositories: repo1,repo2,repo3
```

---

# ⚠️ Common Errors & Fixes

### ❌ `Integration not found`

👉 Fix:

* App not installed on repo
* Repo not selected during install

---

### ❌ `Resource not accessible by integration`

👉 Fix:

* Missing permissions (e.g., Contents: Write)

---

### ❌ 404 on `/installation`

👉 Fix:

* Wrong owner/repo
* App installed in different account

---

# 🔒 Why GitHub App Token is Better

| Feature                  | GitHub App | PAT |
| ------------------------ | ---------- | --- |
| Scoped per repo          | ✅          | ❌   |
| Short-lived token        | ✅ (1 hr)   | ❌   |
| Secure for CI/CD         | ✅          | ⚠️  |
| Fine-grained permissions | ✅          | ❌   |

---

# 💡 Pro Tips (DevOps Level)

### ✔ Always specify owner + repositories

Avoid auto-detection issues:

```yaml
owner: basivireddy1212
repositories: test-sync
```

---

### ✔ Use bot identity

```bash
github-app[bot]
```

---

### ✔ Rotate keys periodically

Regenerate private key if exposed

---

# 🧠 Key Takeaway

> GitHub App token works **only if the app is installed on the target repo**
> → Your previous error was exactly this issue

---

If you want, I can:

* 🔁 Build **multi-repo sync workflow (like repo-file-sync-action but with App token + signed commits)**
* 🔐 Add **GPG commit signing using GitHub App**
* 🏗️ Provide **enterprise-grade reusable workflow template**



Adding **GPG commit signing with a GitHub App token in GitHub Actions** is doable—but there’s one important reality:

> 🚨 **GitHub Apps do NOT sign commits themselves**
> You must still use a **GPG key (or SSH signing key)** inside the workflow.

So the pattern is:
👉 GitHub App token = authentication
👉 GPG key = commit signing

---

# 🚀 Architecture Overview

* 🔐 GitHub App → generates short-lived token
* 🔑 GPG private key → used to sign commits
* 🤖 Bot identity → mapped to verified signature in GitHub

---

# ✅ Step 1: Generate GPG Key (one-time)

Run locally:

```bash
gpg --full-generate-key
```

Choose:

* RSA 4096
* No expiry (or your policy)

List keys:

```bash
gpg --list-secret-keys --keyid-format=long
```

Example output:

```bash
sec   rsa4096/ABC1234567890DEF
```

👉 Your **KEY_ID = `ABC1234567890DEF`**

---

# ✅ Step 2: Export Private Key

```bash
gpg --export-secret-keys --armor ABC1234567890DEF > private.key
```

---

# ✅ Step 3: Add Secrets in GitHub

Repo → Settings → Secrets → Actions

| Secret            | Value                     |
| ----------------- | ------------------------- |
| `APP_ID`          | GitHub App ID             |
| `APP_PRIVATE_KEY` | App private key           |
| `GPG_PRIVATE_KEY` | contents of `private.key` |
| `GPG_PASSPHRASE`  | passphrase                |
| `GPG_KEY_ID`      | key ID                    |

---

# ⚙️ Step 4: Workflow (FULL WORKING)

```yaml
name: Signed Commit with GitHub App

on:
  workflow_dispatch:

jobs:
  signed-commit:
    runs-on: ubuntu-latest

    steps:
      # 🔐 Generate GitHub App token
      - name: Generate App Token
        id: app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}

      # 📥 Checkout using App token
      - uses: actions/checkout@v4
        with:
          token: ${{ steps.app-token.outputs.token }}

      # 🔑 Import GPG key
      - name: Import GPG key
        run: |
          echo "${{ secrets.GPG_PRIVATE_KEY }}" | gpg --batch --import

      # 🔧 Configure Git for signing
      - name: Configure Git
        run: |
          git config --global user.name "github-app[bot]"
          git config --global user.email "github-app[bot]@users.noreply.github.com"
          
          git config --global user.signingkey ${{ secrets.GPG_KEY_ID }}
          git config --global commit.gpgsign true

      # 🔓 Allow GPG in CI
      - name: Configure GPG TTY
        run: |
          export GPG_TTY=$(tty)

      # ✍️ Create signed commit
      - name: Commit & Push (signed)
        run: |
          echo "Signed commit test $(date)" >> signed.txt
          git add signed.txt
          
          git commit -S -m "Signed commit via GitHub App"
          
          git push https://x-access-token:${{ steps.app-token.outputs.token }}@github.com/basivireddy1212/test-sync.git
```

---

# ✅ Step 5: Add GPG Key to GitHub

Go to:

```text
GitHub → Settings → SSH & GPG Keys
```

Add:
👉 **Public key**

```bash
gpg --armor --export ABC1234567890DEF
```

---

# 🎯 Result

* ✅ Commit is pushed using GitHub App token
* ✅ Commit is **GPG signed**
* ✅ GitHub shows:
  ✔ “Verified” badge

---

# ⚠️ Common Issues

### ❌ “Unverified signature”

👉 Fix:

* Email in GPG key must match Git config
* Public key must be added to GitHub

---

### ❌ `gpg: signing failed`

👉 Fix:

```bash
export GPG_TTY=$(tty)
```

---

### ❌ Bot vs User mismatch

If using:

```bash
github-app[bot]
```

👉 Ensure:

* Email matches GPG key identity

---

# 💡 Pro Alternative (Better in 2025+)

Instead of GPG, use **SSH commit signing**:

```bash
git config --global gpg.format ssh
```

✔ Simpler
✔ No passphrase issues
✔ Fully supported by GitHub

---

# 🧠 Key Insight

> GitHub App = authentication
> GPG = identity verification

They work **together**, not as a replacement.

---

If you want next level:

* 🔁 Multi-repo sync with **signed commits + GitHub App**
* 🔐 Org-wide enforcement: “require signed commits”
* ⚡ Reusable workflow (drop-in for all repos)

Just tell me 👍
