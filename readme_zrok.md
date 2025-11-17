# Zrok Setup and Configuration Guide

This guide walks you through setting up zrok, reserving a public share, and running it as a persistent systemd service.

## Prerequisites

- Linux system with systemd
- Internet connection
- Root/sudo access for systemd configuration

## Step 1: Install Zrok

Download and install zrok from the official repository:

```bash
# Download zrok from official website

# Extract the archive
tar -xzf zrok_1.1.10_linux_amd64.tar.gz

# Move to system path
sudo mv zrok /usr/local/bin/
sudo chmod +x /usr/local/bin/zrok

# Verify installation
zrok version
```

## Step 2: Enable Zrok

Before using zrok, you need to enable it with your account token:

```bash
zrok enable <your-token>
```

You can get your token from the zrok web console after signing up at https://zrok.io

## Step 3: Login to Zrok (if required)

If you need to authenticate:

```bash
zrok login
```

Follow the prompts to complete authentication.

## Step 4: Reserve a Public Share

Reserve a public share endpoint with your backend configuration:

### For Reverse Proxy Applications (Nginx, Apache, etc.)

Use `--backend-mode proxy` when your application is behind a reverse proxy:

```bash
zrok reserve public 192.168.4.186:80 --backend-mode proxy
```

**When to use proxy mode:**
- Applications running behind Nginx, Apache, Traefik, or other reverse proxies
- Multiple services hosted on the same backend
- Custom routing and SSL termination handled by your proxy

### For Direct Web Applications

Use `--backend-mode web` for direct web server access:

```bash
zrok reserve public 192.168.4.186:80 --backend-mode web
```

**When to use web mode:**
- Standalone web applications (Node.js, Python Flask/Django, Go servers)
- Development servers running directly on a port
- Applications that don't require reverse proxy processing
- Simple static file servers

### Reserve Output

After running the reserve command, you'll see output like:

```
[   6.150]    INFO main.(*reserveCommand).run: your reserved share token is 'lpqbksxoultv'
[   6.150]    INFO main.(*reserveCommand).run: reserved frontend endpoint: https://lpqbksxoultv.share.zrok.io
```

**Important:** Save your share token (e.g., `lpqbksxoultv`) - you'll need it for the next steps.

## Step 5: Test the Share

Before setting up the service, test that sharing works:

```bash
zrok share reserved lpqbksxoultv --headless
```

You should see your application accessible at the provided URL. Press `Ctrl+C` to stop.

## Step 6: Create Systemd Service

Create a systemd service file to run zrok persistently in the background:

```bash
sudo nano /etc/systemd/system/zrok.service
```

Add the following configuration:

```ini
[Unit]
Description=Zrok Share Service
After=network.target

[Service]
Type=simple
User=piva
WorkingDirectory=/home/piva
ExecStart=/usr/local/bin/zrok share reserved lpqbksxoultv --headless
Restart=always
RestartSec=10
StandardOutput=null
StandardError=null

[Install]
WantedBy=multi-user.target
```

**Configuration notes:**
- Replace `piva` with your actual username
- Replace `lpqbksxoultv` with your actual share token
- Adjust `WorkingDirectory` to your home directory

## Step 7: Enable and Start the Service

Reload systemd, enable the service to start on boot, and start it:

```bash
# Reload systemd daemon
sudo systemctl daemon-reload

# Enable service to start on boot
sudo systemctl enable zrok

# Start the service now
sudo systemctl start zrok
```

## Step 8: Verify Service Status

Check that the service is running:

```bash
sudo systemctl status zrok
```

You should see `active (running)` in the output.

## Managing the Service

### Check Service Status
```bash
sudo systemctl status zrok
```

### Stop the Service
```bash
sudo systemctl stop zrok
```

### Restart the Service
```bash
sudo systemctl restart zrok
```

### View Service Logs
```bash
sudo journalctl -u zrok -f
```

### Disable Service from Starting on Boot
```bash
sudo systemctl disable zrok
```

## Troubleshooting

### Service fails to start
- Check logs: `sudo journalctl -u zrok -n 50`
- Verify zrok path: `which zrok`
- Ensure your share token is correct
- Check that the backend service (e.g., Nginx) is running

### Cannot access the public URL
- Verify the service is running: `sudo systemctl status zrok`
- Check your backend application is accessible locally
- Ensure firewall allows the backend port

### Permission issues
- Make sure the `User` in the service file matches your username
- Verify zrok is enabled for that user

## Additional Resources

- Zrok Documentation: https://docs.zrok.io
- Zrok GitHub: https://github.com/openziti/zrok
- Community Support: https://openziti.discourse.group

## Summary

Your zrok share is now running persistently in the background as a systemd service. It will:
- ✅ Start automatically on system boot
- ✅ Restart automatically if it crashes
- ✅ Run without terminal logs cluttering your screen
- ✅ Continue running even when you close your terminal

Access your application at: `https://lpqbksxoultv.share.zrok.io` (replace with your actual URL)
