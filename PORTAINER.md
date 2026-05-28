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

Se os pacotes forem **privados** (padrão em repo privado):

1. GitHub → **Settings** → **Packages** → cada pacote → **Package settings** → **Change visibility** (public), **ou**
2. Portainer → **Registries** → **Add registry** → GitHub (`ghcr.io`) com PAT (`read:packages`).

---

## Passo 3 — Create stack no Portainer (tela Repository)

| Campo | Valor |
|-------|--------|
| **Name** | `progplay-gerenciamento` |
| **Build method** | **Repository** |
| **Repository URL** | `https://github.com/winceroliveira/gerenciamentoclientes_deploy` |
| **Repository reference** | `refs/heads/main` |
| **Compose path** | `docker-compose.yml` |
| **Authentication** | Ligado se o repo deploy for privado (PAT ou usuário/senha) |
| **GitOps updates** | Opcional — útil se você alterar só o `docker-compose`; para código use o **webhook** |

### Environment variables (na mesma tela)

Adicione manualmente (não commite segredos):

| Nome | Exemplo |
|------|---------|
| `JWT_SECRET` | string longa aleatória |
| `PUBLIC_URL` | `https://gerenciamento.seudominio.com.br` |
| `CORS_ORIGIN` | igual ao `PUBLIC_URL` |
| `HTTP_PORT` | `8080` |

Clique em **Deploy the stack**.

---

## Passo 4 — Webhook para automação (recomendado)

1. Portainer → **Stacks** → `progplay-gerenciamento`
2. Abra **Webhook** (ou **Service** → webhook da stack)
3. Copie a URL (algo como `https://seu-portainer:9443/api/stacks/webhooks/xxxx`)

No GitHub, em **ambos** os repos `gerenciamentoclientes_back` e `gerenciamentoclientes_front`:

1. **Settings** → **Secrets and variables** → **Actions**
2. New secret: `PORTAINER_STACK_WEBHOOK` = URL copiada

A cada push em `main`, o Action:

1. Roda testes + build da imagem
2. Publica no GHCR
3. Chama o webhook → Portainer **redeploy** e `pull` das imagens novas

---

## Passo 5 — Secrets nos repositórios back/front

Nenhum secret obrigatório para publicar no GHCR (usa `GITHUB_TOKEN`).

Opcional:

| Secret | Uso |
|--------|-----|
| `PORTAINER_STACK_WEBHOOK` | Redeploy automático após push |

---

## GitOps updates (toggle da sua tela)

- **Ligado:** Portainer verifica o repo **deploy** periodicamente e reaplica se `docker-compose.yml` mudar.
- **Não substitui** o webhook para atualizar imagens `back`/`front` — para isso use o webhook ou **Pull and redeploy** manual.

Combinação ideal:

- **Webhook** → código novo (back/front)
- **GitOps** → mudanças de infra (porta, nginx, variáveis no compose)

---

## Checklist rápido

- [ ] Repo `gerenciamentoclientes_deploy` criado e com compose
- [ ] Actions de `back` e `front` verdes (imagens no GHCR)
- [ ] Registry configurado no Portainer (se imagens privadas)
- [ ] **1 stack** criada via Repository
- [ ] `JWT_SECRET` e URLs no Portainer
- [ ] Secret `PORTAINER_STACK_WEBHOOK` nos dois repos de código
- [ ] Teste: push em `main` → Action → stack redeploy → site responde
