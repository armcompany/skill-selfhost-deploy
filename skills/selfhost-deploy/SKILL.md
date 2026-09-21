---
name: selfhost-deploy
description: Use when deploying, planning, or diagnosing self-hosted projects with Docker/Compose on macOS + Colima or Linux VPS, preserving existing services and offering explicit choices for access, database, proxy, HTTPS, and persistence.
---

# selfhost-deploy

## Objetivo

Implantar projetos de forma repetível e segura em infraestrutura self-hosted, especialmente:

- Mac Apple Silicon com macOS + Colima + Docker;
- VPS Linux com Docker;
- múltiplos projetos no mesmo host;
- frontend web + API + banco/cache/telemetria;
- acesso local, privado por Tailscale ou público por domínio.

A skill deve **investigar antes de alterar**. Nunca presumir nomes de variáveis, portas, banco, framework, estrutura do repositório ou estratégia de migrations.

---

## Princípios obrigatórios

1. **Não derrubar serviços existentes.**
   - Antes de criar containers, proxy ou portas, inspecionar o host.
   - Não executar `docker compose down`, `docker-compose down`, `kill`, `pkill`, `rm`, `down -v` ou remoção de volumes de outro projeto sem autorização explícita.

2. **Não inventar configuração.**
   - Descobrir como a aplicação realmente lê configuração.
   - Procurar `DATABASE_URL`, `DB_HOST`, `ConfigService`, Prisma, TypeORM, Sequelize, Redis, Vite envs etc.

3. **Separar os três contextos de rede.**
   - Container → container: nome do serviço Docker, ex. `postgres:5432`.
   - Host → container: `localhost:<porta-publicada>`.
   - Outro dispositivo → host: IP/hostname Tailscale, IP público ou domínio.

4. **Banco e cache ficam privados por padrão.**
   - Não publicar PostgreSQL, Redis, Timescale etc. no host se não houver necessidade real.

5. **Persistência é obrigatória para dados.**
   - Bancos, Redis persistente, uploads e outros dados precisam de volumes/bind mounts adequados.

6. **Produção não depende de `synchronize=true`.**
   - Preferir migrations.
   - `synchronize` só pode ser oferecido como bootstrap temporário de banco novo e vazio, com aviso explícito.

7. **Secrets nunca vão para Git.**
   - Usar `.env.production`, secret manager ou equivalente.
   - Garantir `.gitignore`.

8. **Frontend Vite é build-time.**
   - Mudança em `VITE_*` exige rebuild.
   - Não usar `localhost` como URL de API quando o frontend será aberto em outro dispositivo.

9. **HTTPS e API devem preferencialmente compartilhar origem.**
   - Preferir `/api` atrás de reverse proxy a URLs/portas diferentes.
   - Evitar mixed content e reduzir CORS.

10. **Toda implantação termina com validação.**
    - containers;
    - health endpoint;
    - banco;
    - frontend;
    - frontend → API;
    - persistência;
    - restart;
    - acesso externo escolhido.

---

# Fluxo operacional

## Fase 0 — Determinar o alvo

Identificar:

- projeto/repositório;
- host de destino;
- sistema operacional e arquitetura;
- se já há outros projetos rodando;
- se o usuário quer apenas preparar arquivos ou executar o deploy.

Se o usuário não escolher o nível de exposição, apresentar:

**Opção A — Local**
- acesso somente no host.

**Opção B — Privado via Tailscale HTTP**
- rápido para equipe/homologação.

**Opção C — Privado via Tailscale HTTPS**
- Tailscale Serve e hostname `*.ts.net`.

**Opção D — Público com domínio + HTTPS**
- reverse proxy central, DNS e TLS.

Não escolher silenciosamente uma opção quando isso alterar exposição ou segurança.

---

# Fase 1 — Auditoria do host

Antes de escolher portas:

```bash
docker ps
docker ps -a
sudo lsof -nP -iTCP -sTCP:LISTEN
```

Em macOS + Colima:

```bash
colima status
docker --version
docker-compose --version || docker compose version
```

Se necessário, identificar processos:

```bash
ps -p PID -o pid,ppid,command
lsof -a -p PID -d cwd
```

Registrar:

- portas ocupadas;
- containers existentes;
- nomes já usados;
- reverse proxies existentes;
- Tailscale;
- recursos relevantes do host.

Não modificar serviços encontrados apenas porque parecem conflitantes. Primeiro escolher outra porta ou apresentar a decisão.

---

# Fase 2 — Auditoria do projeto

Inspecionar antes de escrever Compose.

Arquivos prioritários:

```text
package.json
package-lock.json / pnpm-lock.yaml / yarn.lock
Dockerfile*
docker-compose*.yml
.env*
src/**/configuration*
src/**/database*
prisma/schema.prisma
vite.config.*
next.config.*
README*
```

Descobrir:

- frontend e framework;
- backend e framework;
- comando de build;
- comando de runtime;
- porta interna;
- endpoint de health;
- banco;
- cache;
- filas;
- storage;
- WebSockets;
- serviços externos;
- migrations;
- variáveis obrigatórias.

Para NestJS/TypeORM, procurar:

```bash
grep -R "TypeOrmModule\|DATABASE_URL\|DB_HOST\|ConfigService\|config.get" src --include="*.ts"
```

Para envs:

```bash
grep -R "process.env" src --include="*.ts"
```

Para Vite:

```bash
grep -R "VITE_" . --exclude-dir=node_modules --exclude-dir=dist
```

Não assumir que `DATABASE_HOST` existe se o projeto usa `DATABASE_URL`.

---

# Fase 3 — Escolher arquitetura

## Opção A — Aplicação simples

```text
frontend ou API
└── container
```

## Opção B — Frontend + API

```text
frontend
API
```

## Opção C — Stack comum

```text
frontend
API
PostgreSQL/PostGIS
Redis
```

## Opção D — Stack com telemetria

```text
frontend
API
PostgreSQL/PostGIS
Redis
TimescaleDB
```

## Opção E — Dependências externas

Manter banco/cache/storage gerenciados externamente quando o projeto já depender deles e a migração não tiver sido solicitada.

---

# Fase 4 — Estratégia de banco

Apresentar a opção aplicável.

## A. Banco novo com migrations

Preferida.

```text
models/entities
→ migrations
→ banco
```

Executar somente migrations compatíveis com o projeto.

## B. Bootstrap temporário

Somente para banco novo, vazio e quando o framework suporta criação automática.

Exemplo TypeORM:

```ts
synchronize: env !== 'production'
```

Procedimento:

1. confirmar banco vazio;
2. ativar sincronização temporariamente;
3. iniciar API;
4. verificar tabelas;
5. voltar imediatamente a `NODE_ENV=production`;
6. recriar API;
7. planejar migrations para mudanças futuras.

Nunca manter essa opção como padrão de produção.

## C. Banco existente

- obter dump/backup;
- restaurar;
- verificar extensões;
- validar schema;
- executar migrations pendentes somente depois.

Não sobrescrever banco existente sem confirmação.

---

# Fase 5 — Dockerfiles

## Backend Node/NestJS base

Adaptar ao projeto real:

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
ENV PORT=3000
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=build /app/dist ./dist
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

Se o lockfile não estiver confiável, não forçar `npm ci`; estabilizar dependências primeiro.

NestJS deve normalmente ouvir:

```ts
await app.listen(process.env.PORT || 3000, '0.0.0.0');
```

## Frontend Vite base

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runtime
WORKDIR /app
RUN npm install -g serve
COPY --from=build /app/dist ./dist
EXPOSE 3000
CMD ["sh", "-c", "serve -s dist -l ${PORT:-3000}"]
```

Se houver Nginx/Caddy já adotado pelo projeto, preservar a arquitetura quando adequada.

---

# Fase 6 — Compose

Gerar nomes exclusivos por projeto.

Exemplo conceitual:

```yaml
services:
  api:
    build:
      context: .
    container_name: PROJECT-api
    restart: unless-stopped
    environment:
      NODE_ENV: production
      PORT: 3000
      DATABASE_URL: postgresql://app:${POSTGRES_PASSWORD}@postgres:5432/app
    ports:
      - "HOST_API_PORT:3000"
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgis/postgis:16-3.4
    container_name: PROJECT-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: app
    volumes:
      - PROJECT_pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  PROJECT_pgdata:
```

Adicionar Redis/Timescale somente quando o projeto realmente usar.

Entre containers:

```text
postgres:5432
redis:6379
timescale:5432
```

Nunca:

```text
localhost:5432
```

para comunicação container → container.

---

# Fase 7 — Portas

Se não houver reverse proxy central, reservar portas sem conflito.

Exemplo de convenção:

```text
Projeto A  front 8080  api 3000
Projeto B  front 8081  api 3001
Projeto C  front 8082  api 3002
```

A convenção é exemplo, não obrigação. Sempre verificar o host antes.

Banco/cache não recebem porta pública por padrão.

---

# Fase 8 — Frontend → API

## Opção A — Navegador no próprio host

```env
VITE_API_URL=http://localhost:3000
```

## Opção B — Outro dispositivo via Tailscale

```env
VITE_API_URL=http://TAILSCALE_IP:API_PORT
```

O dispositivo cliente precisa alcançar a Tailnet.

## Opção C — Reverse proxy / mesma origem — preferida

```env
VITE_API_URL=/api
```

Topologia:

```text
https://host/
├── /      → frontend
└── /api   → backend
```

Após alterar qualquer `VITE_*`, rebuildar a imagem do frontend.

---

# Fase 9 — Exposição

## A. Local

Validar com:

```bash
curl http://localhost:PORT/health
```

## B. Tailscale HTTP

Acessar:

```text
http://TAILSCALE_IP:PORT
```

ou hostname MagicDNS quando aplicável.

## C. Tailscale HTTPS

No macOS com app Tailscale, o CLI pode estar em:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale
```

Exemplo:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale serve --bg http://localhost:8080
```

Status:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale serve status
```

Desativar configuração HTTPS na porta 443:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale serve --https=443 off
```

Antes de substituir uma configuração Serve existente, inspecionar `serve status`.

## D. Domínio público + HTTPS

Preferir reverse proxy central:

```text
Internet
→ 443
→ Caddy/Nginx/Traefik
→ projeto
```

Para múltiplos projetos:

```text
app-a.example.com → A
app-b.example.com → B
app-c.example.com → C
```

Não expor banco/cache.

---

# Fase 10 — Secrets

Criar arquivo de produção separado, por exemplo:

```env
POSTGRES_PASSWORD=...
JWT_SECRET=...
```

Garantir:

```gitignore
.env.production
.env*.local
```

Não imprimir secrets completos em relatórios ou logs.

---

# Fase 11 — Deploy

Escolher o comando compatível com o host.

Standalone:

```bash
docker-compose \
  --env-file .env.production \
  -f docker-compose.prod.yml \
  up -d --build
```

Plugin moderno:

```bash
docker compose \
  --env-file .env.production \
  -f docker-compose.prod.yml \
  up -d --build
```

Não trocar automaticamente entre ambos se um deles já foi validado no host.

---

# Fase 12 — Validação obrigatória

## Containers

```bash
docker ps
```

## Compose

```bash
docker-compose -f docker-compose.prod.yml ps
```

## Logs

```bash
docker-compose -f docker-compose.prod.yml logs --tail=100 api
```

## API

```bash
curl -i http://localhost:API_PORT/health
```

## Banco

```bash
docker exec PROJECT-postgres pg_isready -U USER -d DATABASE
```

Quando necessário:

```bash
docker exec -it PROJECT-postgres psql -U USER -d DATABASE -c '\dt'
```

## Frontend

```bash
curl -I http://localhost:FRONT_PORT
```

## Remoto

Validar do dispositivo que realmente consumirá o sistema.

Se frontend abre mas login retorna `Failed to fetch`, verificar:

1. URL efetiva da API no bundle;
2. `localhost` incorreto;
3. CORS;
4. mixed content;
5. firewall/Tailscale;
6. porta publicada;
7. API health.

---

# Diagnóstico rápido

## `ECONNREFUSED` no banco

Verificar se a aplicação usa `localhost`.

Container → Postgres deve normalmente usar:

```text
postgres:5432
```

## `relation "usuarios" does not exist`

A conexão funciona; schema não existe.

Resolver com migration, restore ou bootstrap explicitamente escolhido.

## `Failed to fetch` fora do servidor

Se o bundle contém:

```env
VITE_API_URL=http://localhost:3000
```

o browser procura a API na máquina cliente.

Usar endereço acessível ou `/api`.

## HTTPS + API HTTP

Possível mixed content.

Preferir mesma origem HTTPS com `/api`.

## Vite continua usando URL antiga

Rebuild:

```bash
docker build --no-cache -t PROJECT-front .
```

## Porta aparentemente ocupada por `ssh` no macOS/Colima

Não presumir processo indevido. Inspecionar antes; Colima pode usar encaminhamento SSH para portas publicadas pelos containers.

---

# Operações destrutivas

## Permitidas somente com intenção explícita

```bash
docker-compose down -v
docker volume rm ...
docker system prune --volumes
rm -rf ...
```

Antes de qualquer operação destrutiva:

- identificar projeto;
- identificar volume;
- explicar impacto;
- confirmar que existe backup quando houver dados importantes.

Parar containers sem apagar volumes é diferente de apagar persistência.

---

# Perfis de implantação

## Perfil 1 — Desenvolvimento interno

```text
Docker/Compose
portas individuais
Tailscale HTTP opcional
volumes
```

## Perfil 2 — Homologação privada

```text
Docker/Compose
Tailscale
HTTPS privado
migrations
backup
frontend/API mesma origem quando possível
```

## Perfil 3 — Produção pública

```text
Docker/Compose
reverse proxy central
domínio
HTTPS
migrations
backup automatizado
monitoramento
health checks
secrets
rollback
exposição mínima
```

---

# Checklist final

Antes de declarar sucesso:

```text
[ ] Stack real identificada
[ ] Dependências reais identificadas
[ ] Variáveis reais identificadas
[ ] Portas livres verificadas
[ ] Containers com nomes exclusivos
[ ] Volumes exclusivos
[ ] Secrets fora do Git
[ ] Banco/cache não expostos desnecessariamente
[ ] Estratégia de migrations definida
[ ] API health OK
[ ] Frontend OK
[ ] Frontend chama a API correta
[ ] Acesso remoto escolhido funciona
[ ] HTTPS validado quando aplicável
[ ] Restart policy configurada
[ ] Dados sobrevivem a restart
[ ] Serviços existentes permaneceram intactos
[ ] Comandos de operação/rollback documentados
```

---

# Formato da resposta da skill

Ao implementar, responder por etapas e apresentar decisões relevantes como opções.

Exemplo:

```text
Diagnóstico
- stack:
- serviços:
- portas ocupadas:
- riscos:

Escolha de exposição
A. Local
B. Tailscale HTTP
C. Tailscale HTTPS
D. Domínio público

Plano selecionado
- ...

Arquivos a criar/alterar
- ...

Comandos
- ...

Validação
- ...

Rollback
- ...
```

Não despejar dezenas de comandos antes de confirmar decisões que alterem exposição pública, banco existente ou dados persistentes.

---

# Regra de ouro

Antes de qualquer alteração, responder internamente a estas perguntas:

1. O que já está rodando neste host?
2. Como este projeto realmente lê suas configurações?
3. Quais dados precisam sobreviver?
4. Quem precisa acessar o projeto e de onde?
5. Qual endereço é válido em cada contexto: container, host e cliente remoto?
6. Como reverter esta alteração sem perder dados?

Se alguma resposta essencial estiver desconhecida, investigar antes de executar.
