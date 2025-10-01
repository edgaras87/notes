# 🐧 Fedora DNF Cheat Sheet

## 📦 Installing & Removing

```bash
sudo dnf install <pkg>       # install package
sudo dnf remove <pkg>        # uninstall package
```

* `-y` → auto-confirm yes

  ```bash
  sudo dnf install -y <pkg>
  ```

## 🔄 Updating

```bash
sudo dnf upgrade             # upgrade all packages
sudo dnf upgrade <pkg>       # upgrade single package
```

* `--refresh` → ignore old cache, fetch fresh repo data

## 🔎 Searching & Info

```bash
dnf search <keyword>         # search packages
dnf info <pkg>               # details about package
```

## 🗂 Repositories

```bash
dnf repolist                 # list enabled repos
dnf repolist all             # list all repos
```

### Manage repos with `config-manager` (from `dnf-plugins-core`):

```bash
sudo dnf config-manager --add-repo <url>    # add new repo
sudo dnf config-manager --set-enabled <repoid>
sudo dnf config-manager --set-disabled <repoid>
```

### 📝 Example

Let’s say you want to add the **Docker CE repo** (so you can install Docker directly from Docker’s official packages).

1. Add the repo:

   ```bash
   sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
   ```

2. Then install Docker:

   ```bash
   sudo dnf install docker-ce docker-ce-cli containerd.io
   ```

👉 Different vendors (like Oracle, Microsoft, Google) give you a similar `.repo` URL or a `.repo` file to place in `/etc/yum.repos.d/`. Once added, you can install their packages with normal `dnf install`.

---

## 🧹 Cleaning Cache

```bash
sudo dnf clean all     # clear ALL cached metadata & packages
sudo dnf makecache     # rebuild cache with fresh repo metadata
```

* Use `clean all` if cache is outdated/corrupted or you want to free space.
* Use `makecache` to pre-load fresh repo info for faster installs/search.

---

## 🕵️ Useful Extras

```bash
dnf list installed     # show installed packages
dnf list available     # show available packages
dnf list updates       # show upgradable packages

dnf provides /path/to/file   # find which package owns a file
```

---

## 🛠 If Package is Missing

1. **Flatpak (universal packages):**

   ```bash
   flatpak install flathub <app-id>
   ```

   Example:

   ```bash
   flatpak install flathub com.mysql.Workbench
   ```

2. **Vendor repo:** add vendor’s `.repo` (with `config-manager` or manual file in `/etc/yum.repos.d/`), then install via `dnf install`.

---

## ✅ Quick Survival Rules

* Try **`dnf` first** (Fedora repos).
* If not there → use **Flatpak** (Flathub).
* Still missing → add **vendor repo**.
* If things act weird → `sudo dnf clean all && sudo dnf makecache`.







---

# 📚 Understanding `dnf config-manager`

## 🔧 What is `dnf config-manager`?

* `config-manager` is a **subcommand (plugin)** of `dnf`.
* It’s not installed by default on minimal Fedora systems, but it comes with the package **`dnf-plugins-core`**.
  If you don’t have it, install it:

  ```bash
  sudo dnf install dnf-plugins-core
  ```

Once installed, you get extra commands like `dnf config-manager`.

---

## 🏷️ What does it do?

It’s used to **manage DNF/YUM repo configurations**.
Think of it as a helper to enable, disable, or add new repositories without manually editing files.

---

## 🚩 Common Flags (Options)

* **`--add-repo <url>`** → Add a new repository (from a `.repo` file or repo URL).
* **`--set-enabled <repoid>`** → Enable a repo that exists but is disabled.
* **`--set-disabled <repoid>`** → Disable a repo temporarily or permanently.

### Example:

Enable the `fedora-modular` repo:

```bash
sudo dnf config-manager --set-enabled fedora-modular
```

Disable it:

```bash
sudo dnf config-manager --set-disabled fedora-modular
```

---

## 🗂 Manual Alternative

Without `config-manager`, you could manually place `.repo` files into:

```
/etc/yum.repos.d/
```

Each `.repo` file is just a config text file telling DNF where to fetch packages.
Example file `/etc/yum.repos.d/docker.repo`:

```ini
[docker-ce-stable]
name=Docker CE Stable - $basearch
baseurl=https://download.docker.com/linux/fedora/$releasever/$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://download.docker.com/linux/fedora/gpg
```

---

✅ **Summary:**

* `dnf config-manager` = a helper command from `dnf-plugins-core`.
* `--add-repo` = quick way to add new repos.
* Other useful flags: `--set-enabled`, `--set-disabled`.
* If you don’t want to use it, you can always just drop a `.repo` file into `/etc/yum.repos.d/`.


