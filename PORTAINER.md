# Portainer — uma stack, automação via GitHub Actions

## Resposta direta

| Pergunta | Resposta |
|----------|----------|
| Uma stack por repositório? | **Não.** Use **1 stack** (`progplay-gerenciamento`) apontando para o repo **`gerenciamentoclientes_deploy`**. |
| O que os outros repos fazem? | `back` e `front` publicam **imagens Docker** no GHCR via Actions. |
| Como atualiza na VM? | Action chama o **webhook** da stack no Portainer (pull das imagens `:latest`). |

```
┌─────────────────┐     push main      ┌──────────────────┐
│ back / front    │ ─────────────────► │ GitHub Actions   │
│ (código)        │                    │ build + GHCR     │
└─────────────────┘                    └────────┬─────────┘
                                                │ POST webhook
                                                ▼
┌─────────────────┐     compose + env  ┌──────────────────┐
│ gerenciamentoclientes_deploy          │ Portainer        │
│ (docker-compose) │ ◄── GitOps opcional │ 1 stack          │
└─────────────────┘                    └────────┬─────────┘
                                                ▼
                                         api + web + gateway
```

---

## Passo 1 — Criar repositório deploy no GitHub

Crie um repo vazio: **https://github.com/winceroliveira/gerenciamentoclientes_deploy**

Envie o conteúdo desta pasta (`docker-compose.yml`, `nginx-gateway.conf`, etc.).

---

## Passo 2 — GitHub Container Registry (GHCR)

Após o primeiro push em `back` e `front`, as imagens aparecem em:

- `ghcr.io/winceroliveira/gerenciamentoclientes_back:latest`
- `ghcr.io/winceroliveira/gerenciamentoclientes_front:latest`

Se os pacotes forem **privados** (padrão em repo privado) — erro comum: `ghcr.io ... unauthorized`:

**Opção A — Pacotes públicos (mais simples)**

1. https://github.com/winceroliveira?tab=packages  
2. Abra `gerenciamentoclientes_back` → **Package settings** → **Change visibility** → **Public**  
3. Repita para `gerenciamentoclientes_front`  
4. No Portainer: **Stacks** → `progplay` → **Pull and redeploy** (ou deploy de novo)

**Opção B — Registry no Portainer (mantém privado)**

O PAT na stack (**Repository → Authentication**) serve só para clonar o `docker-compose.yml`.  
O pull das imagens usa **outro** login: **Registries**.

1. GitHub → **Settings** → **Developer settings** → **Personal access tokens**  
2. **Não use só fine-grained “repository-scoped”** — precisa permissão **Packages: Read** no token, ou use token **classic** com:
   - `read:packages` (obrigatório)
   - `repo` (se os repos forem privados)
3. Portainer → menu **Registries** (não dentro da stack) → **Add registry**

| Campo | Valor |
|-------|--------|
| Registry provider | Custom |
| Name | `GHCR` |
| Registry URL | `ghcr.io` (sem `https://`) |
| Username | `winceroliveira` |
| Password | o PAT `ghp_...` inteiro |

4. **Test connection** (se o botão existir) ou teste na VM (abaixo)  
5. **Environments** → seu ambiente local → **Registries** → confirme que `GHCR` está disponível  
6. **Stacks** → `progplay` → **Editor** → **Update the stack** (ou delete e crie de novo)

Se criou a stack **antes** do registry, o deploy antigo não “herda” credencial — precisa **Update** ou recriar.

**Teste na VM (SSH):**

```bash
echo "COLE_SEU_PAT_AQUI" | docker login ghcr.io -u winceroliveira --password-stdin
docker pull ghcr.io/winceroliveira/gerenciamentoclientes_back:latest
docker pull ghcr.io/winceroliveira/gerenciamentoclientes_front:latest
```

Se `pull` funcionar no SSH mas falhar no Portainer, o registry não está ligado ao ambiente — revise passo 5.

### Erro `denied` (autenticou mas sem permissão)

| Causa | Correção |
|-------|----------|
| Token fine-grained **sem** Packages Read | Editar token → Permissions → **Packages: Read** nos dois pacotes, ou criar token **classic** com `read:packages` |
| Registry `GIT` com senha antiga | **Registries** → editar `GIT` → colar PAT novo → salvar → **Update stack** |
| `nginx-gateway.conf` not a directory | Compose usa `configs:` embutido; remova stack antiga se falhou antes (pasta fantasma em `/data/compose/`) |
| Token “Never used” | Credencial do registry nunca funcionou — refaça `docker login` no SSH |

> Autenticação do **repositório Git** (compose) é separada da autenticação do **GHCR** (imagens Docker).

---

## Passo 3 — Create stack no Portainer (tela Repository)

| Campo | Valor |
|-------|--------|
| **Name** | `progplay-gerenciamento` |
| **Build method** | **Repository** |
| **Repository URL** | `https://github.com/winceroliveira/gerenciamentoclientes_deploy` |
| **Repository reference** | `refs/heads/main` |
| **Compose path** | `docker-compose.yml` |
| **Additional paths** | não necessário (nginx embutido em `configs:`) |

### Portas e banco — o que é normal

| O que você vê | Normal? |
|---------------|---------|
| `api` e `web` **sem** porta publicada no host | **Sim** — ficam só na rede Docker interna |
| `gateway` **com** `8080:80` | **Obrigatório** — se não aparecer, a stack está errada |
| **Nenhum** container `postgres` | **Sim** — usamos **SQLite** no volume `progplay-sqlite-data` |
| Volume `progplay-sqlite-data` em **Volumes** | **Sim** — é o “banco” (arquivo `app.db`) |

Se `gateway` não mostra `8080`, remova a stack e recrie com o compose atual (sem `container_name`).

### NPM na mesma VM

Use **IP + porta** (rede externa não é obrigatória no compose):

| Campo | Valor |
|-------|--------|
| Forward Hostname / IP | `168.231.97.170` |
| Forward Port | `8080` |

Opcional (rede Docker interna): no SSH rode `docker network ls`, ache a rede do stack n8n/NPM e adicione no compose uma rede `external` com o **nome exato** — ex.: pode ser `n8nwpp_default`, `n8n_default`, etc.
| **Authentication** | Ligado se o repo deploy for privado (PAT ou usuário/senha) |
| **GitOps updates** | Opcional — útil se você alterar só o `docker-compose`; para código use o **webhook** |

### Environment variables (na mesma tela)

Adicione manualmente (não commite segredos):

| Nome | Exemplo |
|------|---------|
| `JWT_SECRET` | string longa aleatória |
| `PUBLIC_URL` | `https://www.progplay.com.br` |
| `CORS_ORIGIN` | igual ao `PUBLIC_URL` |
| `HTTP_PORT` | `8080` |

Clique em **Deploy the stack**.

---

## Passo 4 — Redeploy automático (Community Edition)

> **Webhook de stack** só existe no **Portainer Business**. No CE, use a **API** com access token.

### 4.1 — IDs da stack

Abra a stack **progplay** no Portainer e copie da URL do navegador:

```
https://168.231.97.170:9443/#!/3/docker/stacks/progplay?id=25&...
                              ↑ endpoint ID              ↑ stack ID
```

| Campo | Exemplo (sua VM) |
|-------|------------------|
| `PORTAINER_ENDPOINT_ID` | `3` |
| `PORTAINER_STACK_ID` | `25` |

### 4.2 — Access token

1. Portainer → **My account** → **Access tokens** → **Add access token**
2. Copie o token (`ptr_...`)

### 4.3 — Secrets no GitHub

Em **ambos** os repos `gerenciamentoclientes_back` e `gerenciamentoclientes_front`:

**Settings** → **Secrets and variables** → **Actions**

| Secret | Valor |
|--------|--------|
| `PORTAINER_URL` | `https://168.231.97.170:9443` |
| `PORTAINER_API_TOKEN` | token `ptr_...` |
| `PORTAINER_STACK_ID` | `25` |
| `PORTAINER_ENDPOINT_ID` | `3` |

A cada push em `main`, o Action:

1. Roda testes + build da imagem
2. Publica no GHCR
3. Chama a API → Portainer **redeploy** com **pull** das imagens `:latest`

Teste manual (SSH):

```bash
curl -sk -X PUT \
  "https://168.231.97.170:9443/api/stacks/25/git/redeploy?endpointId=3" \
  -H "X-API-Key: SEU_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"Prune": false, "RepullImageAndRedeploy": true}'
```

---

## Passo 5 — Secrets nos repositórios back/front

Nenhum secret obrigatório para publicar no GHCR (usa `GITHUB_TOKEN`).

Opcional (redeploy automático no CE):

| Secret | Uso |
|--------|-----|
| `PORTAINER_URL` | URL do Portainer |
| `PORTAINER_API_TOKEN` | Access token |
| `PORTAINER_STACK_ID` | ID numérico da stack |
| `PORTAINER_ENDPOINT_ID` | ID do environment (ex.: `3`) |

---

## GitOps updates (toggle da sua tela)

- **Ligado:** Portainer verifica o repo **deploy** periodicamente e reaplica se `docker-compose.yml` mudar.
- **Não substitui** a API para atualizar imagens `back`/`front` — para isso use os secrets acima ou **Pull and redeploy** manual.

Combinação ideal:

- **API Portainer** → código novo (back/front)
- **GitOps** → mudanças de infra (porta, nginx, variáveis no compose)

---

## Checklist rápido

- [ ] Repo `gerenciamentoclientes_deploy` criado e com compose
- [ ] Actions de `back` e `front` verdes (imagens no GHCR)
- [ ] Registry configurado no Portainer (se imagens privadas)
- [ ] **1 stack** criada via Repository
- [ ] `JWT_SECRET` e URLs no Portainer
- [ ] Secrets Portainer API nos dois repos de código (`URL`, `TOKEN`, `STACK_ID`, `ENDPOINT_ID`)
- [ ] Teste: push em `main` → Action → stack redeploy → site responde

---

## Domínio progplay.com.br (site na raiz www)

O frontend já separa rotas:

| URL | Página |
|-----|--------|
| `https://www.progplay.com.br/` | Site público (orçamentos) |
| `https://www.progplay.com.br/admin` | Painel administrativo |

### DNS na Hostinger (progplay.com.br)

| Tipo | Nome | Aponta para | Observação |
|------|------|------------|------------|
| **A** | `@` | `168.231.97.170` | Raiz `progplay.com.br` → VPS |
| **CNAME** | `www` | `progplay.com.br` | Já existe — passa a resolver para a VPS junto com o `@` |

Não remova registros de **e-mail** (TXT/CNAME `hostingermail`, `autodiscover`, etc.).

Aguarde propagação e teste: `nslookup www.progplay.com.br` → `168.231.97.170`.

> Se antes existia site da Hostinger na raiz, ele deixa de ser servido de lá e passa a ser este sistema na VPS.

### Nginx Proxy Manager

**Proxy Hosts → Add Proxy Host**

**Details**

| Campo | Valor |
|-------|--------|
| Domain Names | `www.progplay.com.br`, `progplay.com.br` |
| Scheme | `http` |
| Forward Hostname / IP | `168.231.97.170` |
| Forward Port | `8080` |
| Websockets Support | ligado |

**SSL**

- Request a new SSL Certificate (Let's Encrypt)
- Force SSL + HTTP/2

Assim `https://progplay.com.br` e `https://www.progplay.com.br` funcionam com o mesmo certificado.

**Redirect opcional (só www):** em **Advanced** do NPM, custom location ou segundo proxy só para redirecionar `progplay.com.br` → `www` — só necessário se quiser URL canônica única.

### Portainer (stack progplay-gerenciamento)

```
PUBLIC_URL=https://www.progplay.com.br
CORS_ORIGIN=https://www.progplay.com.br
HTTP_PORT=8080
JWT_SECRET=<sua-chave>
```

**Update the stack** após alterar.

### Testes finais

- https://www.progplay.com.br
- https://www.progplay.com.br/admin
- https://www.progplay.com.br/health
