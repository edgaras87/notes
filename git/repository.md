# 📝 Git Local ↔ Remote Cheat Sheet

## 🔑 Core ideas

* **clone** → make a new local folder from remote
* **init** → make current folder a repo
* **remote** = nickname + URL (`origin`)
* **upstream** = default branch link (set with `-u`)

---

## 🚀 Common Scenarios

### 1. Start from Remote → Local

```bash
git clone <REMOTE-URL> [folder]
cd [folder]
```

✅ Now just `git pull` and `git push`

---

### 2. Start from Local → Remote

```bash
git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin <REMOTE-URL>
git push -u origin main
```

---

### 3. Already Have Both (Sync Them)

```bash
git remote -v
git fetch origin
git switch main
git pull --rebase
git push
```

If unrelated histories:

```bash
git pull --rebase --allow-unrelated-histories
```

---

## 📂 Folder Rules

* `git clone <url>` → makes a new folder
* `git clone <url> myfolder` → makes *myfolder*
* `git clone <url> .` → current empty folder
* **Always run Git inside the repo folder**

---

## 🤔 Which First?

* **Solo / quick test:** local → push
* **Team / templates / CI:** remote → clone

---

## ⚡ Handy Commands

```bash
git remote -v                # list remotes
git push -u origin main      # set upstream (first push)
git pull --rebase            # cleaner pulls
git remote set-url origin <url>   # change remote
```

---

## ⚠️ Pitfalls

* Remote has README → `git pull --rebase --allow-unrelated-histories`
* Wrong branch name → `git branch -M main`
* Tracked junk files → fix `.gitignore`, then:

  ```bash
  git rm -r --cached .
  git add .
  git commit -m "clean tracked files"
  ```

---

## 🗂️ Decision Tree

* **Have remote?** → `git clone <url>`
* **Have local only?** → `git init` → `git push -u origin main`
* **Both exist?** → `git fetch` → `git pull --rebase` → `git push`

---

👉 This is the minimal flow you’ll use 99% of the time.

---
