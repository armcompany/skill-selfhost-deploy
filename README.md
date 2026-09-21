# Selfhost Deploy Skill

[![skills.sh](https://skills.sh/b/armcompany/skill-selhost-deploy)](https://www.skills.sh/armcompany/skill-selhost-deploy/selfhost-deploy)

Skill para auditar, planejar, implantar e diagnosticar projetos self-hosted com Docker/Compose em macOS + Colima ou VPS Linux, preservando serviços existentes e oferecendo opções explícitas de acesso, banco, proxy, HTTPS e persistência.

Install:

```sh
npx skills add armcompany/skill-selhost-deploy --skill selfhost-deploy
```

Cobre stacks com frontend web + API + banco/cache/telemetria ("stack comum" e "stack com telemetria"), acesso local, privado por Tailscale ou público por domínio com HTTPS.