# Selfhost Deploy Skill

[![skills.sh](https://skills.sh/b/armcompany/skill-selhost-deploy)](https://www.skills.sh/armcompany/skill-selhost-deploy/selfhost-deploy)

Skill for auditing, planning, deploying, and diagnosing self-hosted projects with Docker/Compose on macOS + Colima or Linux VPS, preserving existing services and offering explicit choices for access, database, proxy, HTTPS, and persistence.

Install:

```sh
npx skills add armcompany/skill-selhost-deploy --skill selfhost-deploy
```

Covers stacks with web frontend + API + database/cache/telemetry ("common stack" and "stack with telemetry"), local access, private via Tailscale, or public via a domain with HTTPS. Also covers deploy automation on git push via GitHub webhook + Tailscale Funnel.