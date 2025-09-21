---
sidebar_position: 2
---

# Docusaurus Installation

This section will go over how I installed Docusaurus, setup a Cloudflare tunnel, and installed a GitHub runner to avoid opening port 22 on my pfSense firewall.

## Prerequisites

I first start off by updating and upgrading the packages.

```bash
sudo apt update && sudo apt upgrade -y
```

Docusaurus requires Node.js 18.0 or higher.

I install the latest LTS version.

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

I then create a user.

```bash
git config --global user.name "Ernesto Diaz"
git config --global user.email "admin@ernestodiaz.tech"
```

## Installation

I first make the directory to host the files.

```bash
mkrdir ~/my-docusaurus-site
cd ~/my-docusaurus-site
```

Then create the Docusaurus site.

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

From here I can verify if everything is currently working by running the development server.

```bash
nmp start
```

It will take a 1-3 minutes, and then I can navigate to `http://localhost:3000` to see a preview of the website.

I then created the production build.

```bash
nmp run build
```

### Serve

:::info

Serve is a convenient tool so you can preview the built site exactly as it will behave in production.

- During development you can use `npm run start`.
- To test production build locally, you can use `serve build`.

:::

I first installed serve globally.

```bash
npm install -g server
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
WorkingDirectory=/home/nesto/my-docusaurus-site/my-website
ExecStart=/usr/bin/serve -s build -p 3000
Restart=on-failure
RestartSec=10
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

I then enabled and started the service.

```bash
sudo systemctl daemon-reload
sudo systemctl enable docusaurus-prod
sudo systemctl start docusaurus-prod
```

Now when I want to run the development server, I will change the port number, since the production server will be using port 3000.

```bash
npm start -- --host 0.0.0.0 --port 3001
```

## Cloudflare Tunnel

## GitHub Runner

Installing a GitHub runner will eliminate the need to expose SSH to the internet.

:::info

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

:::note

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
    runs-on: self-hosted  # Changed from ubuntu-latest

    steps:
      # Step 1: Checkout code (automatically uses local runner)
      - name: Checkout repository
        uses: actions/checkout@v4

      # Step 2: Setup Node.js (if not already installed)
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      # Step 3: Install dependencies
      - name: Install dependencies
        run: npm ci

      # Step 4: Build the site
      - name: Build Docusaurus site
        run: npm run build

      # Step 5: Deploy locally (no SSH needed!)
      - name: Deploy to production
        run: |
          echo "🚀 Starting local deployment at $(date)"

          # Copy built files to production location
          # Since we're running on the same machine, we can copy directly
          sudo cp -r build/* /home/nesto/nesto-docs-site/my-website/build/

          # Restart the production service
          sudo systemctl restart docusaurus

          # Wait and verify service is running
          sleep 5
          if systemctl is-active --quiet docusaurus; then
            echo "✅ Deployment successful!"
            echo "📊 Service status: $(systemctl is-active docusaurus)"
          else
            echo "❌ Service failed to start"
            exit 1
          fi

      # Step 6: Health check
      - name: Verify deployment
        run: |
          echo "🔍 Running health checks..."

          # Test if the service is responding
          sleep 10
          if curl -f http://localhost:3000 > /dev/null 2>&1; then
            echo "✅ Health check passed - site is responding"
          else
            echo "⚠️  Health check warning - site may not be responding"
          fi

          echo "🎉 Deployment completed at $(date)"
```

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
