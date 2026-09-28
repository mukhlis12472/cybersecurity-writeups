# 🔍 The Careless Developer

## Challenge Information

| Field | Details |
|---|---|
| **Challenge Name** | The Careless Developer |
| **Category** | OSINT / Git / Encoding |
| **Points** | 150 |
| **Solves** | 19 |
| **Status** | ✅ Solved |

---

## 📖 Challenge Description

> Gatnexa-Systems is a small logistics startup building an internal inventory management tool. Their lead developer, Akmal Tang, has been moving fast maybe a little too fast.
>
> Rumor has it something sensitive slipped into a public repo at some point... and even though it was "cleaned up," the internet never really forgets.
>
> Find the leak. Recover the secret. Decode what's hidden inside.

### Hint

> **"It is there, but it isn't."**

---

## 🧠 Initial Analysis

The challenge description provides several useful clues:

- Company: `Gatnexa-Systems`
- Developer: `Akmal Tang`
- A **public repository** is involved
- Sensitive information was accidentally committed
- The information was later **"cleaned up"**
- The recovered secret needs to be **decoded**

The most important clue was:

> **"even though it was cleaned up, the internet never really forgets."**

This suggested that the sensitive information was no longer available in the current version of the repository, but might still exist inside the **Git commit history**.

---

# 🔎 Step 1 — Finding the Repository

Using the information provided by the challenge, I searched for the developer/company and eventually discovered a public repository named:

```text
Inventory-API
```

The repository belonged to the developer mentioned in the challenge:

```text
Akmal Tang
```

Looking through the current repository did not immediately reveal a flag or secret.

The repository contained files and directories such as:

```text
middleware/
models/
routes/
tests/
.eslintrc.json
.gitignore
README.md
package.json
server.js
```

Nothing particularly sensitive was visible in the latest version.

However, the repository had:

```text
22 Commits
```

This made the commit history the next obvious place to investigate.

---

# 🕰️ Step 2 — Investigating the Commit History

I opened the repository's **Commits** page and reviewed the previous commits.

Several commit messages immediately stood out:

```text
Remove github config, moved to secrets manager
Add github actions config for CI
Remove staging config, no longer needed
Add staging secrets config
Remove env file, add to gitignore
Add local env config for testing
```

The most interesting pair was:

```text
Add local env config for testing
```

followed later by:

```text
Remove env file, add to gitignore
```

The commit that added the environment configuration had the hash:

```text
61116f6
```

This was suspicious because `.env` files commonly contain sensitive information such as:

```text
Database usernames
Database passwords
API keys
Tokens
Secrets
```

The developer had removed the `.env` file later, but Git still retained the older version.

This perfectly matched the hint:

> **"It is there, but it isn't."**

---

# 🔬 Step 3 — Inspecting the Old Commit

I opened commit:

```text
61116f6
```

with the commit message:

```text
Add local env config for testing
```

The commit showed that a new `.env` file had been added.

Inside the file were the following values:

```env
NODE_ENV=development
PORT=3000
DB_HOST=localhost
DB_USER=admin
DB_PASS=temp_pass_123
API_KEY=VGtGRVNYdERlV0l6Y2xNelkzVnlNWFI1WDBrMVgwWlZUbjA9Cg==
```

The most suspicious value was:

```text
API_KEY=VGtGRVNYdERlV0l6Y2xNelkzVnlNWFI1WDBrMVgwWlZUbjA9Cg==
```

The `==` padding at the end strongly suggested that the value was **Base64 encoded**.

---

# 🔐 Step 4 — Recovering the Secret

The leaked API key was:

```text
VGtGRVNYdERlV0l6Y2xNelkzVnlNWFI1WDBrMVgwWlZUbjA9Cg==
```

I copied the value without:

```text
API_KEY=
```

and attempted to decode it using Base64.

This can be done using **CyberChef**, Linux, Python, or another Base64 decoder.

---

# 🧩 Step 5 — First Base64 Decode

Using CyberChef:

```text
From Base64
```

The first decoding operation produced:

```text
TkFESXtDeWIzclMzY3VyMXR5X0k1X0ZVTn0=
```

This output also looked like Base64.

Therefore, the secret had been encoded **twice**.

The process at this point was:

```text
Original API_KEY
       ↓
From Base64
       ↓
Another Base64 string
```

---

# 🧩 Step 6 — Second Base64 Decode

I decoded the result again using:

```text
From Base64
```

So:

```text
TkFESXtDeWIzclMzY3VyMXR5X0k1X0ZVTn0=
```

became:

```text
NADI{Cyb3rS3cur1ty_I5_FUN}
```

The second decoding revealed the final flag.

---

# 🚩 Flag

```text
NADI{Cyb3rS3cur1ty_I5_FUN}
```

---

# 🧑‍💻 Alternative Method — Linux Terminal

The same process can be performed without CyberChef.

First decode:

```bash
echo "VGtGRVNYdERlV0l6Y2xNelkzVnlNWFI1WDBrMVgwWlZUbjA9Cg==" | base64 -d
```

Output:

```text
TkFESXtDeWIzclMzY3VyMXR5X0k1X0ZVTn0=
```

Decode it again:

```bash
echo "TkFESXtDeWIzclMzY3VyMXR5X0k1X0ZVTn0=" | base64 -d
```

Output:

```text
NADI{Cyb3rS3cur1ty_I5_FUN}
```

---

# 🐍 Alternative Method — Python

The flag can also be recovered using Python:

```python
import base64

encoded = "VGtGRVNYdERlV0l6Y2xNelkzVnlNWFI1WDBrMVgwWlZUbjA9Cg=="

first_decode = base64.b64decode(encoded).decode().strip()
print(first_decode)

second_decode = base64.b64decode(first_decode).decode()
print(second_decode)
```

Output:

```text
TkFESXtDeWIzclMzY3VyMXR5X0k1X0ZVTn0=
NADI{Cyb3rS3cur1ty_I5_FUN}
```

---

# 🔄 Attack Path

The complete solution path was:

```text
Challenge Description
        │
        ▼
Identify Gatnexa-Systems / Akmal Tang
        │
        ▼
Find Inventory-API Repository
        │
        ▼
Inspect Current Repository
        │
        ▼
No Secret Found
        │
        ▼
Check Commit History
        │
        ▼
Find "Add local env config for testing"
        │
        ▼
Open Commit 61116f6
        │
        ▼
Discover .env File
        │
        ▼
Recover API_KEY
        │
        ▼
Base64 Decode
        │
        ▼
Base64 Decode Again
        │
        ▼
🚩 NADI{Cyb3rS3cur1ty_I5_FUN}
```

---

# 💡 Why the Vulnerability Exists

The developer accidentally committed a `.env` file containing sensitive information.

Later, the developer attempted to fix the mistake with the commit:

```text
Remove env file, add to gitignore
```

However, deleting a file in a newer Git commit does **not** remove the file from previous commits.

Conceptually:

```text
Commit 1
│
├── Application files
│
▼
Commit 61116f6
│
├── .env
│   ├── DB_USER=admin
│   ├── DB_PASS=temp_pass_123
│   └── API_KEY=<secret>
│
▼
Later Commit
│
├── .env deleted
│
▼
Current Repository
    └── .env no longer visible
```

The current repository therefore looks clean, but the sensitive information still exists inside its history.

That is why the hint says:

> **"It is there, but it isn't."**

---

# 🛡️ Security Lessons

This challenge demonstrates an important Git security issue.

Sensitive files such as:

```text
.env
credentials.json
config.json
*.pem
*.key
private keys
API tokens
database passwords
cloud credentials
```

should never be committed to a public repository.

A `.gitignore` file should be configured before committing:

```gitignore
.env
.env.*
*.pem
*.key
credentials.json
secrets.json
node_modules/
```

More importantly, if a real credential has already been committed publicly, simply deleting the file is **not sufficient**.

The credential should be:

1. **Revoked or rotated immediately**
2. Removed from the application
3. Removed from Git history where appropriate
4. Replaced with a properly managed secret
5. Stored using environment variables or a secrets manager

Once a credential has been exposed publicly, it should be treated as compromised.

---

# 📝 Conclusion

**The Careless Developer** was an OSINT and Git-history challenge that demonstrated how sensitive information can remain accessible even after a developer deletes it from a repository.

The key breakthrough was examining the commit:

```text
61116f6 — Add local env config for testing
```

which exposed a `.env` file containing:

```text
DB_USER=admin
DB_PASS=temp_pass_123
API_KEY=VGtGRVNYdERlV0l6Y2xNelkzVnlNWFI1WDBrMVgwWlZUbjA9Cg==
```

The API key was Base64 encoded twice.

```text
VGtGRVNYdERlV0l6Y2xNelkzVnlNWFI1WDBrMVgwWlZUbjA9Cg==
                            ↓
                      Base64 Decode
                            ↓
TkFESXtDeWIzclMzY3VyMXR5X0k1X0ZVTn0=
                            ↓
                      Base64 Decode
                            ↓
              NADI{Cyb3rS3cur1ty_I5_FUN}
```

Therefore, the final flag was:

```text
NADI{Cyb3rS3cur1ty_I5_FUN}
```

---

## 🛠️ Tools Used

- GitHub
- Git commit history
- OSINT
- CyberChef
- Base64
- Linux terminal / Python

---

## 🎯 What I Learned

From this challenge, I learned how to:

- Perform basic GitHub OSINT
- Investigate a developer's public repository
- Analyse Git commit history
- Identify suspicious commit messages
- Recover deleted files from previous commits
- Find exposed `.env` files
- Identify Base64 encoded data
- Handle multiple layers of encoding
- Understand why deleting secrets from Git is not enough

---

## 🚩 Final Flag

```text
NADI{Cyb3rS3cur1ty_I5_FUN}
```

---

**Challenge:** The Careless Developer  
**Points:** 150  
**Category:** OSINT / Git / Encoding  
**Status:** ✅ Solved
