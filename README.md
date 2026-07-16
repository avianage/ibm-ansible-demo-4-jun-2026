# ibm-ansible-demo

A CI/CD pipeline demo where **Ansible** — not raw `kubectl` — drives the Kubernetes deployment step. Jenkins builds and pushes a Docker image, then hands off to an Ansible playbook to deploy it.

This README assumes a **Windows 11 laptop** with **Docker Desktop** already installed, and walks through everything else needed from a clean machine.

---

## Part 1 — One-time machine setup

Do this once per laptop. If you already completed this setup for another demo (e.g. the Ansible hands-on lab), you can skip to [Part 2](#part-2--project-setup).

### 1.1 Install a real Ubuntu distro in WSL2

**Important:** Docker Desktop installs its own internal Linux distro called `docker-desktop`, visible if you run `wsl -l -v`. **Do not use this one.** It's a stripped-down system built only to run Docker's engine internals — no `apt`, no `sudo`, and Docker can wipe or rebuild it on any update. We need a proper, general-purpose Ubuntu distro instead.

**Open PowerShell as Administrator** (right-click Start → search "PowerShell" → right-click **Windows PowerShell** → **Run as administrator**), then run:

```
wsl --install -d Ubuntu
```

Wait for the install to finish. If it asks you to reboot, do that now before continuing.

After install (or after reboot — run `wsl -d Ubuntu` if it doesn't open automatically), you'll be prompted:

```
Enter new UNIX username:
New password:
Retype new password:
```

Pick anything you like — it doesn't need to match your Windows login. **Remember this password**, you'll need it for `sudo` in the next steps.

> **Always enter Ubuntu with `wsl -d Ubuntu`, not just `wsl`.** Even after setting Ubuntu as default, it's easy to accidentally end up back in a plain Windows prompt (e.g. after `exit`) and not notice. Before running any Linux command, check your prompt: Ubuntu looks like `yourname@COMPUTERNAME:/path$`. If you see `PS D:\...>` or `C:\...>` instead, you're still in Windows — commands like `sudo` or `ansible-playbook` will fail there, sometimes with confusing errors (Windows 11 has its own unrelated built-in `sudo` feature that's disabled by default, so typing Linux's `sudo` at a Windows prompt gives an unrelated "Sudo is disabled on this machine" message — that's a sign you're in the wrong shell, not a real problem to fix).

### 1.2 Set Ubuntu as your default WSL distro

Back in a **normal (non-admin)** terminal:

```
wsl --set-default Ubuntu
```

Confirm both distros are present, with Ubuntu marked as default (`*`):

```
wsl -l -v
```

Expected output:

```
  NAME              STATE           VERSION
* Ubuntu            Running         2
  docker-desktop    Running         2
```

### 1.3 Install Ansible inside Ubuntu

```
wsl
sudo apt update
sudo apt install -y ansible
ansible-playbook --version
```

The last command should print a real version number (e.g. `ansible-playbook [core 2.16.x]`). That confirms Ansible is ready.

### 1.4 Enable Docker Desktop's WSL integration for Ubuntu

Open **Docker Desktop → Settings (gear icon) → Resources → WSL Integration**.

You'll see a list of toggles, one per installed WSL distro. Switch **Ubuntu** to **ON**, then click **Apply & Restart**.

This is what makes `kubectl` (and the `docker-desktop` Kubernetes context) visible from inside Ubuntu — without it, `kubectl` commands inside WSL will fail with `Command 'kubectl' not found`, even though Docker Desktop itself is running fine on Windows.

> **Open a brand new WSL session after enabling this.** An already-open `wsl` shell won't pick up the change. Close it (`exit`) and re-enter with `wsl -d Ubuntu` before testing `kubectl`.

### 1.5 Confirm kubectl works inside Ubuntu

Enter Ubuntu explicitly:

```
wsl -d Ubuntu
```

Check your prompt looks like `yourname@COMPUTERNAME:/path$` before continuing. Then:

```
kubectl version --client
kubectl get deployments
```

You should see your cluster's existing deployments listed. If `kubectl` isn't found, double-check Step 1.4 and that you opened a fresh session afterward.

```
exit
```

to leave WSL and return to Windows.

### CMD or PowerShell?

**Either works, no difference.** `wsl` is a native Windows executable — it behaves identically from `cmd.exe` or PowerShell for every command in this guide. The one exception: use an **elevated (Administrator)** terminal specifically for `wsl --install -d Ubuntu` in Step 1.1 — installing a new distro needs admin rights. Every other command here can run from a normal, non-admin terminal in either CMD or PowerShell.

---

## Part 2 — Project setup

### 2.1 Clone the repo

```
git clone https://github.com/avianage/ibm-ansible-demo-4-jun-2026.git
cd ibm-ansible-demo-4-jun-2026
```

### 2.2 Repo structure

```
ibm-ansible-demo-4-jun-2026/
├── app/                    Node.js app (Express server + Jest tests)
│   ├── Dockerfile
│   ├── server.js
│   ├── package.json
│   └── test/
├── k8s/                    Kubernetes manifests
│   ├── deployment.yaml
│   └── service.yaml
├── ansible/                Ansible playbook that performs the deploy
│   ├── inventory.ini
│   └── deploy-playbook.yml
└── Jenkinsfile
```

### 2.3 Create your DockerHub credential in Jenkins

1. **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**
2. Kind: **Username with password**
3. Scope: **Global**
4. Username: your DockerHub username
5. Password: a DockerHub **Personal Access Token** (not your account password — go to hub.docker.com → Account Settings → Security → New Access Token, with Read & Write scope)
6. ID: `DOCKERHUB_CREDENTIALS` (must match exactly — the Jenkinsfile references this ID)
7. Click **Create**

Paste the token from a plain text editor first (no trailing spaces/newline) rather than pasting directly from the browser — stray whitespace here is a common cause of silent login failures.

### 2.4 Update the image name

Open `Jenkinsfile` and change:

```groovy
IMAGE_NAME = "vamandeshmukh/ibm-ansible-demo"
```

to `<your-dockerhub-username>/ibm-ansible-demo`. Also update the same username inside `ansible/deploy-playbook.yml` (`image_name` variable) and `k8s/deployment.yaml` (`image:` line) to match.

### 2.5 Kubernetes setup is automatic

Unlike the `ibm-cicd-demo` pipeline, this one **doesn't need a manual `kubectl apply` step**. The Ansible playbook's first task runs `kubectl apply -f k8s/` every time it deploys — this creates the Deployment/Service on the very first run, and safely does nothing on every run after (that's what `kubectl apply` is designed for: it only changes what's actually different). Push code, trigger Jenkins, done.

One tradeoff worth knowing: this means the *shape* of the Deployment (replica count, ports, labels) is also managed by Git now — if you hand-edit anything in the live cluster with `kubectl edit`, the next pipeline run will silently revert it back to whatever's in `k8s/deployment.yaml`. That's expected behavior for this kind of setup (it's the same principle behind GitOps), but worth calling out explicitly to trainees so it doesn't look like a bug.

### 2.6 Create the Jenkins job

1. **New Item → Pipeline**
2. Name it `ibm-ansible-demo`
3. Under **Pipeline**, set **Definition** to **Pipeline script from SCM**
4. SCM: **Git**, Repository URL: your repo's URL, Branch: `main`
5. Script Path: `Jenkinsfile`
6. Save

---

## Part 3 — Running the pipeline

Click **Build Now**. The pipeline runs these stages in order:

| Stage | What it does |
|---|---|
| Checkout | Clones the repo |
| Install & Test | `npm install` + `npm test` on the Node.js app |
| Docker Build | Builds and tags the image with the current build number and `latest` |
| Docker Push | Logs into DockerHub and pushes both tags |
| Deploy to Kubernetes (Ansible) | Runs `ansible-playbook` inside WSL2, which patches the deployment's image and waits for rollout |
| Verify | Checks pod status and curls the app's `/health` endpoint |

On success, the app is reachable at:

```
http://localhost:30081/health
```

---

## Troubleshooting

**`ansible-playbook: not found`**
Ansible isn't installed inside your default WSL distro, or your default distro is still `docker-desktop` instead of Ubuntu. Re-check Part 1.

**`sudo: not found`**
You're inside the `docker-desktop` distro, not Ubuntu. Run `wsl -l -v` to check which distro is marked default (`*`), and re-run `wsl --set-default Ubuntu` if needed.

**`Sudo is disabled on this machine` or `ansible-playbook is not recognized as the name of a cmdlet...`**
You're actually in a **Windows** PowerShell/CMD prompt, not inside WSL — easy to end up here after an `exit`. Check your prompt: Ubuntu looks like `yourname@COMPUTERNAME:/path$`; PowerShell looks like `PS D:\...>`. Run `wsl -d Ubuntu` to get back into Ubuntu before retrying.

**`Command 'kubectl' not found` inside the Ubuntu shell**
Docker Desktop's WSL Integration toggle for Ubuntu isn't enabled (Step 1.4), or you're still in an old WSL session opened before enabling it. Enable the toggle, fully close and reopen the WSL session (`exit` then `wsl -d Ubuntu`), and retry.

**Docker Build fails with `500 Internal Server Error ... dockerDesktopLinuxEngine/_ping`**
Docker Desktop's engine isn't fully started. Quit and relaunch Docker Desktop, wait for the whale icon to settle, then retry.

**`docker login` fails with `unauthorized: incorrect username or password` despite a correct token**
This is usually Windows `cmd.exe` mangling special characters when piping the token. The Jenkinsfile already works around this by routing the login through PowerShell — if you're troubleshooting a variant of this pipeline, check that the `docker login` step uses:
```groovy
bat 'powershell -Command "$env:DOCKERHUB_CREDENTIALS_PSW | docker login -u $env:DOCKERHUB_CREDENTIALS_USR --password-stdin"'
```
rather than a plain `echo %VAR% | docker login`.

**`kubectl` errors with `deployments.apps "ibm-ansible-demo" not found`**
This shouldn't happen anymore — the playbook's first task (`kubectl apply -f k8s/`) creates the Deployment automatically on first run. If you still see this, check that `k8s/deployment.yaml` and `k8s/service.yaml` actually exist in the repo and were pulled correctly by the Checkout stage, and that `kubectl config current-context` (run from inside Ubuntu) says `docker-desktop`.

**Pod stuck in `ImagePullBackOff` or `ErrImageNeverPull`**
Check `k8s/deployment.yaml`'s `imagePullPolicy`. Since this pipeline always pushes to DockerHub, it should be `IfNotPresent` (not `Never`) so Kubernetes can pull the freshly pushed image if it's not already cached locally.
