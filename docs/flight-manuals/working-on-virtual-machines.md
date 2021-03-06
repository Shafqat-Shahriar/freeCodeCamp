# Flight Manual for working on Virtual Machines

As a member of the staff or the dev-team, you may have been given access to our cloud service providers like Azure, Digital Ocean, etc.

Here are some handy commands that you can use to work on the Virtual Machines (VM), for instance performing maintenance updates or doing general houeskeeping.

# Get a list of the VMs

> [!NOTE]
> While you may already have SSH access to the VMs, that alone will not let you list VMs unless you been granted access to the cloud portals as well.

## Azure

Install Azure CLI `az`: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli

> **(One-time) Install on macOS with [`homebrew`](https://brew.sh):**

```
brew install azure-cli
```

> **(One-time) Login:**

```
az login
```

> **Get the list of VM names and IP addresses:**

```
az vm list-ip-addresses --output table
```

## Digital Ocean

Install Digital Ocean CLI `doctl`: https://github.com/digitalocean/doctl#installing-doctl

> **(One-time) Install on macOS with [`homebrew`](https://brew.sh):**

```
brew install doctl
```

> **(One-time) Login:**

Authentication and context switching: https://github.com/digitalocean/doctl#authenticating-with-digitalocean

```
doctl auth init
```

> **Get the list of VM names and IP addresses:**

```
doctl compute droplet list --format "ID,Name,PublicIPv4"
```

# Keeping VMs Updated

You should keep the VMs up to date by performing updates and upgrades. This will ensure that the virtual machine is patched with latest security fixes.

> [!WARNING]
> Before you run these commands:
>
> - Make sure that the VM has been provisioned completely and there is no post-install steps running.
> - If you are updating packages on a VM that is already serving an application, make sure the app has been stopped / saved. Package updates will cause network bandwidth, memory and/or CPU usage spikes leading to outages on running applications.

Update package information

```console
sudo apt update
```

Upgrade installed packages

```console
sudo apt upgrade -y
```

Cleanup unused packages

```console
sudo apt autoremove -y
```

# Work on Web Servers (Proxy)

We are running load balanced (Azure Load Balancer) instances for our web servers. These servers are running NGINX which reverse proxy all of the traffic to freeCodeCamp.org from various applications running on their own infrastructures.

The NGINX config is available on [this repository](https://github.com/freeCodeCamp/nginx-config).

## First install

### 0. Prerequisites (workspace Setup) for Staff

Get a login session on azure cli, and clone the `cloud-setup` (private repo) for setting up template workspace.

```console
az login
git clone cloud-setup
cd cloud-setup
```

### 1. Provision VMs on Azure.

List all Resource Groups

```console
az group list --output table
```

```console
Name                               Location       Status
---------------------------------  -------------  ---------
tools-rg                           eastus         Succeeded
```

Create a Resource Group

```
az group create --location eastus --name stg-rg-eastus
```

```console
az group list --output table
```

```console
Name                               Location       Status
---------------------------------  -------------  ---------
tools-rg                           eastus         Succeeded
stg-rg-eastus                      eastus         Succeeded
```

Next per the need, provision a single VM or a scaleset.

#### A. provision single instances

```console
az vm create \
  --resource-group stg-rg-eastus \
  --name <VIRTUAL_MACHINE_NAME> \
  --image UbuntuLTS \
  --custom-data cloud-init/nginx-cloud-init.yaml \
  --admin-username <USERNAME> \
  --ssh-key-values <SSH_KEYS>.pub
```

#### B. provision scaleset instance

```console
az vmss create \
  --resource-group stg-rg-eastus \
  --name <VIRTUAL_MACHINE_SCALESET_NAME> \
  --image UbuntuLTS \
  --upgrade-policy-mode automatic \
  --custom-data cloud-init/nginx-cloud-init.yaml \
  --admin-username <USERNAME> \
  --ssh-key-values <SSH_KEYS>.pub
```

> [!NOTE]
> The custom-data config should allow you to configure and add SSH keys, install packages etc. via the cloud-init templates in your local workspace. Tweak the files in your local workspace as needed. The cloud-init config is optional and you can omit it completely to do setups manually as well.

### 2. (Optional) Install NGINX and configure from repository.

The basic setup should be ready OOTB, via the cloud-init configuration. SSH and make changes as necessary for the particular instance(s).

If you did not use the cloud-init config previously use the below for manual setup of NGINX and error pages:

```console
sudo su

cd /var/www/html
git clone https://github.com/freeCodeCamp/error-pages

cd /etc/
rm -rf nginx
git clone https://github.com/freeCodeCamp/nginx-config nginx

cd /etc/nginx
```

### 3. Install Cloudflare origin certificates and upstream application config.

Get the Cloudflare origin certificates from the secure storage and install at required locations.

**OR**

Move over existing certificates:

```console
# Local
scp -r username@source-server-public-ip:/etc/nginx/ssl ./
scp -pr ./ssl username@target-server-public-ip:/tmp/

# Remote
rm -rf ./ssl
mv /tmp/ssl ./
```

<details>

<summary> Custom workflow with managed keys and hosts (Mrugesh) </summary>

```console
# Local
scp -r -i ~/.ssh/id_rsa_fcc source-server-hostname:/etc/nginx/ssl ./
scp -pr -i ~/.ssh/id_rsa_fcc ./ssl target-server-hostname:/tmp/

# Remote
rm -rf ./ssl
mv /tmp/ssl ./
```
</details>

Update Upstream Configurations:

```console
vi configs/upstreams.conf
```

Add/update the source/origin application IP addresses.

### 4. Setup networking and firewalls.

Configure Azure firewalls and `ufw` as needed for ingress origin addresses.

### 5. Add the VM to the load balancer backend pool.

Configure and add rules to load balancer if needed. You may also need to add the VMs to load balancer backend pool if needed.

## Logging and Monitoring

1. Check status for NGINX service using the below command:

```console
sudo systemctl status nginx
```

2. Logging and monitoring for the servers are available at:

> <h3 align="center"><a href='https://amplify.nginx.com' _target='blank'>https://amplify.nginx.com</a></h3>

## Updating Instances (Maintenance)

Config changes to our NGINX instances are maintained on GitHub, these should be deployed on each instance like so:

1. SSH into the instance and enter sudo

```console
sudo su
```

2. Get the latest config code.

```console
cd /etc/nginx
git fetch --all --prune
git reset --hard origin/master
```

3. Test and reload the config [with Signals](https://docs.nginx.com/nginx/admin-guide/basic-functionality/runtime-control/#controlling-nginx).

```console
nginx -t
nginx -s reload
```

# Work on API Instances

> **Todo: Add VM setup and installation details**

1. Install build tools for node binaries (`node-gyp`) etc.

```console
sudo apt install build-essential
```

## First Install

Provisioning VMs with the Code

1. Install Node LTS.

2. Update `npm` and install PM2 and setup logrotate and startup on boot

   ```console
   npm i -g npm
   npm i -g pm2
   pm2 install pm2-logrotate
   pm2 startup
   ```

3. Clone freeCodeCamp, setup env and keys.

   ```console
   git clone https://github.com/freeCodeCamp/freeCodeCamp.git
   cd freeCodeCamp
   ```

4. Create the `.env` from the secure credentials storage.

5. Install dependencies

   ```console
   npm ci
   ```

6. Build the server

   ```console
   npm run ensure-env && npm run build:server
   ```

7. Start Instances

   ```console
   cd api-server
   pm2 start production-start.js -i max --max-memory-restart 600M --name org
   ```

## Logging and Monitoring

```console
pm2 logs
```

```console
pm2 monitor
```

## Updating Instances (Maintenance)

Code changes need to be deployed to the API instances from time to time. It can be a rolling update or a manual update. The later is essential when changing dependencies or adding enviroment variables.

> [!DANGER]
> The automated pipelines are not handling dependencies updates at the minute. We need to do a manual update before any deployment pipeline runs.

### 1. Manual Updates - Used for updating dependencies, env variables.

1. Stop all instances

```console
pm2 stop all
```

2. Install dependencies

```console
npm ci
```

3. Build the server

```console
npm run ensure-env && npm run build:server
```

4. Start Instances

```console
pm2 start all --update-env && pm2 logs
```

### 2. Rolling updates - Used for logical changes to code.

```console
pm2 reload all --update-env && pm2 logs
```

> [!NOTE]
> We are handling rolling updates to code, logic, via pipelines. You do not need to run these commands. These are here for documentation.
