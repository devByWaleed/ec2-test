# 🚀 Node.js Deployment Guide — EC2 + PM2 + Nginx

This guide walks through creating an AWS EC2 instance from scratch and deploying a Node.js app on it using PM2 (process manager) and Nginx (reverse proxy).

---

## Part 1: Create Your EC2 Instance

### 1️⃣ Log in to AWS Console

Go to [console.aws.amazon.com](https://console.aws.amazon.com) and sign in. Search for **EC2** in the top search bar and open the EC2 dashboard.

### 2️⃣ Launch a new instance

Click **"Launch Instance"** (orange button, top right of the EC2 dashboard).

### 3️⃣ Configure the instance

- **Name**: give it something recognizable, e.g. `node-app-server`.
- **AMI (Amazon Machine Image)**: choose **Ubuntu Server 22.04 LTS** — this is the OS the server will run.
- **Instance type**: choose **t2.micro** (or t3.micro) — this is free-tier eligible and enough for a basic Node app.
- **Key pair**: click **"Create new key pair"**, name it (e.g. `ec2-key`), key type **RSA**, format **.pem**. Click **Create** — this automatically downloads the `.pem` file to your computer. **This file is your only way to SSH into the server, so don't lose it.**
- **Network settings**: click **Edit**, and make sure these inbound rules exist (add them if missing):
  - **SSH (port 22)** — source: My IP (safer) or Anywhere (0.0.0.0/0) if you need flexible access
  - **HTTP (port 80)** — source: Anywhere (0.0.0.0/0) — this is what lets browsers reach your app later
- **Storage**: default 8GB is fine for a basic Node app.

### 4️⃣ Launch it

Click **"Launch Instance"**. Wait about 30–60 seconds, then go to **Instances** in the left sidebar — you should see your instance with status **Running**.

### 5️⃣ Note your connection details

Click on the instance, and from the details panel copy:
- **Public IPv4 address** (e.g. `51.21.182.234`), or
- **Public IPv4 DNS** (e.g. `ec2-51-21-182-234.eu-north-1.compute.amazonaws.com`)

You'll use one of these to connect via SSH in Part 2.

---

## Part 2: Deploy the App

### 1️⃣ Open Git Bash in the right folder

Open Git Bash, then navigate to wherever your `.pem` key file is saved. For example, if it's in Downloads:
```bash
cd ~/Downloads
```
> This just moves your terminal "into" that folder, so the next commands can find the key file by name instead of needing its full path.

Confirm it's there:
```bash
ls
```
> Lists all files in the current folder — just a quick check that the `.pem` file is actually where you think it is.

You should see your `.pem` file listed (e.g. `my-key.pem`).

### 2️⃣ Lock down the key file permissions

```bash
chmod 400 my-key.pem
```
> Restricts the key file so only you can read it (no one else can write or execute it). SSH refuses to use keys that are too "open," so this step is required.

This is required — SSH refuses to use keys with overly open permissions.

### 3️⃣ Get your instance's Public DNS or IP

Go to AWS Console → EC2 → Instances → click your instance → copy either the **Public IPv4 DNS** (looks like `ec2-xx-xx-xx-xx.compute.amazonaws.com`) or **Public IPv4 address**.

### 4️⃣ Connect via SSH

```bash
winpty ssh -i my-key.pem ubuntu@your-public-dns-or-ip
```
> `ssh` opens a secure remote connection to your server. `-i my-key.pem` tells it which key to authenticate with. `winpty` is a Windows-only wrapper that fixes a common Git Bash display bug so the session shows properly instead of appearing "stuck."

Replace `your-public-dns-or-ip` with what you copied. Replace `ubuntu` with `ec2-user` if you launched an Amazon Linux AMI instead of Ubuntu.

- First connection → it'll ask `Are you sure you want to continue connecting (yes/no)?` → type `yes`, press Enter.
- If successful, your prompt changes to something like `ubuntu@ip-172-31-xx-xx:~$` — **you are now inside the EC2 instance.** Everything from here runs on the server, not your laptop.

> !!! This same command will use if the connection lost and you have to reconnect after connection is stable

> If issue occurs, click this `View Fix` dropdown
<details>
  <summary>View Fix</summary>

`chmod` alone won't fix this on Windows — you need to fix the Windows ACL (permissions) directly.
1. Open **PowerShell** (not Git Bash) — search "PowerShell" in Start menu.
2. Navigate to the folder with your key:
   ```powershell
   cd D:\Path
   ```
   > Moves PowerShell into the folder containing your key file, same idea as `cd` in Git Bash.
3. Run these commands one by one:
   ```powershell
   icacls.exe my-key.pem /reset
   icacls.exe my-key.pem /grant:r "$($env:USERNAME):(R)"
   icacls.exe my-key.pem /inheritance:r
   ```
   - `/reset` clears existing permissions
   - `/grant:r "$env:USERNAME:(R)"` gives only your Windows user read access
   - `/inheritance:r` removes inherited permissions from the parent folder (this is what "Authenticated Users" access was coming from)

4. Verify it worked:
   ```powershell
   icacls.exe my-key.pem
   ```
   > Prints the current permission list for the file, so you can confirm only your username has access.

   You should now see only your username listed (and maybe SYSTEM/Administrators), not "Authenticated Users" or "Everyone".
</details>

### 5️⃣ Update the system

```bash
sudo apt update && sudo apt upgrade -y
```
> `sudo apt update` refreshes the list of available software versions. `sudo apt upgrade -y` actually installs any pending updates, auto-confirming prompts with `-y`. Keeps the server's software current and secure.

Wait for it to finish (may take a minute or two, might ask you to confirm with `Y` — just press Enter/`Y` if prompted).

### 6️⃣ Install Node.js 18+

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
```
> The `curl` command downloads and runs a script that adds Node.js 18's official package source to the server. `sudo apt install -y nodejs` then installs Node.js itself from that source.

Verify:
```bash
node -v
npm -v
```
> Prints the installed Node.js and npm (Node's package manager) versions, confirming the install worked.

You should see `v18.x.x` and an npm version number.

### 7️⃣ Install PM2 and Nginx

```bash
sudo npm install -g pm2
sudo apt install -y nginx
```
> `npm install -g pm2` installs PM2 globally so it keeps your Node app running in the background and restarts it automatically if it crashes. `apt install -y nginx` installs Nginx, the web server that will sit in front of your app.

Check Nginx is running:
```bash
sudo systemctl status nginx
```
> Shows whether the Nginx service is currently active. Press `q` to exit this status screen.

Should say `active (running)`. Press `q` to exit that screen.

### 8️⃣ Get your code onto the server

**If your project is on GitHub:**
```bash
git clone https://github.com/your-username/your-repo.git node_app
cd node_app
```
> `git clone` downloads a copy of your GitHub repository onto the server into a folder named `node_app`. `cd node_app` moves into that folder.

**If it's only on your laptop**, open a **second, new Git Bash window** (don't close your SSH session) and run this from your local project folder:
```bash
scp -i /path/to/my-key.pem -r ./node_app ubuntu@your-public-dns-or-ip:~/node_app
```
> `scp` securely copies files from your laptop to the server over SSH. `-r` copies the whole folder (recursively), and `-i` points to your key for authentication.

Then switch back to your SSH window and run:
```bash
cd node_app
```
> Moves into the newly copied project folder on the server.

### 9️⃣ Install dependencies and configure environment variables

```bash
npm install
cp .env.example .env
nano .env
```
> `npm install` downloads all the packages your app depends on (listed in `package.json`). `cp .env.example .env` duplicates the example env file into a real `.env` file. `nano .env` opens that file in a simple text editor so you can fill in real values.

Edit the values (PORT, MONGO_URI, JWT_SECRET, etc.) using arrow keys. Save with `Ctrl+O` then Enter, exit with `Ctrl+X`.

### 🔟 Start the app with PM2

```bash
pm2 start server.js --name node_app
pm2 save
pm2 startup
```
> `pm2 start` launches your app and keeps it running in the background under the name `node_app`. `pm2 save` remembers this process list. `pm2 startup` generates a command that makes PM2 (and your app) start automatically if the server reboots.

`pm2 startup` will print a command starting with `sudo env PATH=...` — **copy that exact line and run it too**, so your app survives reboots.

Check logs to confirm it's running:
```bash
pm2 logs node_app
```
> Streams your app's live console output, so you can confirm it started correctly or spot errors.

Press `Ctrl+C` to stop watching (app keeps running in background).

### 1️⃣1️⃣ Configure Nginx as reverse proxy

```bash
sudo nano /etc/nginx/sites-available/default
```
> Opens Nginx's default site configuration file in a text editor, so you can tell it how to route incoming traffic.

Delete everything and replace with:
```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
> This tells Nginx: "listen on port 80 (standard web traffic), and forward every request to the Node app running on port 3000." The extra header lines pass along the visitor's real IP and protocol info to your app.

(If you don't have a domain yet, you can leave `server_name` as `_;` to accept any hostname/IP.)

Save (`Ctrl+O`, Enter, `Ctrl+X`), then:
```bash
sudo nginx -t
sudo systemctl restart nginx
```
> `nginx -t` checks your config file for typos/errors before applying it. `systemctl restart nginx` reloads Nginx with the new configuration.

`nginx -t` must say "syntax is ok" / "test is successful" before you restart.

### 1️⃣2️⃣ Open port 80 in the Security Group

Go to AWS Console → EC2 → your instance → **Security** tab → click the security group name → **Edit inbound rules** → **Add rule**:
- Type: `HTTP`
- Port: `80`
- Source: `Anywhere-IPv4` (0.0.0.0/0)

Save rules.

### 1️⃣3️⃣ Test it

Open a browser and go to:
```
http://your-public-dns-or-ip
```
> This sends a request to Nginx on your server, which forwards it to your Node app and returns the response — confirming the full chain (browser → Nginx → PM2 → Node app) works.

You should see your app respond (e.g. "Server is running!" if you're using the basic server.js from earlier).