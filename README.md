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

Open **Docker Desktop → Settings (gear icon) → Resources → WSL Integration**, switch the toggle next to **Ubuntu** to **ON**, then click **Apply & Restart**.

This is what lets `kubectl` inside Ubuntu see the same `docker-desktop` Kubernetes cluster as Windows does — without it, Ansible's playbook won't be able to reach the cluster.

### 1.5 Confirm kubectl works inside Ubuntu

Still inside the `wsl` shell (re-enter with `wsl` if you closed it):

```
kubectl get deployments
```

You should see your cluster's existing deployments listed. If this errors out, double-check Step 1.4.

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
git clone https://github.com/dyesmuk/ibm-ansible-demo-4-jun-2026.git
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

### 2.5 One-time manual Kubernetes setup

The pipeline **updates** an existing Deployment — it doesn't create one from scratch. Create it once manually before the first Jenkins run:

```
kubectl apply -f k8s/
kubectl get deployments
```

You should see `ibm-ansible-demo` listed.

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

**Docker Build fails with `500 Internal Server Error ... dockerDesktopLinuxEngine/_ping`**
Docker Desktop's engine isn't fully started. Quit and relaunch Docker Desktop, wait for the whale icon to settle, then retry.

**`docker login` fails with `unauthorized: incorrect username or password` despite a correct token**
This is usually Windows `cmd.exe` mangling special characters when piping the token. The Jenkinsfile already works around this by routing the login through PowerShell — if you're troubleshooting a variant of this pipeline, check that the `docker login` step uses:
```groovy
bat 'powershell -Command "$env:DOCKERHUB_CREDENTIALS_PSW | docker login -u $env:DOCKERHUB_CREDENTIALS_USR --password-stdin"'
```
rather than a plain `echo %VAR% | docker login`.

**`kubectl` errors with `deployments.apps "ibm-ansible-demo" not found`**
The one-time manual `kubectl apply -f k8s/` (Step 2.5) hasn't been run yet, or was run against a different `kubectl` context. Check `kubectl config current-context` — it should say `docker-desktop`.

**Pod stuck in `ImagePullBackOff` or `ErrImageNeverPull`**
Check `k8s/deployment.yaml`'s `imagePullPolicy`. Since this pipeline always pushes to DockerHub, it should be `IfNotPresent` (not `Never`) so Kubernetes can pull the freshly pushed image if it's not already cached locally.
