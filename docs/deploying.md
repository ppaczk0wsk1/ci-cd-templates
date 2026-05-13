# Deploying to Your Server

[< Back to README](../README.md)

---

## Overview

GitHub Actions runs in isolated VMs that don't have access to your server by default. To deploy, you need to give the runner a way to authenticate. This guide covers the most common approaches.

## SSH Deploy (most common for VPS / bare metal)

This is the typical setup for deploying to a server you manage (DigitalOcean, Hetzner, AWS EC2, any VPS).

### 1. Generate a dedicated SSH key pair

On your local machine (not the server), create a key pair specifically for CI. Don't reuse your personal key.

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/deploy_key -N ""
```

This creates two files:
- `~/.ssh/deploy_key` (private key - goes into GitHub Secrets)
- `~/.ssh/deploy_key.pub` (public key - goes onto your server)

### 2. Add the public key to your server

Copy the public key to the server's `authorized_keys`:

```bash
# Option A: ssh-copy-id (if you have existing access)
ssh-copy-id -i ~/.ssh/deploy_key.pub youruser@yourserver.com

# Option B: manual
cat ~/.ssh/deploy_key.pub | ssh youruser@yourserver.com "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

For better security, create a dedicated deploy user with limited permissions:

```bash
# On your server
sudo adduser deploy --disabled-password
sudo mkdir -p /home/deploy/.ssh
sudo cp ~/.ssh/authorized_keys /home/deploy/.ssh/authorized_keys
sudo chown -R deploy:deploy /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
sudo chmod 600 /home/deploy/.ssh/authorized_keys
```

### 3. Add the private key to GitHub Secrets

1. Go to your repo on GitHub
2. Settings > Secrets and variables > Actions > New repository secret
3. Name: `SSH_PRIVATE_KEY`
4. Value: paste the entire contents of `~/.ssh/deploy_key` (including the `-----BEGIN` and `-----END` lines)
5. Add these secrets too:
   - `SSH_HOST` - your server's IP or hostname (e.g., `203.0.113.50`)
   - `SSH_USER` - the user to log in as (e.g., `deploy`)

### 4. Add the deploy job to your workflow

Uncomment the `deploy` job in your workflow and replace it with:

```yaml
deploy:
  runs-on: ubuntu-latest
  needs: [ci]
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  environment: production

  steps:
    - uses: actions/checkout@v6

    # Set up SSH access to your server.
    # This writes the private key to a file and configures SSH to use it.
    - name: Setup SSH
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/deploy_key
        chmod 600 ~/.ssh/deploy_key

        # Add the server to known_hosts so SSH doesn't prompt.
        # ssh-keyscan fetches the server's host key.
        ssh-keyscan -H ${{ secrets.SSH_HOST }} >> ~/.ssh/known_hosts

    # Run deploy commands on the server over SSH.
    - name: Deploy
      run: |
        ssh -i ~/.ssh/deploy_key ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST }} << 'EOF'
          cd /var/www/myapp
          git pull origin main
          npm install --production
          npm run build
          pm2 restart myapp
        EOF

     # Docker deploy
     # - name: Deploy
     #   run: |
     #     ssh -i ~/.ssh/deploy_key ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST }} << 'EOF'
     #       cd ${{ secrets.DEPLOY_DIR }}
     #
     #       echo ${{ secrets.GHCR_PAT }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin
     #
     #       docker compose pull
     #       docker compose up -d --remove-orphans
     #     EOF
```

Adjust the commands inside the `EOF` block to match your project (docker compose up, systemctl restart, etc.).

### 5. (Recommended) Set up environment protection

1. Go to Settings > Environments > New environment
2. Name it `production` (must match the `environment:` value in the workflow)
3. Enable "Required reviewers" if you want manual approval before deploys
4. Optionally restrict which branches can deploy (e.g., only `main`)

## Alternative: rsync deploy

If you want to upload built files rather than running git on the server:

```yaml
- name: Deploy via rsync
  run: |
    rsync -avz --delete \
      -e "ssh -i ~/.ssh/deploy_key" \
      ./dist/ \
      ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST }}:/var/www/myapp/
```

## Alternative: Docker deploy

If your server runs Docker and you've set up the Docker job from the templates:

```yaml
- name: Deploy
  run: |
    ssh -i ~/.ssh/deploy_key ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST }} << 'EOF'
      docker pull ghcr.io/youruser/yourrepo:latest
      docker compose up -d
    EOF
```

For this to work, the server also needs access to pull from GHCR. You can set that up with:

```bash
# On the server, log into GHCR using a personal access token with read:packages scope
echo "YOUR_PAT" | docker login ghcr.io -u YOUR_GITHUB_USER --password-stdin
```

## Cloud Platform Deploys

For managed platforms, you typically don't need SSH. Instead, you install their CLI and use a deploy token.

### Vercel

```yaml
- name: Deploy to Vercel
  run: npx vercel --prod --token=${{ secrets.VERCEL_TOKEN }}
```

Secret needed: `VERCEL_TOKEN` (get it from Vercel dashboard > Settings > Tokens).

### Fly.io

```yaml
- name: Deploy to Fly.io
  uses: superfly/flyctl-actions/setup-flyctl@master
- run: flyctl deploy --remote-only
  env:
    FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
```

Secret needed: `FLY_API_TOKEN` (run `flyctl tokens create deploy` locally).

### AWS (ECS, S3, Lambda)

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1

- name: Deploy to ECS
  run: |
    aws ecs update-service \
      --cluster production \
      --service myapp \
      --force-new-deployment
```

Secrets needed: `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`. For better security, use OIDC instead of long-lived keys (see [AWS docs on GitHub OIDC](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)).

## Security Tips

- **Never commit private keys or tokens** to your repo. Always use GitHub Secrets.
- **Use a dedicated deploy key** with minimal permissions. Don't reuse your personal SSH key.
- **Create a deploy user** on your server with only the permissions it needs (no sudo).
- **Use environment protection rules** to require manual approval for production deploys.
- **Rotate keys periodically.** If a key is compromised, revoke it in GitHub Secrets and on the server.
- **Prefer OIDC over long-lived tokens** for AWS, GCP, and Azure. OIDC tokens are short-lived and scoped to specific repos/branches.
- **Restrict the deploy user's SSH access** with `ForceCommand` or `command=` in `authorized_keys` to limit what the CI runner can do on your server.

## Secrets Reference

| Secret | Used for | Where to get it |
|---|---|---|
| `SSH_PRIVATE_KEY` | SSH deploy | `ssh-keygen` (see step 1 above) |
| `SSH_HOST` | SSH deploy | Your server's IP or hostname |
| `SSH_USER` | SSH deploy | The user to SSH as (e.g., `deploy`) |
| `VERCEL_TOKEN` | Vercel deploy | Vercel dashboard > Settings > Tokens |
| `FLY_API_TOKEN` | Fly.io deploy | `flyctl tokens create deploy` |
| `AWS_ACCESS_KEY_ID` | AWS deploy | IAM console |
| `AWS_SECRET_ACCESS_KEY` | AWS deploy | IAM console |
| `GITHUB_TOKEN` | Docker push to GHCR | Automatic (no setup needed) |
