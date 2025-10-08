# git reset — undo last commit locally and on remote

---

## 1) Quick summary — undo **last commit** locally and on remote

* If you want to **keep the work (uncommit)** but remove the commit:

  ```
  # keep changes staged
  git reset --soft HEAD~1

  # or keep changes unstaged
  git reset --mixed HEAD~1   # same as `git reset HEAD~1`
  ```
* If you want to **discard the commit and its changes**:

  ```
  git reset --hard HEAD~1
  ```
* Update the remote to match your local (force rewrite remote branch):

  ```
  # safer force (recommended even when alone)
  git push --force-with-lease origin <your-branch>

  # or force (less safe)
  git push --force origin <your-branch>
  ```

Notes:

* `--force-with-lease` prevents clobbering a remote you don’t expect — it fails if remote changed since you last fetched.
* If you accidentally delete something, `git reflog` can usually recover the commit SHA.


## Create a PAT (Personal Access Token) via GitHub web UI

1. Sign into GitHub in your browser.
2. Go to **Settings → Developer settings → Personal access tokens** (or **Tokens (classic)** / **Fine-grained tokens**) and click **Generate new token** (choose classic or fine-grained per your needs).
3. Give it a name, expiry, and select scopes — for normal repo push/pull choose `repo` (or more narrow scopes for fine-grained tokens).
4. Click **Generate** and **copy the token now** — you won't be able to view it again.
