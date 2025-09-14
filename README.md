# cloudflared
Step-by-step instructions to configure a local service to run as a web app over cloudflare tunneling

- I have a service at [http://localhost:8001](http://localhost:8001/)

- I have a cloudflare domain name something.org

- I use Ubuntu 24.04

- I want to access my service at testit.something.org

- I want to use cloudflared tunneling

### Step 1: Set Up Your Subdomain in Cloudflare

**Goal:** Add a DNS record for testit.something.org in your Cloudflare dashboard.

#### Instructions step 1

1. **Log into Cloudflare Dashboard**

   - Go to https://dash.cloudflare.com

   - Select your domain: something.org

2. **Add a CNAME DNS Record**

   - Navigate to the **DNS** tab

   - Click **Add record**

      - **Type**: CNAME

      - **Name**: testit

      - **Target**: your-tunnel-name.cloudflareTunnel.com _(well fill this later, for now use a placeholder like_ placeholder.cloudflareTunnel.com_)_

      - **TTL**: Auto

      - **Proxy status**: Leave it **Proxied (orange cloud)**

   >This step allows Cloudflare to route traffic for testit.something.org once your tunnel is ready.

#### Verification step 1

You should see a new DNS record like:

```txt
Type: CNAME | Name: testit | Target: placeholder.cloudflareTunnel.com | Proxied: Yes
```

### Step 2: Install cloudflared on Ubuntu 24.04

We'll install the official package to ensure you get updates and stability.

#### Instructions step 2

1. **Install the Cloudflare package repository**
   ```sh
   # just to be sure
   sudo apt install curl lsb-release
   ```

   ```sh
   curl -L https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-archive-keyring.gpg >/dev/null
   echo "deb [signed-by=/usr/share/keyrings/cloudflare-archive-keyring.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee  /etc/apt/sources.list.d/cloudflared.list 
   # would this work on Ubuntu? this should be like this:
   # sudo mkdir -p /etc/apt/keyrings
   # curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /etc/apt/keyrings/cloudflare-main.gpg > /dev/null
   # echo 'deb \[signed-by=/etc/apt/keyrings/cloudflare-main.gpg\] https://pkg.cloudflare.com/cloudflared focal main' | sudo tee /etc/apt/sources.list.d/cloudflared.list 
   ```

2. **Update your package list and install** cloudflared

   ```sh
   sudo apt update
   sudo apt install cloudflared
   ```

3. **Verify installation**

   ```sh
   cloudflared --version
   ```

#### Verification step 2

You should see output like:

```txt
cloudflared version 2025.x.x (built ...)
```

### Step 3: Authenticate with Cloudflare

This step connects your local machine to your Cloudflare account via your browser. You need to be logged in to cloudflare for this to work. If you are not logged in the browser which will pop up, and log in, then the selector will not be shown.

#### Instructions step 3

Run the following command:

```sh
cloudflared tunnel login
```

- This will open a browser window. (If a login window pops up, the workflow might not work. Log in, go back to the terminal, and restart try again.)

- It will show you a list of your domains.

- Select something.org.

Once done, cloudflared will save a certificate to:

~/.cloudflared/cert.pem

This certificate is used to authorize creation and management of tunnels.

#### Verification step 3

After the command finishes, you should see a message like:

```txt
You have successfully logged in.
Your certificate has been saved to /home/your-user/.cloudflared/cert.pem
```

### Step 4: Create a Tunnel

We'll create a tunnel named testit-tunnel.

#### Instructions step 4

1. Run this command to create the tunnel:

   ```sh
   cloudflared tunnel create testit-tunnel
   ```

   This will:

   - Create a tunnel credential file (e.g., ~/.cloudflared/testit-tunnel.json)

   - Register the tunnel with Cloudflare

2. Note the tunnel ID that gets printed - something like:

   ```txt
   Created tunnel testit-tunnel with id: 12345678-aaaa-bbbb-cccc-123456abcdef
   ```

   Keep this tunnel name consistent, we'll use it in the configuration next.

#### Verification step 4

You should see output confirming:

- Tunnel was created

- A .json credentials file was saved under ~/.cloudflared

For example:

```sh
$ cloudflared tunnel create testit-tunnel

Tunnel credentials written to /home/user/.cloudflared/8e137011-ae9a-4800-90b8-f4ca6e2dcda3.json. cloudflared chose this file based on where your origin certificate was found. Keep this file secret. To revoke these credentials, delete the tunnel. Created tunnel testit-tunnel with id 8e137011-ae9a-4800-90b8-f4ca6e2dcda3
```

### Step 5: Create the config.yml for the Tunnel

Well create the file in ~/.cloudflared/config.yml

#### Instructions step 5

1. Open the config file in your editor:

   ```sh
   nano ~/.cloudflared/config.yml
   ```

2. Paste the following contents (replace with your actual tunnel ID):

   ```txt
   tunnel: 8e137011-ae9a-4800-90b8-f4ca6e2dcda3
   credentials-file: /home/user/.cloudflared/8e137011-ae9a-4800-90b8-f4ca6e2dcda3.json
   ingress:
    - hostname: testit.something.org
      service: http://localhost:8001
    - service: http\_status:404
   ```

3. Save and exit:

- Press CTRL+O then Enter to save

- Press CTRL+X to exit

#### Verification step 5

You should have a file at ~/.cloudflared/config.yml with:

- The correct tunnel ID

- A hostname: testit.something.org

- A service pointing to http://localhost:8001

### Step 6: Create a Tunnel Route (Cloudflare Zero Trust)

This connects your domain (testit.something.org) to the tunnel on the Cloudflare side.

#### Instructions step 6

1. Run this command to create the route:

   ```sh
   cloudflared tunnel route dns testit-tunnel testit.something.org
   ```

This tells Cloudflare:

Send traffic for testit.something.org through the tunnel testit-tunnel.

Once this runs, Cloudflare will **update the DNS record automatically** to point to your tunnel (youll see the CNAME target change to something like 8e137011-ae9a-...cfargotunnel.com in the dashboard).

#### Verification setp 6

 The command should finish with something like:

```txt
Success! Route added: testit.something.org -> testit-tunnel
```

You can check in Cloudflare DNS panel: the testit subdomain should now be a CNAME pointing to a \*.cfargotunnel.com value.

### Step 7: Start the Tunnel and Test Access

Lets now start the tunnel using your config file and test everything.

#### Instructions step 7

Run this command:

```sh
cloudflared tunnel run testit-tunnel
```

You should see logs like:

```txt
INF Starting tunnel
INF Connection ... registered connIndex=0
Leave this running for now to keep the tunnel open.
```

#### Test Access

1. Open your browser

2. Go to:  
https://testit.something.org

You should see the service you had running at http://localhost:8001.

If it doesn't load:

- Check your local service is running on port 8001

- Check for errors in the cloudflared output

### About the Warnings You Might See

The messages you're seeing are **non-critical warnings** related to **ICMP proxying** (used for features like ping over the tunnel). Here's a quick breakdown:

WRN ... cloudflared process has a GID ... not within ping\_group\_range

WRN ICMP proxy feature is disabled

This means:

- Cloudflared is trying to enable **ICMP (ping) proxy support**, but the user it runs as isn't allowed.

- Since your use case is HTTP (localhost:8096)  **this has no effect** on your tunnel functionality.

**Safe to ignore** unless you specifically need ping/ICMP forwarding, which 99% of users do not.

### Step 8: Run cloudflared as a Systemd Service

Before installing and starting the systemd service, you should **stop the manually running tunnel** so there's no conflict.

You need to move config and credentials to /etc/cloudflared

#### Step 8.1: Create the config directory (if not already)

```sh
sudo mkdir -p /etc/cloudflared
```

#### Step 8.2: Move your config and credentials file there

```sh
sudo cp ~/.cloudflared/config.yml /etc/cloudflared/

sudo cp ~/.cloudflared/9e137011-ae9a-4800-80b8-f4ca6e2dcda3.json /etc/cloudflared/
```

These are the two files referenced in your config:

a. /etc/cloudflared/config.yml

b. /etc/cloudflared/9e137011-ae9a-4800-80b8-f4ca6e2dcda3.json

#### Step 8.3: Update permissions (just in case)

```sh
sudo chown -R root:root /etc/cloudflared
sudo chmod 600 /etc/cloudflared/\*
```

#### Step 8.4: Install the systemd service

Run:

```sh
sudo cloudflared service install
```

This will:

- Copy your tunnel config

- Set up a systemd unit under /etc/systemd/system/cloudflared.service

You should see:

```txt
2025-... INF Adding systemd service for running tunnel
```

#### Step 8.5: Start and enable the service

```sh
sudo systemctl enable --now cloudflared
```

This:

- Enables the service to start at boot

- Starts it immediately

#### Verification step 8

Run:

```sh
sudo systemctl status cloudflared
```

You should see:

Active: active (running)

And some logs showing that your tunnel started successfully.

### Final Quick Checks

1. **Try browsing again**

   Visit https://testit.something.org to confirm your service is still reachable.

2. **Manage your tunnel service**

   Restart:

   ```sh
   sudo systemctl restart cloudflared
   ```

   View logs (live):

   ```sh
   sudo journalctl -fu cloudflared
   ```

   Stop:

   ```sh
   sudo systemctl stop cloudflared
   ```
