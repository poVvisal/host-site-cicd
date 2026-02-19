# Static Site CI/CD: GitHub → EC2 (Nginx)

Reusable checklist for auto-deploying this static site from `main` to an EC2 instance running Nginx.

## 1) One-time setup on EC2

### Security group
- Allow `TCP 22 (SSH)` from your IP.
- Allow `TCP 80 (HTTP)` from anywhere.

### Install and start Nginx
```bash
sudo apt update && sudo apt -y upgrade
sudo apt -y install nginx
sudo systemctl enable --now nginx
```

### Verify
- Open `http://EC2_PUBLIC_IP`.
- You should see the default Nginx page.

## 2) One-time setup in GitHub

### Repository
- Keep your static site files in the repo root (for example: `index.html`, `assets/`, `css/`, `js/`).
- Push to `main`.

### GitHub Actions secrets
Add these in **Repo → Settings → Secrets and variables → Actions**:
- `EC2_HOST` = EC2 public IP
- `EC2_USER` = `ubuntu`
- `EC2_SSH_KEY` = full private key content (PEM)

## 3) Workflow file

Create/update `.github/workflows/deploy.yml`:

```yaml
name: Deploy static site to EC2

on:
  push:
    branches: ["main"]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Copy site to EC2 (/var/www/html)
        uses: appleboy/scp-action@v1
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          source: "./*"
          target: "/var/www/html"
          strip_components: 0

      - name: Reload nginx
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            sudo chown -R www-data:www-data /var/www/html
            sudo nginx -t
            sudo systemctl reload nginx
```

## 4) Day-to-day deploy flow

After making site changes:

```bash
git add .
git commit -m "Update site"
git push origin main
```

GitHub Actions deploys automatically. Refresh `http://EC2_PUBLIC_IP` to see updates.

## 5) Quick troubleshooting

- If deployment fails, check **Actions tab → latest run logs**.
- If SSH fails, re-check `EC2_HOST`, `EC2_USER`, and `EC2_SSH_KEY` secrets.
- If site does not update, confirm files exist in `/var/www/html` and Nginx reload passed.