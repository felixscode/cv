# Deployment Setup

This document describes how to set up automatic deployment to your Hetzner VM.

## Overview

The CV automatically deploys to your Hetzner VM at `188.245.151.245` whenever changes are pushed to the `main` branch.

## Prerequisites

1. SSH access to your Hetzner VM (root@188.245.151.245)
2. Git installed on the server
3. The repository cloned at `/var/www/cv` on the server
4. Caddy configured to serve from `/var/www/cv`

## Server Setup

### 1. Clone the repository on your Hetzner VM

```bash
ssh root@188.245.151.245
cd /var/www
git clone https://github.com/felixscode/cv.git
cd cv
git checkout main
```

### 2. Ensure proper permissions

```bash
chown -R root:root /var/www/cv
chmod -R 755 /var/www/cv
```

### 3. Configure Caddy

Make sure your Caddyfile includes a configuration for your CV domain:

```caddy
your-domain.com {
    root * /var/www/cv
    file_server
    encode gzip

    # Optional: Add security headers
    header {
        Strict-Transport-Security "max-age=31536000;"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        Referrer-Policy "no-referrer-when-downgrade"
    }
}
```

Then reload Caddy:
```bash
systemctl reload caddy
```

## GitHub Secrets Setup

You need to configure the following secrets in your GitHub repository:

### 1. Generate SSH Key (if you don't have one)

On your local machine or server:

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_deploy_key
```

This creates two files:
- `~/.ssh/github_deploy_key` (private key)
- `~/.ssh/github_deploy_key.pub` (public key)

### 2. Add public key to server

Copy the public key to your Hetzner VM:

```bash
ssh-copy-id -i ~/.ssh/github_deploy_key.pub root@188.245.151.245
```

Or manually add it to `/root/.ssh/authorized_keys` on the server.

### 3. Add secrets to GitHub

Go to your GitHub repository:
- Navigate to **Settings** → **Secrets and variables** → **Actions**
- Click **New repository secret**

Add the following secrets:

| Secret Name | Value | Description |
|-------------|-------|-------------|
| `SSH_HOST` | `188.245.151.245` | Your Hetzner VM IP address |
| `SSH_USERNAME` | `root` | SSH username |
| `SSH_PRIVATE_KEY` | Contents of `~/.ssh/github_deploy_key` | The entire private key file |
| `SSH_PORT` | `22` | SSH port (default is 22) |

**Important:** When adding `SSH_PRIVATE_KEY`, copy the entire contents including:
```
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```

## Testing the Deployment

1. Make a small change to your CV
2. Commit and push to the `main` branch:
   ```bash
   git add .
   git commit -m "Test deployment"
   git push origin main
   ```
3. Go to **Actions** tab in your GitHub repository
4. Watch the deployment workflow run
5. Verify the changes are live on your domain

## Troubleshooting

### Deployment fails with SSH authentication error

- Verify the SSH private key is correctly added to GitHub secrets
- Ensure the public key is in `/root/.ssh/authorized_keys` on the server
- Check SSH port is correct (default: 22)

### Git pull fails on server

- Ensure the repository exists at `/var/www/cv`
- Check that the server has network access to GitHub
- Verify git is installed: `git --version`

### Changes not visible after deployment

- Check Caddy is serving the correct directory
- Verify file permissions: `ls -la /var/www/cv`
- Clear browser cache or try in incognito mode
- Check Caddy logs: `journalctl -u caddy -n 50`

## Manual Deployment

If you need to deploy manually:

```bash
ssh root@188.245.151.245
cd /var/www/cv
git pull origin main
systemctl reload caddy  # Optional
```

## Security Notes

- Keep your SSH private key secure and never commit it to the repository
- Consider using a dedicated deployment user instead of root
- Regularly rotate SSH keys
- Monitor deployment logs for suspicious activity
- Consider adding IP restrictions to SSH access
