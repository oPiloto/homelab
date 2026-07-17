# Homelab

Minha stack self-hosted.

| Stack        | Serviço principal                        | Domínio               |
|--------------|------------------------------------------|-----------------------|
| `traefik/`   | Reverse proxy + CrowdSec + socket-proxy  | `traefik.example.com` |
| `authelia/`  | SSO / MFA (forward-auth)                 | `auth.example.com`    |
| `nextcloud/` | Nextcloud (app + cron + MariaDB + Redis) | `cloud.example.com`   |
| `vault/`     | Vaultwarden (Bitwarden self-hosted)      | `vault.example.com`   |

Cada pasta é um stack Compose independente (`compose.yaml` + `.env` próprios),
mas todos se conectam à mesma rede Docker externa `traefik` para que o
reverse proxy alcance os containers das aplicações.

## Instalação (primeira vez)

A ordem abaixo **não é opcional**. Ela existe porque `traefik/compose.yaml`
declara os volumes `vault_logs` e `nextcloud_logs` como `external: true`
(consumidos pelo CrowdSec para analisar os logs dessas aplicações) — se
esses volumes ainda não existirem, `docker compose up` do stack do Traefik
falha.

```bash
# 1) Criar a rede compartilhada
docker network create traefik

# 2) Preparar variáveis de ambiente (repita para cada stack)
cp vault/.env.example vault/.env
cp nextcloud/.env.example nextcloud/.env
cp authelia/.env.example authelia/.env
cp traefik/.env.example traefik/.env
cp authelia/config/users_database.yml.example authelia/config/users_database.yml
# edite cada .env e preencha os segredos (senhas, tokens, chaves — use
# `openssl rand -hex 32` para gerar valores fortes)

# 3) Pré-criar o acme.json com as permissões corretas
touch traefik/certs/acme.json
chmod 600 traefik/certs/acme.json

# 4) Subir vault e nextcloud primeiro (criam os volumes externos de log)
# ou: `docker volume create vault_logs && docker volume create nextcloud_logs`
# ou: remover configuração de traefik/crowdsec/config/acquis.yaml
cd vault && docker compose up -d && cd ..
cd nextcloud && docker compose up -d && cd ..

# 5) Subir authelia
cd authelia && docker compose up -d && cd ..

# 6) Gerar a chave do bouncer do CrowdSec e configurar o Traefik
cd traefik
docker compose up -d socket-proxy crowdsec   # sobe só as dependências do CrowdSec primeiro
docker exec crowdsec cscli bouncers add traefik-bouncer
# copie a chave impressa para:
#   - traefik/.env            -> CROWDSEC_BOUNCER_KEY=<chave>
#   - traefik/secrets/crowdsec_lapi_key  (crie a partir do .example, cole só a chave)

# 7) Subir o Traefik completo
docker compose up -d
```

Depois disso, todo `docker compose up -d` dentro de cada pasta é idempotente
e pode ser repetido normalmente (a ordem só importa por causa dos volumes
externos na primeira criação).

## Atualização

```bash
cd <stack>
docker compose pull
docker compose up -d
docker image prune -f # remove imagens antigas não usadas
```

Caso as imagens estejam fixadas em versões específicas, 
`docker compose pull` não vai trazer nada novo até você atualizar manualmente a tag no
`compose.yaml`.

## Backup

Todos os dados que importam estão em **named volumes** (não em bind mounts
soltos), então sobrevivem a reboot, atualização de imagem e restart do
Docker. Faça backup dos volumes, não dos containers:

```bash
# Exemplo genérico para qualquer volume nomeado
docker run --rm \
  -v <nome_do_volume>:/data:ro \
  -v "$(pwd)/backups":/backup \
  alpine tar czf /backup/<nome_do_volume>-$(date +%F).tar.gz -C /data .
```

Volumes que precisam de backup regular:

| Volume                                 | Conteúdo                              |
|----------------------------------------|---------------------------------------|
| `vault_data`                           | Cofre do Vaultwarden (o mais crítico) |
| `nextcloud_data`                       | Arquivos do Nextcloud                 |
| `nextcloud_db`                         | Banco MariaDB do Nextcloud            |
| `authelia_logs`, `authelia_redis`      | Sessões (baixa prioridade, recriável) |
| `traefik/certs/acme.json` (bind mount) | Certificados TLS                      |

Para o MariaDB, prefira um dump lógico consistente em vez de copiar os
arquivos brutos do volume a quente:

```bash
docker exec nextcloud-db sh -c 'exec mariadb-dump -u root -p"$MYSQL_ROOT_PASSWORD" --all-databases' \
  | gzip > backups/nextcloud-db-$(date +%F).sql.gz
```

Coloque o Nextcloud em modo manutenção antes de um backup consistente do
volume de dados:

```bash
docker exec -u www-data nextcloud-app php occ maintenance:mode --on
# ... backup dos volumes nextcloud_data e nextcloud_db ...
docker exec -u www-data nextcloud-app php occ maintenance:mode --off
```

## Restore

```bash
docker compose down
docker volume rm <nome_do_volume>
docker volume create <nome_do_volume>
docker run --rm \
  -v <nome_do_volume>:/data \
  -v "$(pwd)/backups":/backup \
  alpine sh -c "cd /data && tar xzf /backup/<arquivo>.tar.gz"
docker compose up -d
```

Para o MariaDB, restaure o dump lógico:

```bash
cat backups/nextcloud-db-<data>.sql.gz | gunzip | \
  docker exec -i nextcloud-db sh -c 'exec mariadb -u root -p"$MYSQL_ROOT_PASSWORD"'
```
