# Next.js Deployment Guide

## 1. Prepare the VPS

- **SSH into your VPS:**

  ```bash
  ssh user@your-vps-ip
  ```

- **Update the system:**

  ```bash
  sudo apt update && sudo apt upgrade -y
  ```

- **Install essential tools:**

  ```bash
  sudo apt install -y git curl
  ```

- **Install nvm & node:**

  ```bash
    curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.0/install.sh -o /tmp/nvm_setup.sh

    bash /tmp/nvm_setup.sh

    source ~/.bashrc

    nvm install --lts
  ```

## 2. Install Docker

- Install Docker to containerize your Next.js app:

  ```bash
  curl -fsSL https://get.docker.com -o get-docker.sh
  sudo sh get-docker.sh
  sudo usermod -aG docker $USER
  ```

- Log out and back in to apply the Docker group change, or run:

  ```bash
  newgrp docker
  ```

- Verify Docker installation:

  ```bash
  docker --version
  ```

## 3. Install Zellij

- Zellij is a terminal workspace manager to organize your deployment tasks. Install it:

  ```bash
  curl -L https://github.com/zellij-org/zellij/releases/latest/download/zellij-x86_64-unknown-linux-musl.tar.gz | tar -xz
  sudo mv zellij /usr/local/bin/
  ```

- Start a Zellij session:

  ```bash
  zellij
  ```

  Zellij opens a terminal workspace. Use `Ctrl+t` to open the tab menu, create new panes with `Alt+n`, and navigate with `Alt+arrow keys`. You can split panes to monitor logs, edit files, or run commands simultaneously.

## 4. Clone the GitHub Repository

- In a Zellij pane, clone your repository.

  - **For a public repository:**

    ```bash
    git clone https://github.com/your-username/your-repo.git
    cd your-repo
    ```

  - **For a private repository:**

    1.  Generate a Personal Access Token (PAT) on GitHub: Go to GitHub > Settings > Developer settings > Personal access tokens > Tokens (classic). Create a token with `repo` scope, and copy it.
    2.  Clone using the token:

        ```bash
        git clone https://<your-token>@github.com/your-username/your-repo.git
        cd your-repo
        ```

## 5. Create a Dockerfile

- In the `your-repo` directory, create a `Dockerfile` for your Next.js app. This example assumes a standard Next.js setup:

  ```dockerfile
  # Use Node.js LTS image
  FROM node:20-alpine

  # Set working directory
  WORKDIR /app

  # Copy package files
  COPY package*.json ./

  # Install dependencies
  RUN npm install

  # Copy the rest of the app
  COPY . .

  # Build the Next.js app
  RUN npm run build

  # Expose port 3000
  EXPOSE 3000

  # Start the app
  CMD ["npm", "start"]
  ```

  This Dockerfile:

  - Uses a lightweight Node.js Alpine image to fit your 2GB RAM constraint.
  - Installs dependencies, builds the Next.js app, and runs it on port 3000.

## 6. Build and Run the Docker Container

- Build the Docker image:

  ```bash
  docker build -t nextjs-app .
  ```

- Run the container, mapping port 3000:

  ```bash
  docker run -d -p 3000:3000 --name nextjs-container nextjs-app
  ```

- Verify the app is running:

  ```bash
  curl http://localhost:3000
  ```

  You should see your app’s HTML output. If not, check logs in a Zellij pane:

  ```bash
  docker logs nextjs-container
  ```

## 7. Set Up Caddy for Reverse Proxy and Free SSL

- Caddy is a lightweight web server that auto-provisions SSL via Let’s Encrypt and acts as a reverse proxy. Install it:

  ```bash
  sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
  curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
  curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
  sudo apt update
  sudo apt install -y caddy
  ```

- Create a Caddyfile in `/etc/caddy/Caddyfile`:

  ```bash
  sudo nano /etc/caddy/Caddyfile
  ```

- Add the following configuration:

  ```text
  haviij.sbs {
      reverse_proxy localhost:3000
  }
  ```

  This tells Caddy to:

  - Serve `haviij.sbs` and auto-provision SSL.
  - Proxy requests to your Next.js app on port 3000.

- Restart Caddy to apply changes:

  ```bash
  sudo systemctl restart caddy
  ```

  Caddy will automatically obtain and renew an SSL certificate from Let’s Encrypt for `haviij.sbs`.

## 8. Configure Cloudflare DNS

- Log in to Cloudflare and configure your domain:

  - Go to DNS > Records.
  - Add an A record:
    - Name: `@` (for `haviij.sbs`)
    - IPv4 address: Your VPS’s public IP
    - Proxy status: Proxied (enables Cloudflare’s CDN and security)
  - Ensure SSL/TLS settings are set to Full (strict): Go to SSL/TLS > Overview, select “Full (strict)” to enforce HTTPS.

## 9. Open Firewall Ports

- Allow HTTP (80) and HTTPS (443) traffic on your VPS:

  ```bash
  sudo ufw allow 80
  sudo ufw allow 443
  sudo ufw enable
  ```

- Confirm the firewall status:

  ```bash
  sudo ufw status
  ```

## 10. Test Your App

- Visit `https://haviij.sbs` in a browser. Your Next.js app should load with SSL enabled. If it doesn’t, check:

  - Caddy logs in a Zellij pane:

    ```bash
    sudo journalctl -u caddy
    ```

  - Docker container logs:

    ```bash
    docker logs nextjs-container
    ```

  - Cloudflare DNS propagation (may take up to 24 hours, but usually faster).

## 11. Manage with Zellij

- Use Zellij to monitor and manage your setup:

  - Create a pane for Caddy logs: `sudo journalctl -u caddy -f`
  - Create a pane for Docker logs: `docker logs -f nextjs-container`
  - Create a pane for editing files or running commands.
  - Save your Zellij layout for reuse:

    ```bash
    zellij setup --dump-layout > nextjs-deployment.zellij
    ```

  - Restart Zellij with the layout:

    ```bash
    zellij --layout nextjs-deployment.zellij
    ```

## 12. (Optional) Update Your App

- To update your app with new code:

  - Pull the latest code:

    ```bash
    cd your-repo
    git pull origin main
    ```

  - Rebuild and restart the Docker container:

    ```bash
    docker stop nextjs-container
    docker rm nextjs-container
    docker build -t nextjs-app .
    docker run -d -p 3000:3000 --name nextjs-container nextjs-app
    ```
