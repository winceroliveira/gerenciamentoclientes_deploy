# gerenciamentoclientes_deploy

Repositório da **stack Docker** para Portainer (compose + nginx gateway).

**Leia:** [PORTAINER.md](./PORTAINER.md) — configuração completa e automação com GitHub Actions.

## Repositórios do projeto

| Repo | Função |
|------|--------|
| [gerenciamentoclientes_back](https://github.com/winceroliveira/gerenciamentoclientes_back) | API — publica imagem GHCR |
| [gerenciamentoclientes_front](https://github.com/winceroliveira/gerenciamentoclientes_front) | Frontend — publica imagem GHCR |
| **gerenciamentoclientes_deploy** (este) | Compose usado pela **única** stack no Portainer |
