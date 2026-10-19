# Deploying SUTRA on an Azure VM (Azure for Students)

One Ubuntu VM runs the whole stack with Docker Compose, and Caddy in front of it gets an HTTPS
certificate for the VM's free Azure DNS name. This is [Option A](README.md#option-a-one-linux-server-with-docker-compose)
of the deployment guide with every setting filled in.

**Cost** (Central India list prices, from the Azure Retail Prices API, October 2026):

| Item | Price |
|---|---|
| VM `Standard_B2as_v2` (2 vCPU, 8 GiB) | about $36/month while running |
| OS disk, 64 GiB Standard SSD (E6) | about $5.30/month |
| Static public IPv4 (Standard) | about $3.70/month |

That is about $45/month if the VM runs all the time, so the $100 student credit lasts a little over
two months. A stopped (deallocated) VM costs only the disk and IP, about $9/month.

## 1. Create the VM

Azure portal → **Virtual machines** → **Create** → **Azure virtual machine**.

**Basics**

| Setting | Value |
|---|---|
| Subscription | Azure for Students |
| Resource group | **Create new** → `sutra-demo-rg` |
| Virtual machine name | `sutra-demo` |
| Region | (Asia Pacific) Central India |
| Availability options | No infrastructure redundancy required |
| Security type | Trusted launch virtual machines (the default) |
| Image | Ubuntu Server 24.04 LTS - x64 Gen2 |
| VM architecture | x64 |
| Run with Azure Spot discount | Off |
| Size | **See all sizes** → search `B2as_v2` → `Standard_B2as_v2` (2 vCPUs, 8 GiB) |
| Authentication type | SSH public key |
| Username | `azureuser` |
| SSH public key source | Generate new key pair |
| Key pair name | `sutra-demo_key` |
| Public inbound ports | Allow selected ports |
| Select inbound ports | SSH (22), HTTP (80), HTTPS (443) |

**Disks**

| Setting | Value |
|---|---|
| OS disk size | 64 GiB |
| OS disk type | Standard SSD (locally-redundant storage) |
| Delete with VM | On |

**Networking**: keep the new virtual network, subnet and public IP Azure suggests, keep the NIC
security group on **Basic** with the ports above, and turn on **Delete public IP and NIC when VM is
deleted**.

**Management** (optional, saves credit): turn on **Auto-shutdown**, for example at 01:00 in
(UTC+05:30) Chennai, Kolkata, Mumbai, New Delhi. It stops the VM every night; start it again from
the portal before a demo.

Leave the other tabs as they are, then **Review + create** → **Create** → **Download private key
and create resource**. Keep the downloaded `sutra-demo_key.pem`: it is the only way to sign in.

If creation fails with *"Resource was disallowed by Azure"*, your student subscription only allows
certain regions; the error lists them. Pick the closest one (for example South India) and keep every
other setting. If `B2as_v2` is not offered in that region, pick another region rather than a bigger size.

## 2. Give it a web address

1. Open the VM → **Overview** → **DNS name**: *Not configured* → set **DNS name label** to something
   unique, such as `sutra-demo-yourname` → **Save**.
2. Your address is now `sutra-demo-yourname.centralindia.cloudapp.azure.com` (with your region's
   name if you picked another one).
3. Recommended: VM → **Networking** → the inbound rule for port 22 → **Source**: *My IP address* →
   **Save**, so only you can reach SSH.

## 3. Connect from your Mac

```bash
mkdir -p ~/.ssh && mv ~/Downloads/sutra-demo_key.pem ~/.ssh/ && chmod 400 ~/.ssh/sutra-demo_key.pem
```

```bash
ssh -i ~/.ssh/sutra-demo_key.pem azureuser@sutra-demo-yourname.centralindia.cloudapp.azure.com
```

Every command from here on runs **on the VM**.

## 4. Set up the server

Your address, used by the commands below (change it to yours):

```bash
echo 'export SUTRA_HOST=sutra-demo-yourname.centralindia.cloudapp.azure.com' >> ~/.bashrc && source ~/.bashrc
```

**Updates and 4 GB of swap** (headroom for the image build, which trains the models):

```bash
sudo apt update && sudo apt -y upgrade && sudo apt install -y git
sudo fallocate -l 4G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

**Docker Engine and Compose** ([official instructions](https://docs.docker.com/engine/install/ubuntu/)):

```bash
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
```

Then `exit` and connect again with the same `ssh` command, so `docker` works without `sudo`.

**Caddy** ([official instructions](https://caddyserver.com/docs/install#debian-ubuntu-raspbian)):

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo chmod o+r /usr/share/keyrings/caddy-stable-archive-keyring.gpg
sudo chmod o+r /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install -y caddy
```

**The code, its secrets, and restart-on-boot**:

```bash
git clone https://github.com/rokadepiyush49-rgb/Sutra.git ~/sutra && cd ~/sutra
cat > .env <<EOF
SUTRA_JWT_SECRET=$(python3 -c 'import secrets; print(secrets.token_urlsafe(48))')
SUTRA_PII_HMAC_KEY=$(python3 -c 'import secrets; print(secrets.token_urlsafe(48))')
SUTRA_DEMO_PASSWORD=$(python3 -c 'import secrets; print(secrets.token_urlsafe(16))')
EOF
chmod 600 .env
cat > docker-compose.override.yml <<'EOF'
# Server only: bring the app back after the VM restarts. The seed job is left out on purpose,
# so a restart keeps cases and notes instead of re-seeding the demo world.
services:
  postgres: { restart: unless-stopped }
  valkey: { restart: unless-stopped }
  api: { restart: unless-stopped }
  web: { restart: unless-stopped }
EOF
```

You never need these secrets yourself: the demo personas sign in without a password.

**Build and start** (one image at a time keeps memory in check; the first build takes a while,
because the API image trains the models):

```bash
cd ~/sutra && docker compose build api && docker compose build web && docker compose up -d
```

```bash
docker compose ps
```

`migrate` and `seed` should show *Exited (0)*; `postgres`, `valkey`, `api` and `web` *healthy*.

**HTTPS**:

```bash
sudo tee /etc/caddy/Caddyfile >/dev/null <<EOF
$SUTRA_HOST {
	reverse_proxy 127.0.0.1:3000
}
EOF
sudo systemctl reload caddy
```

Caddy gets a Let's Encrypt certificate on the first request, which needs ports 80 and 443 open (step 1).

## 5. Check it

```bash
curl -fsS https://$SUTRA_HOST/api/health
```

It should print `{"status":"ok","service":"sutra-web"}`. Then open `https://<your address>` in a
browser and pick a demo persona.

## Day to day

| Task | How |
|---|---|
| Save credit | Portal → VM → **Stop**. Wait for *Stopped (deallocated)*: only then does compute billing stop |
| Start again | Portal → VM → **Start**. The app comes back on its own in about a minute, with its data |
| See logs | `cd ~/sutra && docker compose logs -f api` (or `web`) |
| Reset the demo data | `cd ~/sutra && docker compose run --rm seed` |
| Deploy the latest code | `cd ~/sutra && git pull && docker compose build api && docker compose build web && docker compose up -d` (this re-seeds the demo world) |
| Check the credit left | Portal → **Cost Management + Billing** |

## Good to know

- Demo mode is on: anyone with the link can sign in as any persona, and the mock bank is open.
  Share the link only with people who should see the demo, and keep the data synthetic.
- Only ports 22, 80 and 443 are open. The API (8000) and Postgres (5433) listen on `127.0.0.1` only.
- Tested before writing this: the Compose stack behind Caddy on a Mac (Docker Desktop, `tls internal`
  instead of Let's Encrypt). Sign-in sets `Secure`, `HttpOnly` cookies, the BFF reaches the API, and
  HSTS and CSP headers come through. The Azure steps themselves have not been run end to end yet.
