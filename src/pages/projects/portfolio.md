---
layout: '../../layouts/Project.astro'
title: Portfolio Website
description: "This very static website you're reading right now."
badge: "Website"
tags: ["Astro", "Tailwind", "Caddy"]
priority: 5
---

## Continuous Deployment via GitHub Actions

Updating the static frontend portfolio (built on Astro) does not involve dragging files over SFTP like it is 2004.

```yaml
name: Deploy to Self-Hosted NAS
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Source
        uses: actions/checkout@v4

      - name: Connect to Mesh
        uses: tailscale/github-action@v2
        with:
          authkey: ${{ secrets.TAILSCALE_AUTHKEY }}

      - name: Build & Rsync Static Payload
        run: |
          npm ci
          npm run build
          rsync -avz --delete ./dist/ deploy@nas.internal:/var/www/osquijrdev/
```

Every `git push` to `main` spins up an ephemeral runner, authenticates onto the private Tailscale tailnet using an ephemeral auth key, compiles the Astro production bundle, and rsyncs the build artifact directly into the web root on the NAS.

If a build fails, it fails in GitHub's container, not in production.

---