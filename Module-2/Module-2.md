# Module 2: The Application Runtime

## Learning objectives

By the end of this module, you will be able to:

- Deploy a Vue.js frontend and FastAPI backend to an Ubuntu server.
- Install Python and Node.js dependencies in isolated environments.
- Build and verify a production frontend.
- Run FastAPI with Gunicorn and Uvicorn workers under Supervisor.
- Configure Nginx to serve the frontend and proxy API requests.
- Apply security headers safely and prepare the application for HTTPS.
- Troubleshoot service failures, file permissions, and HTTP 502 errors.

## 1. Manual deployment

### Introduction

In Module 1, you established a secure server foundation. In this module, you will turn that server into a web application host.

The example application uses:

- **Vue.js** for the frontend.
- **FastAPI** for the backend API.
- **Python virtual environments** to isolate backend dependencies.
- **Node.js and pnpm** to install frontend dependencies and build static assets.
- **Gunicorn and Uvicorn** to run the backend.
- **Supervisor** to manage the backend process.
- **Nginx** to serve frontend files and forward API requests.

The commands assume an Ubuntu server and a project containing `frontend/` and `backend/` directories. Adjust paths and application entry points to match your own project.

## 2. Move the project to the server

### 2.1 Prepare the deployment archive

On your local computer, open a terminal in the project root directory.

Before creating an archive, check that the project does not contain secrets such as `.env` files, private keys, database dumps, access tokens, or production credentials. Do not include these in a deployment archive.

Create the archive while excluding files that can be regenerated:

```bash
tar -czvf source_code.tar.gz \
  --exclude='node_modules' \
  --exclude='venv' \
  --exclude='__pycache__' \
  --exclude='.git' \
  --exclude='.env' \
  --exclude='.env.*' \
  .
```

Review the archive before uploading it:

```bash
tar -tzf source_code.tar.gz
```

**Important:** The exclusions above do not guarantee that every secret is excluded. Inspect the file list and add any project-specific secret paths to the exclusions. Keep production secrets outside the source archive and load them through a protected configuration file or secrets-management system.

### 2.2 Upload the archive

If you configured an SSH alias named `my-website` in Module 1, upload the archive with:

```bash
scp source_code.tar.gz my-website:/tmp/
```

Connect to the server and confirm the file arrived:

```bash
ssh my-website
ls -lh /tmp/source_code.tar.gz
```

The `/tmp` directory is used as a temporary transfer location. It is not the final location for application code.

### 2.3 Create the application directory

Create a dedicated directory for the application:

```bash
sudo mkdir -p /web_app
```

Move and extract the archive:

```bash
sudo mv /tmp/source_code.tar.gz /web_app/
sudo tar -xzf /web_app/source_code.tar.gz -C /web_app
```

Inspect the extracted structure:

```bash
ls -la /web_app
```

You should see the project files, including `frontend/` and `backend/`.

For this learning deployment, you can assign the project to your deployment account:

```bash
sudo chown -R <your_username>:<your_username> /web_app
```

Replace `<your_username>` with your actual Linux username.

**Production security note:** Do not make the application directory broadly writable. A stronger production arrangement separates the account that deploys code from the unprivileged account that runs the application. The runtime account should not be able to modify application source code or its own executable files. The rest of this module uses restricted permissions rather than world-writable directories.

Check ownership and permissions:

```bash
ls -ld /web_app
ls -la /web_app
```

### 2.4 Install Python prerequisites

Install the packages required to create and build Python environments:

```bash
sudo apt update
sudo apt install -y python3-pip python3-venv python3-dev build-essential
```

Verify the installed versions:

```bash
python3 --version
pip3 --version
```

Use the Python version supported by your application and its dependencies. Do not assume that the newest available Python version is compatible with every project.

## 3. Run and verify the FastAPI backend

### 3.1 Create a Python virtual environment

Move into the backend directory:

```bash
cd /web_app/backend
```

Create and activate a virtual environment:

```bash
python3 -m venv venv
source venv/bin/activate
```

A virtual environment keeps the project's Python packages separate from the system Python installation.

Confirm that Python and pip point to the virtual environment:

```bash
which python
which pip
```

The paths should point to `/web_app/backend/venv/`.

### 3.2 Install backend dependencies

Upgrade pip and install the project dependencies:

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt
```

For repeatable deployments, maintain and review a dependency lock or constraints file, pin tested versions where appropriate, and install from that controlled dependency set. Do not casually upgrade production dependencies during deployment.

Verify the relevant packages:

```bash
pip show fastapi uvicorn
```

If the application imports additional packages, confirm that they are installed as well.

### 3.3 Test the backend locally

Start Uvicorn bound to the loopback interface:

```bash
uvicorn main:app --host 127.0.0.1 --port 8000
```

Here, `main:app` means that Python should import the `app` object from `main.py`. Change this value if your application's module or FastAPI object has a different name.

Binding to `127.0.0.1` makes the test server accessible only from the server itself. Avoid binding to `0.0.0.0` for this test unless external access is specifically required and properly restricted.

Open a second SSH session and test the health endpoint:

```bash
curl -i http://127.0.0.1:8000/api/health
```

A successful response might look like:

```http
HTTP/1.1 200 OK
content-type: application/json

{"status":"ok"}
```

The exact response depends on your application.

If the endpoint returns `404 Not Found`, confirm that `/api/health` is the correct route. If the connection is refused, check whether Uvicorn is still running and whether it started successfully.

Stop the temporary server with `Ctrl+C` when testing is complete.

**Firewall reminder:** An inactive UFW firewall does not make an application safe to expose. Binding to `127.0.0.1` prevents external network connections to this test service regardless of UFW, while your cloud provider may also have separate firewall rules.

## 4. Set up Node.js and build the frontend

### 4.1 Install Node.js

Vue.js projects commonly use Node.js and a package manager to install dependencies and create a production build.

For a user-managed development environment, NVM can install a Node.js LTS version:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source ~/.bashrc
```

Then install and verify Node.js:

```bash
nvm install --lts
nvm use --lts

node --version
npm --version
```

**Security note:** Downloading a script and piping it directly to `bash` executes it immediately. For stricter environments, download the script first, inspect it, verify its source, and then run it.

Use a Node.js version supported by your project. For automated or production builds, pin the intended version instead of relying on an unspecified future LTS version.

### 4.2 Install pnpm and frontend dependencies

Install the pnpm version supported by your project. For example:

```bash
npm install -g pnpm@10
```

Verify it:

```bash
pnpm --version
```

Move to the frontend directory:

```bash
cd /web_app/frontend
```

If the repository contains a committed `pnpm-lock.yaml`, install the locked dependency versions:

```bash
pnpm install --frozen-lockfile
```

The frozen-lockfile option helps prevent a deployment from silently changing the dependency lockfile. If the project uses a different package manager, follow the project's committed lockfile and build instructions instead.

### 4.3 Build the frontend

Run the production build:

```bash
pnpm run build
```

A successful Vue/Vite build typically creates a `dist/` directory containing static HTML, JavaScript, CSS, and other assets.

Check the output:

```bash
ls -lah /web_app/frontend/dist
```

If the build fails with an out-of-memory error, first check the server's available resources:

```bash
free -h
df -h /
```

On a small server, adding swap can help with memory pressure, although swap is slower than physical RAM and cannot guarantee that a build will succeed.

### 4.4 Optional: Add swap if necessary

First check whether swap is already configured:

```bash
swapon --show
grep -nE '^[^#].*\sswap\s' /etc/fstab
```

Only create a new swap file if your server needs one and sufficient disk space is available. The following example creates a 2 GiB swap file on a system that does not already have one:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Verify it:

```bash
swapon --show
free -h
```

To enable it after reboot, add the following entry to `/etc/fstab` **only if an equivalent entry does not already exist**:

```text
/swapfile none swap sw 0 0
```

You can edit the file with:

```bash
sudo nano /etc/fstab
```

Avoid repeatedly appending the same entry, which can create configuration problems. After editing, verify the configuration carefully before rebooting.

### 4.5 Handle Node.js heap limits carefully

If Node.js reports a JavaScript heap-limit error, you can test a higher heap limit, for example:

```bash
NODE_OPTIONS="--max-old-space-size=2048" pnpm run build
```

This sets the V8 old-generation heap limit to approximately 2 GiB. It does **not** mean that Node.js can safely use 2 GiB of RAM, and it does not convert disk swap into physical memory. The operating system must still have enough memory and other resources for the build.

If the build continues to fail, consider building the frontend on a suitable CI runner or development machine and deploying the resulting `dist/` artifact. This can be more reliable than increasing memory use on a small production VM.

### 4.6 Test the static build without exposing a development port

You can test the built files through an SSH tunnel rather than opening port 8080 to the internet.

From the server, run the temporary web server bound to loopback:

```bash
cd /web_app/frontend/dist
python3 -m http.server 8080 --bind 127.0.0.1
```

Keep this process running. On your local computer, open a second terminal and establish the tunnel:

```bash
ssh -L 8080:127.0.0.1:8080 my-website
```

Now visit `http://localhost:8080` in your local browser.

This verifies that the static files can be served. API requests may fail at this stage because the production Nginx routing has not been configured yet.

Stop the temporary web server and SSH tunnel with `Ctrl+C` when finished.

## 5. Run FastAPI with Gunicorn and Supervisor

### 5.1 Understand the process architecture

The production request flow will be:

1. Nginx receives requests from users.
2. Nginx serves the Vue.js static files directly.
3. Nginx forwards `/api` requests to the backend through a Unix socket.
4. Gunicorn manages the backend worker processes.
5. Uvicorn workers run the FastAPI application.
6. Supervisor starts and monitors the Gunicorn process.

Supervisor can restart a process when it exits, but it does not guarantee zero downtime or correct application behavior. Health checks, logging, and deployment testing are still required.

### 5.2 Install the backend server packages

Activate the backend virtual environment:

```bash
cd /web_app/backend
source venv/bin/activate
```

Install Gunicorn and the Uvicorn worker package:

```bash
pip install gunicorn uvicorn-worker
```

The separate `uvicorn-worker` package is the preferred worker-class package for new deployments rather than relying on the deprecated `uvicorn.workers` module. Record these dependencies in the project's controlled dependency file so the installation can be reproduced.

### 5.3 Create a dedicated socket group

Nginx runs as `www-data` on a standard Ubuntu installation. It needs permission to reach the Gunicorn socket, but it should not receive broad access to your personal files.

Create a dedicated group for socket access:

```bash
sudo groupadd --system webapp-sock
sudo usermod -aG webapp-sock www-data
```

If the group already exists, do not create it again. Confirm membership:

```bash
getent group webapp-sock
id www-data
```

Use a dedicated non-root account for the application process in production. The following examples use `<app_user>` for that account. Replace it with the account you actually configure; do not run the application as root.

Ensure the application account can access the socket group. For example, if that account already exists:

```bash
sudo usermod -aG webapp-sock <app_user>
```

Group membership changes apply to newly started processes. Nginx must be restarted or otherwise started with the updated supplementary group membership before it can use the group.

### 5.4 Create the socket directory

For production, a runtime directory under `/run` is preferable to creating a socket inside the application source tree. Runtime files are temporary and are normally recreated after reboot.

Create the directory:

```bash
sudo install -d -o <app_user> -g webapp-sock -m 2770 /run/web_app
```

The setgid bit on the directory helps new entries inherit the directory's group. The group permissions allow members of `webapp-sock` to access the socket directory.

Because `/run` is temporary, this directory must be recreated at startup. For a production deployment, configure a systemd-tmpfiles rule or a suitable systemd runtime-directory arrangement. Do not assume that a manually created `/run/web_app` directory will survive reboot.

### 5.5 Create the Gunicorn startup script

Create a script outside the virtual environment so that it is not lost when dependencies are recreated:

```bash
sudo nano /web_app/backend/gunicorn_start
```

Use the following script, adjusting the application entry point if necessary:

```bash
#!/bin/bash
set -e

APPDIR="/web_app/backend"
SOCKFILE="/run/web_app/gunicorn.sock"

cd "$APPDIR"

exec "$APPDIR/venv/bin/gunicorn" main:app \
  --name web_app \
  --workers 2 \
  --worker-class uvicorn_worker.UvicornWorker \
  --timeout 120 \
  --bind "unix:$SOCKFILE" \
  --umask 007 \
  --forwarded-allow-ips="*" \
  --access-logfile - \
  --error-logfile - \
  --log-level info
```

Save the file and restrict who can modify it:

```bash
sudo chown root:root /web_app/backend/gunicorn_start
sudo chmod 755 /web_app/backend/gunicorn_start
```

The script's key settings are:

- `main:app`: identifies the FastAPI application.
- `--workers 2`: starts two worker processes as an initial setting; adjust after measuring memory and traffic.
- `uvicorn_worker.UvicornWorker`: runs FastAPI using the ASGI worker implementation.
- `--timeout 120`: sets Gunicorn's worker timeout. It is not a universal solution for slow requests.
- `--bind unix:...`: uses a local Unix socket instead of exposing an application TCP port.
- `--umask 007`: restricts socket permissions to the owner and group rather than allowing access to everyone.
- `--forwarded-allow-ips="*"`: trusts forwarded headers from all peers. This is appropriate only when the application is reachable exclusively through a trusted local proxy path and no untrusted process can connect to the socket. Do not use this setting blindly in a different network architecture.
- `--access-logfile -` and `--error-logfile -`: send logs to standard output and standard error, which Supervisor can capture.
- `--log-level info`: avoids verbose debug logging in normal production operation.

The worker count is a starting point, not a universal formula. Each worker consumes memory, so check resource use on small VMs before increasing the number.

### 5.6 Configure Supervisor

Install Supervisor:

```bash
sudo apt update
sudo apt install -y supervisor
```

Create a log directory:

```bash
sudo install -d -o <app_user> -g <app_user> -m 750 /web_app/backend/logs
```

Create a Supervisor configuration file:

```bash
sudo nano /etc/supervisor/conf.d/web_app.conf
```

Add:

```ini
[program:web_app]
command=/web_app/backend/gunicorn_start
directory=/web_app/backend
user=<app_user>
autostart=true
autorestart=true
startsecs=5
startretries=3
stopsignal=TERM
stopasgroup=true
killasgroup=true
redirect_stderr=true
stdout_logfile=/web_app/backend/logs/supervisor.log
stdout_logfile_maxbytes=20MB
stdout_logfile_backups=5
environment=LANG="C.UTF-8"
```

Replace `<app_user>` with the actual non-root application account. Ensure that account can read the application and its virtual environment, and write only to the directories where the application genuinely needs to write. If the app needs persistent uploads or generated files, store them in a separate, deliberately permissioned directory.

The log configuration limits the size of each log file and retains a small number of backups. For larger deployments, use centralized logging or a dedicated log-management service.

Update Supervisor and start the application:

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl status web_app
```

If the program is already registered, restart it after changing its configuration:

```bash
sudo supervisorctl restart web_app
```

### 5.7 Verify the backend and socket

Check the Supervisor status:

```bash
sudo supervisorctl status web_app
```

Inspect recent application logs:

```bash
sudo supervisorctl tail -100 web_app
```

Test the API through the Unix socket:

```bash
curl --unix-socket /run/web_app/gunicorn.sock \
  http://localhost/api/health
```

The expected response is whatever your health endpoint returns, such as `{"status":"ok"}`.

Inspect the socket and directory permissions:

```bash
stat /run/web_app
stat /run/web_app/gunicorn.sock
namei -l /run/web_app/gunicorn.sock
id www-data
```

If the socket is missing, check the Supervisor status and logs first. If the socket exists but requests fail with permission errors, inspect every directory in the path, the socket's owner and group, its mode, and Nginx's effective group membership.

**Important:** Do not solve socket access problems by making the entire project world-writable or by adding Nginx to your personal user group. Grant access only to the dedicated socket path and its required parent directories.

### 5.8 Diagnose processes safely

Inspect the running processes:

```bash
ps -eo pid,ppid,user,rss,args | grep '[g]unicorn'
```

Check the process Supervisor manages:

```bash
sudo supervisorctl pid web_app
```

Do not assume that an old process or a process with parent PID 1 is automatically safe to kill. Inspect its full command, parent process, service ownership, and relationship to Supervisor.

**Avoid this as a routine fix:**

```bash
sudo pkill -f gunicorn
```

That command can terminate Gunicorn processes belonging to other applications, interrupt active requests, and cause unnecessary downtime. Prefer `sudo supervisorctl restart web_app` for the application Supervisor manages. If unrelated or orphaned processes exist, identify each one before stopping it.

## 6. Configure the frontend and API routes

### 6.1 Use a relative API URL

Configure the frontend to call the API through the same origin as the website. For example, if your Axios configuration is in `main.js`:

```javascript
import axios from "axios";

axios.defaults.baseURL = "/";
```

This setting is suitable only if your API calls include their intended paths, such as `/api/search`. If your project already uses a configured Axios instance, update that instance instead of adding a second global configuration.

Using a relative URL allows the browser to send requests to the current host rather than hard-coding a production IP address or domain into the frontend.

### 6.2 Configure Vite for local development

In `vite.config.js`, add a development proxy while preserving the project's existing configuration:

```javascript
export default defineConfig({
  // Keep your existing configuration.

  server: {
    port: 8080,
    proxy: {
      "/api": {
        target: "http://127.0.0.1:8000",
        changeOrigin: true,
      },
    },
  },
});
```

This tells Vite's development server to forward matching API requests to the local FastAPI development server.

Do not expose the Vite development server as your production web server. Nginx will handle production requests.

### 6.3 Review CORS configuration

If the frontend and API are served from the same scheme, host, and port in production, browser requests are same-origin and generally do not require CORS middleware for that production flow.

However, local development, separate frontend/API domains, and third-party integrations may still require CORS. Do not remove `CORSMiddleware` without checking every environment and client that uses the API.

If CORS is required, explicitly configure the expected origins, methods, and headers. Avoid wildcard origins when using credentialed requests.

### 6.4 Rebuild the frontend

Frontend configuration is often embedded in the built JavaScript, so rebuild after changing the API URL or other build-time settings:

```bash
cd /web_app/frontend
pnpm install --frozen-lockfile
pnpm run build
```

If the project uses a different locked dependency workflow, follow its documented process. Confirm the output exists:

```bash
test -f /web_app/frontend/dist/index.html && echo "Frontend build found"
```

## 7. Install and configure Nginx

### 7.1 Install Nginx

Install Nginx:

```bash
sudo apt update
sudo apt install -y nginx
```

Check its status:

```bash
sudo systemctl status nginx --no-pager
```

On Ubuntu, the main configuration is `/etc/nginx/nginx.conf`. Site-specific configurations are commonly stored in `/etc/nginx/sites-available/` and enabled through symbolic links in `/etc/nginx/sites-enabled/`.

### 7.2 Create the site configuration

Create a configuration file:

```bash
sudo nano /etc/nginx/sites-available/web_app
```

The configuration below serves the Vue frontend and forwards `/api` requests to Gunicorn through the Unix socket. Replace `example.com` with your actual domain. If you have only an IP address, use the appropriate server-name setting until DNS is configured.

```nginx
upstream web_app_backend {
    server unix:/run/web_app/gunicorn.sock;
}

server {
    listen 80;
    listen [::]:80;

    server_name example.com;

    access_log /var/log/nginx/web_app-access.log;
    error_log  /var/log/nginx/web_app-error.log;

    root /web_app/frontend/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Exact /api request: preserve the URI sent to FastAPI.
    location = /api {
        proxy_pass http://web_app_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # API paths such as /api/health and /api/search.
    location ^~ /api/ {
        proxy_pass http://web_app_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**How this works:**

- `upstream` gives the backend socket a reusable name.
- `root` identifies the built frontend directory.
- `try_files` supports client-side routes in a single-page application.
- `location = /api` handles the exact `/api` path.
- `location ^~ /api/` handles API paths below `/api/`.
- `proxy_pass` has no URI suffix, so Nginx preserves the requested URI when forwarding it to the upstream.
- The forwarded headers provide request information to the backend. Configure FastAPI's proxy-header trust to match your actual proxy topology; do not trust arbitrary clients to supply forwarded headers.

Nginx's URI behavior depends on the combination of `location` and `proxy_pass` syntax. Test both `/api` and `/api/` paths and confirm that FastAPI routes match the forwarded paths.

### 7.3 Check Nginx file access

Nginx must be able to read the frontend files and traverse every parent directory. It must also be able to connect to the Gunicorn socket.

Check the frontend path:

```bash
namei -l /web_app/frontend/dist/index.html
```

Check the socket path:

```bash
namei -l /run/web_app/gunicorn.sock
```

Correct only the permissions that are missing. Do not recursively grant broad permissions to the whole application tree.

### 7.4 Enable the site

Create a symbolic link:

```bash
sudo ln -s /etc/nginx/sites-available/web_app \
  /etc/nginx/sites-enabled/web_app
```

If the link already exists, inspect it rather than creating another one.

Before removing the default site, check whether the server already hosts another application that depends on it. If it is only the unused default placeholder, disable it:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

Test the configuration before applying it:

```bash
sudo nginx -t
```

Only if the test succeeds, reload Nginx:

```bash
sudo systemctl reload nginx
```

Check service status:

```bash
sudo systemctl status nginx --no-pager
```

## 8. Configure the firewall and test the deployment

### 8.1 Allow the required web ports

If UFW is active and you intend to serve a public website, allow the Nginx profile:

```bash
sudo ufw allow 'Nginx Full'
sudo ufw status verbose
```

`Nginx Full` allows standard HTTP on port 80 and HTTPS on port 443. It does **not** configure a TLS certificate or enable HTTPS by itself.

Keep the SSH access rule from Module 1. Do not enable UFW blindly if the required SSH rule has not been verified.

If your cloud provider also has a network firewall, make sure its rules align with the intended public access. The operating-system firewall and cloud firewall are separate controls.

### 8.2 Test the website

Open your configured domain or server IP in a browser:

```text
http://example.com
```

Confirm that the Vue frontend loads. Then test a feature that calls the backend API.

From the server, you can also test HTTP responses:

```bash
curl -I http://127.0.0.1/
curl -i http://127.0.0.1/api/health
```

The second request tests Nginx routing locally when Nginx is configured to accept requests for that host. If necessary, include the intended Host header or test using your domain.

### 8.3 Troubleshoot HTTP 502 errors

A `502 Bad Gateway` response often means Nginx cannot connect to the upstream backend or the backend returned an invalid response.

Inspect the Nginx error log:

```bash
sudo tail -n 100 /var/log/nginx/web_app-error.log
```

Inspect the Supervisor status and logs:

```bash
sudo supervisorctl status web_app
sudo supervisorctl tail -100 web_app
```

Test the socket directly:

```bash
curl --unix-socket /run/web_app/gunicorn.sock \
  http://localhost/api/health
```

Common causes include:

- Gunicorn is not running.
- The socket path in Nginx does not match the path used by Gunicorn.
- Nginx cannot traverse the socket directory or access the socket.
- The FastAPI application fails during startup.
- The requested route does not exist or does not match the expected API prefix.

If the error is `Permission denied`, inspect directory traversal permissions, socket owner/group/mode, and Nginx's group membership. If the error is `No such file or directory`, check whether the socket exists and whether Supervisor successfully started Gunicorn.

`fail_timeout=0` is not a guarantee of uninterrupted service. If the backend is unavailable, API requests may still return 502 errors until the service is restored.

## 9. Apply HTTP security headers

Security does not stop at the firewall. HTTP response headers instruct browsers to apply additional security restrictions. They must be configured for the application's actual behavior, and they do not replace TLS, authentication, authorization, input validation, or secure coding.

### 9.1 Start with headers appropriate to the application

The following headers are commonly useful:

```nginx
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=()" always;
```

Their purposes are:

- **`X-Content-Type-Options: nosniff`** tells browsers not to guess a different MIME type from the one declared.
- **`X-Frame-Options: SAMEORIGIN`** limits framing to the same origin, helping mitigate clickjacking. Use a compatible `frame-ancestors` directive in CSP as well when appropriate.
- **`Referrer-Policy`** limits how much referrer information is sent to other sites.
- **`Permissions-Policy`** disables the listed browser features. Remove or adjust restrictions if the application legitimately needs them.

Add `Cross-Origin-Resource-Policy` only after considering how the site serves images, fonts, and other resources. For example, `same-origin` is restrictive, while `cross-origin` allows resources to be loaded by other origins. Neither value is automatically correct for every website.

### 9.2 Configure Content Security Policy carefully

A Content Security Policy (CSP) restricts which sources the browser can use for scripts, styles, fonts, images, frames, and connections. It can reduce the impact of certain cross-site scripting (XSS) attacks, but the policy must match the application.

For this example, a possible starting policy is:

```nginx
add_header Content-Security-Policy "default-src 'self'; base-uri 'self'; object-src 'none'; frame-ancestors 'self'; font-src 'self' https://fonts.gstatic.com https://cdnjs.cloudflare.com; img-src 'self' data:; script-src 'self'; style-src 'self' 'unsafe-inline' https://cdnjs.cloudflare.com https://fonts.googleapis.com; connect-src 'self'; frame-src 'self';" always;
```

This is a **starting example, not a universal policy**. Check the application's real resource requirements before enforcing it. For example:

- `default-src 'self'` restricts resource loading to the same origin unless another directive permits a source.
- `object-src 'none'` blocks embedded plugin content.
- `base-uri 'self'` restricts the document's base URL.
- `frame-ancestors 'self'` restricts which origins can embed the site.
- `font-src`, `img-src`, `style-src`, `script-src`, and `connect-src` control different resource types.
- `style-src 'unsafe-inline'` allows inline styles and weakens CSP protection for styles. Avoid it if the application can use nonces, hashes, or external stylesheets instead.

Do not allow external script or style domains simply because they appear in a browser error. Verify that each source is required and trusted.

### 9.3 Test CSP before enforcing it

A safer rollout is to start with a `Content-Security-Policy-Report-Only` header and observe violations while testing the application. For example:

```nginx
add_header Content-Security-Policy-Report-Only "default-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'self';" always;
```

This example is only a starting policy. A report-only policy does not block resources, so it allows you to identify required sources before enforcement. Browser developer tools can show CSP violations during testing.

After testing all key pages, API calls, fonts, images, scripts, authentication flows, and any third-party integrations, replace the report-only header with a carefully reviewed enforcing policy. Remove the report-only header when it is no longer needed.

### 9.4 Configure HTTPS before HSTS

**Do not deploy HSTS as though it secures a site that only serves HTTP.** HSTS is effective only when received over HTTPS.

First, configure HTTPS using a trusted certificate and confirm that the website and API work over HTTPS. Then consider adding a header such as:

```nginx
add_header Strict-Transport-Security "max-age=31536000" always;
```

Start with a conservative policy. Do not add `includeSubDomains` or `preload` until every affected subdomain supports HTTPS and you understand the long-term implications. HSTS can make browsers refuse HTTP access for the specified period.

Likewise, do not add CSP's `upgrade-insecure-requests` directive until HTTPS is working and all required resources are available over HTTPS. On an HTTP-only site, this directive can cause resources to fail to load.

### 9.5 Understand Nginx header inheritance

Nginx's `add_header` inheritance can change when headers are defined in nested `location` blocks. A configuration that adds headers at the server level may not behave as expected if a location defines its own `add_header` directives.

After adding or changing headers:

1. Test the configuration with `sudo nginx -t`.
2. Reload Nginx only if the test succeeds.
3. Test both frontend and API responses, including error responses.
4. Inspect the browser's Network panel and the response headers.
5. Use a header-scanning service as an additional check, not as proof that the entire application is secure.

A high scanner grade does not establish that authentication, authorization, dependencies, secrets, application logic, or infrastructure are secure.

## 10. Final Nginx configuration

After you have verified your socket path, domain, frontend build, and chosen headers, your Nginx site configuration should follow this general structure. Replace the example domain and tailor the CSP and optional headers to the application. Add the HSTS header only after HTTPS is configured and verified.

```nginx
upstream web_app_backend {
    server unix:/run/web_app/gunicorn.sock;
}

server {
    listen 80;
    listen [::]:80;

    server_name example.com;

    access_log /var/log/nginx/web_app-access.log;
    error_log  /var/log/nginx/web_app-error.log;

    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=()" always;

    # Review and test this policy before enforcement.
    add_header Content-Security-Policy "default-src 'self'; base-uri 'self'; object-src 'none'; frame-ancestors 'self'; font-src 'self' https://fonts.gstatic.com https://cdnjs.cloudflare.com; img-src 'self' data:; script-src 'self'; style-src 'self' 'unsafe-inline' https://cdnjs.cloudflare.com https://fonts.googleapis.com; connect-src 'self'; frame-src 'self';" always;

    root /web_app/frontend/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location = /api {
        proxy_pass http://web_app_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location ^~ /api/ {
        proxy_pass http://web_app_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

After editing the file:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Only reload after the configuration test succeeds.

**Note:** This site block listens on HTTP port 80. It does not itself configure HTTPS, certificate renewal, or HTTP-to-HTTPS redirection. Add those after setting up and verifying TLS.

## 11. Final verification checklist

Run through this checklist before considering the deployment complete.

### Application and process manager

```bash
sudo supervisorctl status web_app
sudo supervisorctl tail -100 web_app
```

### Backend socket and API

```bash
stat /run/web_app/gunicorn.sock
namei -l /run/web_app/gunicorn.sock

curl --unix-socket /run/web_app/gunicorn.sock \
  http://localhost/api/health
```

### Nginx

```bash
sudo nginx -t
sudo systemctl status nginx --no-pager
sudo tail -n 50 /var/log/nginx/web_app-error.log
```

### Network exposure and firewall

```bash
sudo ufw status verbose
sudo ss -tulpn
```

Confirm that the backend is not listening on a public interface. With the Unix-socket configuration shown in this module, Gunicorn should use the socket rather than expose port 8000.

### Browser and headers

- Confirm that the frontend loads.
- Test API requests and routes such as `/api/health`.
- Check the browser console for failed resources and CSP violations.
- Inspect response headers on frontend and API responses.
- Confirm that HTTP and HTTPS behave as intended after TLS is configured.
- Verify that application logs do not expose passwords, tokens, or other sensitive data.

## 12. Deployment and recovery principles

For safer future deployments:

1. Keep a known-good release or backup so you can roll back.
2. Build from controlled dependencies and a reviewed source revision.
3. Keep secrets outside the source archive and protect them with restrictive permissions.
4. Run the application as a non-root account.
5. Keep the backend private and expose it through Nginx.
6. Use the process manager to control the intended application instead of killing processes by name.
7. Test Nginx configuration before every reload.
8. Configure TLS before enabling HSTS or HTTPS-dependent policies.
9. Review logs and resource usage after deployment.
10. Test recovery after a restart or reboot, including recreation of the runtime socket directory.

## Conclusion

In this module, you deployed a full-stack application on Ubuntu, built the Vue frontend, configured FastAPI under Gunicorn and Supervisor, and connected the services through Nginx. You also established a safer approach to process management, Unix-socket permissions, reverse proxy configuration, and browser security headers.

The key principle is to **deploy with least privilege, verify every service boundary, and apply security controls only after confirming that the underlying configuration works**.

In the next module, you can extend the server with another service, such as a self-hosted search engine, while applying the same principles of service isolation, restricted permissions, and verification.
