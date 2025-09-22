---
sidebar_position: 2
---

# Docusaurus Installation + Self-Hosted GitHub Runner

This section goes over how I setup Docusaurus, a systemd service (via serve), Cloudflare Tunnel, and a GitHub self-hosted runner that deploys locally, so I don't have to expose port 22 on my pfSense firewall.

## Prerequisites

I first start off by updating and upgrading the packages.

```bash
sudo apt update && sudo apt upgrade -y
```

Docusaurus requires Node.js 18.0 or higher.

I installed the latest LTS version.

```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs
```

I then verified the installation.

```bash
node --version
npm --version
```

Git comes already installed on Ubuntu 22.04, so I just verify it's the latest version.

```bash
git --version
```

## Git Setup

I first made the directory to host the Docusaurus files.

```bash
mkdir ~/nesto-docs-site
cd ~/nesto-docs-site
```

I then created a user for Git.

```bash
git config --global user.name "Ernesto Diaz"
git config --global user.email "admin@ernestodiaz.tech"
```

I then initialized Git.

```bash
git init
git branch -M main
git add -A
git commit -m "Initial commit: empty workspace"
```

:::info[Command Explanation]

- `git init` - Makes the current folder a new Git repository by creating a hidden `.git/` directory.
- `git branch -M main` - Sets the current branch to `main`.
- `git add -A` - Stages all changes in the working tree for commit (new files, modifications, and deletions)
- `git commit -m` - Create a commit from what's staged, with the given message.

:::

## Installation

Then created the Docusaurus site (within the `my-website/` directory).

```bash
npx create-docusaurus@latest my-website classic
```

That should create the following directories and files.

```bash
my-website/
├── blog/
│ ├── 2019-05-28-hola.md
│ ├── 2019-05-29-hello-world.md
│ └── 2021-08-26-welcome/
├── docs/
│ ├── hello.md
│ ├── intro.md
│ └── tutorial-basics/
├── src/
│ ├── components/
│ ├── css/
│ └── pages/
├── static/
│ └── img/
├── docusaurus.config.js
├── package.json
├── README.md
└── sidebars.js
```

I then move into the directory `my-website`.

```bash
cd my-website
```

and created the `.gitignore` file so I don't commit junk.

```bash title".gitignore"
node_modules/
.build/
build/
.cache/
dist/
.env
```

Then I committed the change.

```bash
git add -A
git commit -m "Add Docusaurus scaffold"
```

From here I can verify if everything is currently working by running the development server.

```bash
npm start
```

It will take a 1-3 minutes, and then I can navigate to `http://localhost:3000` to see a preview of the website.

I then created the production build.

```bash
npm run build
```

### Serve

:::info

Serve is a convenient tool so you can preview the built site exactly as it will behave in production.

- During development you can use `npm run start`.
- To test production build locally, you can use `serve build`.

:::

I first installed serve globally.

```bash
npm install -g serve
```

:::note[Command Explanation]

- `npm` - Node Package Manager - It's used to install and manage JavaScript libraries and tools.
- `g` - Installs the package globally on the system.
- `serve` - Popular npm package that acts as a lightweight static file server.

:::

Next I setup 'serve' as a system service.

```bash title="/etc/systemd/system/docusaurus-prod.service"
[Unit]
Description=Docusaurus Production Site
After=network.target

[Service]
Type=simple
User=nesto
WorkingDirectory=/home/nesto/nesto-docs-site/my-website
ExecStart=/usr/bin/serve -s build -p 3000
Restart=on-failure
RestartSec=10
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

:::info[Command Explanation]

**[Unit]**

- `Description=` - Friendly name you'll see in `systemctl status`.
- `After=network.target` - Don't start until basic networking is up.

**[Service]**

- `Type=simple` - The process started by `ExecStart` is the service, systemd considers it "started" as soon as the process launches.
- `User=nesto` - Run the service as the unprivileged user "nesto".
- `WorkingDirectory=` - Sets the CWD for the process. Any relative paths in `ExecStart` are resolved here.
- `ExecStart=`
  - `serve` - Node static file server.
  - `-s` - SPA mode (any 404 falls back to index.html).
  - `build` - serve the `build/` folder.
  - `-p 3000` - Listen on port 3000.

**[Install]**

- `WantedBy=` - Makes it start on boot.

:::

I then enabled and started the service.

```bash
sudo systemctl daemon-reload
sudo systemctl enable docusaurus
sudo systemctl start docusaurus
```

Now when I want to run the development server, I will change the port number, since the production server will be using port 3000.

```bash
npm start -- --host 0.0.0.0 --port 3001
```

### SSH (Optional)

Before I setup a GitHub runner, I setup an SSH key to do the first `git push origin main`. For this step, I temporarily allowed port 22 on my pfSense firewall. I mainly did this for learning.

I moved into the `my-website` directory.

```bash
cd ~/nesto-docs-site/my-website
```

Then created the GitHub Actions directory.

```bash
mkdir -p .github/workflows
```

I then created the `deploy.yml` file.

```bash title="deploy.yml"
name: Deploy Documentation

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    name: Deploy to Production Server
    runs-on: ubuntu-latest 
    steps:

      - name: Deploy & build on server via SSH
        uses: appleboy/ssh-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            set -euo pipefail
            echo "🚀 Starting deployment on server at $(date)"

            cd /home/nesto/nesto-docs-site

            git pull --ff-only origin main

            if [ -f package-lock.json ]; then
              npm ci
            else
              npm install
            fi

            npm run build

            sudo systemctl restart docusaurus-prod
            sleep 5
            systemctl is-active --quiet docusaurus-prod

            curl -f http://localhost:3000 > /dev/null

            echo "✅ Deployment completed successfully!"

```

:::info[Command Explanation]

**Top-level**

- `name:` - The display name you'll see in GitHub Actions.
- `on:` - Triggers for the workflow.

**Job**

- `jobs:` - A workflow can have 1+ jobs.
- `deploy` - This job's ID.
- `name:` - The job's display name.
- `runs-on` - Where the job runs.

**Steps**

- Deploy & build on server via SSH
  - `host: ${{ secrets.SERVER_HOST }}` - My server's DNS name (stored as a repo secret).
  - `username: ${{ secrets.SERVER_USER }}` - Linux user to log in as.
  - `key: ${{ secrets.SSH_PRIVATE_KEY }}` - the private SSH key (as a secret) that matches a public key in my server’s `~/.ssh/authorized_keys`.
- `set -euo pipefail`
  - `-e` - Exit on first error,
  - `-u` - Error on undefined variables.
  - `-o pipefail` - Make a pipeline fail if any command fails.
  - `echo "🚀 Starting deployment on server at $(date)"` - Prints a timestamped banner to the job logs.

:::

Next, I generated a dedicated SSH key for GitHub Actions.

```bash
ssh-keygen -t ed25519 -f ~/.ssh/github_actions_key -N ""
```

I then copied the private key, which will be needed for GitHug secrets. (I added it to my Bitwarden)

```bash
cat ~/.ssh/github_actions_key
```

I also copied the public key, which will be used for `authorized_keys`. (I also added this to my Bitwarden)

```bash
cat ~/.ssh/github_actions_key.pub
```

Then I added the public key to my `authorized_keys`.

```bash
cat ~/.ssh/github_actions_key.pub >> ~/.ssh/authorized_keys
```

Then set permissions.

```bash
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

:::info

You can test they key by running:

```bash
ssh -i ~/.ssh/github_actions_key nesto@10.33.99.99 "echo 'SSH connection successful'"
```

:::

Next, I needed to configure GitHub Secrets.

I went to my repo, then clicked on `Settings` ➥ `Secrets and variables` ➥ `Actions` ➥ `New repository secret`.

I created 3 separate secrets:

1. **Secret 1**
   - Name: SERVER_HOST
   - Value: docs.nestodiaz.com
2. **Secret 2**
   - Name: SERVER_USER
   - Value: nesto
3. **Secret 3**
   - Name: SSH_PRIVATE_KEY
   - Value: I passed the private key I created earlier

Next, I needed to allow the docusaurus service to restart without a password prompt.

```bash
sudo visudo
```

```bash title="visudo"
nesto ALL=(ALL) NOPASSWD: /bin/systemctl restart docusaurus, /bin/systemctl status docusaurus, /bin/systemctl is-active docusaurus
```

:::info[Command Explanation]

- `nesto` - The user who gets the special sudo rights.
- `ALL` - From any host/tty (for local sudoers this is fine).
- `(ALL)` - May run commands as any target user (default is root). You can make that explicit with (root) for clarity.
- `NOPASSWD:` - No password prompt for the listed commands.

:::

I then added the workflow file to Git.

```bash
git add .github/workflows/deploy.yml
```

I then committed the workflow.

```bash
git commit -m "Add CI/CD workflow for automatic deployment"
```

Then pushed to GitHub.

```bash
git push origin main
```

After this completed successfully, I blocked port 22 and setup a GitHub runner.

## GitHub Runner

Installing a GitHub runner will eliminate the need to expose SSH to the internet.

:::note

- Runner software runs on local network.
- Runner connects outbound to GitHub (no inbound ports needed)
- Deploys locally without network traversal

```bash
GitHub → Self-Hosted Runner (my network) → Local deployment
```

:::

I go to my repository, and click on `Settings`.

Then go to `Actions` ➥ `Runners` ➥ `New self-hosted runner`.

I select `Linux` and `x64 architecture`.

When making the 'actions-runner' directory, I made sure to create it outside of my docusaurus directory.

```bash
mkdir -p ~/actions-runner
cd ~/actions-runner
```

GitHub will show you all the commands needed to setup the runner.

```bash title="Example"
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz
./config.sh --url https://github.com/ernestodiaztech/nestodiaz.com-docs --token ABCDEFGHIJK...
```

:::info

When prompted, enter:

- Runner group: Default (press **Enter**)
- Runner name: docusaurus-vm
- Work folder: \_work (press **Enter**)
- Labels: docu

:::

To start the runner manually

```bash
./run.sh
```

### Runner as a System Service

I moved to the `actions-runner` directory first.

```bash
cd ~/actions-runner
```

Then ran the service installation.

```bash
sudo ./svc.sh install nesto
```

Then I started the service.

```bash
sudo ./svc.sh start
```

I then enabled it to auto-start on boot.

```bash
sudo systemctl enable actions.runner.ernestodiaztech-nestodiaz.com-docs.docu.service
```

:::note

To find the name of the service, you can use:

```bash
sudo systemctl list-unit-files | grep actions.runner
```

:::

### Workflow

I now needed to modify the workflow file to use the GitHub runner.

```bash
cd ~/nesto-docs-site/my-website
sudo nano .github/workflows/deploy.yml
```

```bash title="deploy.yml"
name: Deploy Documentation

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    name: Deploy to Production Server
    runs-on: self-hosted
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Build Docusaurus site
        run: npm run build
        
      - name: Deploy to production
        run: |
          echo "🚀 Starting local deployment at $(date)"
          
          sudo cp -r build/* /home/nesto/nesto-docs-site/my-website/build/
          
          sudo systemctl restart docusaurus
          
          sleep 5
          if systemctl is-active --quiet docusaurus; then
            echo "✅ Deployment successful!"
            echo "📊 Service status: $(systemctl is-active docusaurus)"
          else
            echo "❌ Service failed to start"
            exit 1
          fi
          
      - name: Verify deployment
        run: |
          echo "🔍 Running health checks..."
          
          sleep 10
          if curl -f http://localhost:3000 > /dev/null 2>&1; then
            echo "✅ Health check passed - site is responding"
          else
            echo "⚠️  Health check warning - site may not be responding"
          fi
          
          echo "🎉 Deployment completed at $(date)"
```

:::info[Command Explanation]

**Top-level**

- `name:` - The display name you'll see in GitHub Actions.
- `on:` - Triggers for the workflow.

**Job**

- `jobs:` - A workflow can have 1+ jobs.
- `deploy` - This job's ID.
- `name:` - The job's display name.
- `runs-on` - Where the job runs.

**Steps**

- Checkout Repository
  - `actions/checkout@v4` - Clones the repo into the runner's workspace so later steps can see the code.
- Setup Node.js
  - `actions/setup-node@v4` - Installs Node 18 on the runner.
- Install Dependencies
  - `npm ci` - Clean, reproducible install only when `package-lock.json` exists.
  - `npm install` - Resolves and writes a lockfile if missing.
- Build
  - `run` - Runs Docusaurus build.

:::

Then I needed to configure sudo permissions to allow the runner to restart the docusaurus service and copy files.

```bash
sudo visudo
```

:::note

This is a safe way to edit the `sudoers` file.

- The `sudoers` file defines who can use `sudo` and what commands they are allowed to run as another user (often root).
- You can allow a specific user to run only certain administrative commands without needing the root password.
- If you just run `sudo nano /etc/sudoers` you could accidently save a syntax error.

  - A syntax error in `sudoers` can lock everyone out of `sudo`, making it impossible to perform administrative tasks without booting into recovery mode.

- `visudo` - Opens the `sudoers` file in a safe editor and validates syntax before saving (if there's an error, it warns you instead of saving a broken config). It also prevents multiple people from editing the file at the same time to avoid conflicts.

:::

```bash title="visudo"
nesto ALL=(ALL) NOPASSWD: /bin/systemctl restart docusaurus
nesto ALL=(ALL) NOPASSWD: /bin/systemctl status docusaurus
nesto ALL=(ALL) NOPASSWD: /bin/systemctl is-active docusaurus
nesto ALL=(ALL) NOPASSWD: /bin/cp -r * /home/nesto/nesto-docs-site/my-website/build/
```

I then added the updated workflow.

```bash
git add .github/workflows/deploy.yml
```

Then commited the changes.

```bash
git commit -m "Update workflow to use self-hosted runner"
```

Then pushed to GitHub.

```bash
git push origin main
```

:::note

You can monitor the deployment on GitHub by going to the `Actions` tab.

:::
