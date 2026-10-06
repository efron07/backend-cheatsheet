# 20. Deployment and DevOps

## 20.1 Docker

**What it is:** Docker packages your app with its exact runtime (Node version, OS libraries) into an **image**. A running image is a **container**. It runs the same on your laptop, CI and any server.

```dockerfile
# Dockerfile — multi-stage: small, secure production image
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:22-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build && npm prune --omit=dev

FROM node:22-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
RUN apk add --no-cache tini
COPY --from=build --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist
COPY --from=build --chown=node:node /app/drizzle ./drizzle
COPY --from=build --chown=node:node /app/package.json ./
USER node                                  # never run as root
EXPOSE 4000
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:4000/health/live || exit 1
ENTRYPOINT ["/sbin/tini", "--"]            # forwards SIGTERM properly for graceful shutdown
CMD ["node", "dist/server.js"]
```

```
# .dockerignore
node_modules
dist
.env
.git
coverage
```

```bash
docker build -t pharmacare-api:1.0.0 .
docker run -p 4000:4000 --env-file .env pharmacare-api:1.0.0
docker ps; docker logs -f <id>; docker exec -it <id> sh; docker stop <id>
```

## 20.2 docker compose for local development

```yaml
# compose.yaml — the whole backend locally with one command
services:
  api:
    build: .
    command: npx tsx watch src/server.ts
    ports: ['4000:4000']
    env_file: .env
    environment:
      DATABASE_URL: postgresql://postgres:secret@postgres:5432/pharmacare
      MONGODB_URI: mongodb://mongo:27017/pharmacare
      REDIS_URL: redis://redis:6379
    volumes: ['./src:/app/src']
    depends_on: [postgres, mongo, redis]

  worker:
    build: .
    command: npx tsx watch src/jobs/worker.ts
    env_file: .env
    environment:
      DATABASE_URL: postgresql://postgres:secret@postgres:5432/pharmacare
      REDIS_URL: redis://redis:6379
    depends_on: [redis, postgres]

  postgres:
    image: postgres:17
    environment: { POSTGRES_PASSWORD: secret, POSTGRES_DB: pharmacare }
    ports: ['5432:5432']
    volumes: ['pgdata:/var/lib/postgresql/data']

  mongo:
    image: mongo:8
    ports: ['27017:27017']
    volumes: ['mongodata:/data/db']

  redis:
    image: redis:7
    ports: ['6379:6379']

volumes: { pgdata: {}, mongodata: {} }
```

```bash
docker compose up -d          # start everything
docker compose logs -f api
docker compose down           # stop (add -v to delete data volumes)
```

In production you can use **Neon** and **Atlas** instead of the database containers; only the API, worker and maybe Redis run on your host.

## 20.3 CI/CD with GitHub Actions

**CI** (continuous integration): every push runs type-check, lint and tests. **CD** (continuous delivery/deployment): every merge to `main` builds and deploys automatically.

```yaml
# .github/workflows/deploy.yml
name: CI/CD
on:
  push: { branches: [main] }
  pull_request:

jobs:
  test:
    uses: ./.github/workflows/ci.yml        # the test job from section 17.6

  migrate-and-deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production                 # requires approval + holds production secrets
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - name: Run database migrations
        run: npx drizzle-kit migrate
        env: { DATABASE_URL_UNPOOLED: '${{ secrets.DATABASE_URL_UNPOOLED }}' }
      - name: Build and push image
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .
          docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
      - name: Deploy
        run: curl -fsS -X POST "${{ secrets.DEPLOY_HOOK_URL }}"   # your host's deploy hook, or SSH/flyctl/railway CLI
```

**Neon branch per pull request:** the `neondatabase/create-branch-action` creates a database branch for each PR, runs migrations and tests against it, and a matching delete action cleans up when the PR closes.

## 20.4 Migrations on deploy

1. Run migrations **once**, as a separate step **before** the new code starts (not in every container's startup).
2. Migrations must be **backward-compatible** with the code still running (section 10.4: expand → migrate code → contract).
3. Take a backup or Neon restore point before risky migrations.
4. Have a rollback plan: forward-fix migrations are usually safer than down-migrations.

## 20.5 Hosting options

| Option | Type | Notes |
| --- | --- | --- |
| **Railway**, **Render**, **Fly.io**, **Koyeb** | Platform (PaaS) | Push Docker or code; managed TLS, logs, scaling, cron and workers. Fastest path |
| **DigitalOcean App Platform**, **Heroku** | PaaS | Similar |
| **VPS** (Hetzner, DigitalOcean Droplet, Contabo, a local Tanzanian data centre) | Your own Linux server | Cheapest and full control; you manage OS, security, backups. Use **Coolify** or **Dokploy** for a PaaS-like UI on your VPS |
| **AWS / GCP / Azure** (ECS, Cloud Run, App Service) | Cloud | Most powerful; most complex |
| **Vercel / Netlify functions** | Serverless | Fine for light APIs; Express works but long-running workers and WebSockets do not |
| **Kubernetes** | Container orchestration | Many services, large teams; overkill for one API |

**Data location:** keep the API close to its databases (same region as Neon and Atlas) — every query crosses that network. Check legal requirements for storing health data.

## 20.6 Deploying on a VPS with PM2 and Nginx (without Docker)

```bash
# on the server (Ubuntu)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash - && sudo apt install -y nodejs nginx
sudo npm i -g pm2
git clone git@github.com:efron07/pharmacare-api.git && cd pharmacare-api
npm ci && npm run build
pm2 start dist/server.js --name pharmacare-api -i max      # cluster mode: one process per CPU core
pm2 start dist/jobs/worker.js --name pharmacare-worker
pm2 save && pm2 startup                                     # restart on reboot
pm2 logs; pm2 monit; pm2 reload pharmacare-api              # zero-downtime reload
```

```nginx
# /etc/nginx/sites-available/api.pharmacare.co.tz
server {
  server_name api.pharmacare.co.tz;
  client_max_body_size 5m;

  location / {
    proxy_pass http://127.0.0.1:4000;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Upgrade $http_upgrade;          # WebSockets
    proxy_set_header Connection "upgrade";
    proxy_buffering off;                             # SSE
  }
}
```

```bash
sudo certbot --nginx -d api.pharmacare.co.tz      # free HTTPS certificate (Let's Encrypt)
sudo ufw allow OpenSSH && sudo ufw allow 'Nginx Full' && sudo ufw enable   # firewall
```

## 20.7 Environments and release practices

- **Environments:** local → preview/staging (Neon branch, test M-Pesa sandbox) → production. Same code, different config.
- **Versioning and releases:** tag releases (`v1.4.0`), keep a changelog, deploy small changes often.
- **Zero-downtime deploys:** rolling restarts behind a load balancer + graceful shutdown + backward-compatible migrations.
- **Rollback:** keep the previous image; redeploy it in one command.
- **Feature flags:** ship code switched off, turn it on for one branch first (Unleash, GrowthBook, or a simple DB table).
- **Infrastructure as code** (Terraform, Pulumi, SST) once you have more than a couple of resources.
