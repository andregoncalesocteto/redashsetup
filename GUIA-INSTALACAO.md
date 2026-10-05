# Guia de instalação e operação do Redash

Este documento descreve o que o script `setup.sh` faz, como acessar o Redash após a instalação e quais comandos usar para subir e baixar o ambiente.

---

## Visão geral

Este repositório é um **setup de referência para instalar o [Redash](https://redash.io/) com Docker** em um servidor Linux único. O `setup.sh` automatiza a instalação de dependências, configuração e (por padrão) a subida dos containers.

**Requisitos:**

- Executar como **root**: `./setup.sh`
- Distros suportadas: Alma/Rocky/CentOS/RHEL 8–9, Debian 12, Fedora 38–40, Oracle Linux 9, Ubuntu 20.04/22.04

**Arquivos principais:**

| Arquivo | Função |
|---------|--------|
| `setup.sh` | Script de instalação |
| `compose.yaml` | Definição dos serviços Docker |
| `/opt/redash/env` | Variáveis de ambiente (segredos, URLs) |
| `/opt/redash/redash_make_default.sh` | Script opcional para definir o Redash como projeto Compose padrão |

---

## O que o `setup.sh` faz

### Fluxo de execução

1. **Verificações iniciais**
   - Exige execução como root
   - Detecta se Docker e Docker Compose já estão instalados
   - Se Docker existir, pula a instalação; caso contrário, identifica a distro via `/etc/os-release`

2. **Parâmetros opcionais**

   | Parâmetro | Efeito |
   |-----------|--------|
   | `--dont-start` / `-d` | Instala tudo, mas **não inicia** os containers |
   | `--overwrite` / `-o` | **Apaga** configuração e banco existentes e reinstala do zero |
   | `--preview` / `-p` | Usa a imagem `preview` do Docker Hub |
   | `--version X.Y.Z` | Instala uma versão específica (ex.: `25.8.0`) |
   | `--help` / `-h` | Mostra ajuda |

   > **Nota:** `--preview` e `--version` não podem ser usados juntos.
   > Sem nenhum dos dois, o script busca a **última release estável** via API do GitHub.

3. **Instalação do Docker** (por distro)
   - Debian → `install_docker_debian()`
   - Ubuntu → `install_docker_ubuntu()`
   - Fedora → `install_docker_fedora()`
   - RHEL e compatíveis (AlmaLinux, CentOS, Oracle Linux, Rocky) → `install_docker_rhel()`

4. **Estrutura de diretórios** (`create_directories`)
   - Cria `/opt/redash` e `/opt/redash/postgres-data`
   - Com `--overwrite`, move o banco antigo para `postgres-data-<timestamp>`

5. **Arquivo de ambiente** (`create_env`)
   - Gera segredos com `pwgen`: `REDASH_COOKIE_SECRET`, `REDASH_SECRET_KEY`, `POSTGRES_PASSWORD`
   - Cria `/opt/redash/env` com URLs do Redis, PostgreSQL, CSRF, timeout do Gunicorn, etc.
   - Se `/opt/redash/env` já existir **sem** `--overwrite`, reutiliza e só preenche valores obrigatórios em falta

6. **Docker Compose** (`setup_compose`)
   - Baixa `compose.yaml` do repositório oficial no GitHub
   - Substitui `__TAG__` pela versão escolhida
   - Define `COMPOSE_FILE=/opt/redash/compose.yaml` e `COMPOSE_PROJECT_NAME=redash`

7. **Script auxiliar** (`create_make_default`)
   - Baixa `redash_make_default.sh` para tornar o Redash o projeto Compose padrão no login (opcional)

8. **Inicialização** (`startup`)
   - Se **não** usar `--dont-start`:
     1. `docker compose run --rm server create_db` — inicializa o banco
     2. `docker compose up -d` — sobe todos os serviços
     3. Informa a URL de acesso

### Serviços criados

O `compose.yaml` define os seguintes containers:

| Serviço | Função |
|---------|--------|
| `server` | Aplicação web Redash (porta 5000) |
| `nginx` | Proxy reverso (porta 80) |
| `postgres` | Banco de dados |
| `redis` | Cache/filas |
| `scheduler` | Agendador de queries |
| `worker` / `scheduled_worker` / `adhoc_worker` | Workers de processamento |

---

## URL de acesso

Após a instalação, o script informa:

```
http://<hostname-do-servidor>:5000
```

Substitua `<hostname-do-servidor>` pelo FQDN ou IP da máquina.

**Exemplos:**

- `http://meu-servidor.local:5000`
- `http://192.168.1.100:5000`

### Acesso alternativo via nginx

O `compose.yaml` também expõe o nginx na porta 80:

| URL | Serviço |
|-----|---------|
| `http://<host>:5000` | Redash direto (padrão informado pelo script) |
| `http://<host>` | Via nginx (proxy reverso, porta 80) |

> **Dica:** Na primeira visita, a interface pode demorar para carregar (compilação Python em background). Nas visitas seguintes, deve abrir bem mais rápido.

---

## Comandos para subir e baixar o ambiente

Tudo fica em **`/opt/redash/compose.yaml`**, projeto Compose **`redash`**.

Use `docker compose` (plugin moderno) ou `docker-compose` (binário legado), conforme o que estiver instalado.

### Forma explícita (sempre funciona)

**Subir o ambiente:**

```bash
docker compose -f /opt/redash/compose.yaml up -d
```

**Parar e remover os containers** (dados do Postgres permanecem em `/opt/redash/postgres-data`):

```bash
docker compose -f /opt/redash/compose.yaml down
```

**Só parar, sem remover containers:**

```bash
docker compose -f /opt/redash/compose.yaml stop
```

**Voltar a subir depois de um `stop`:**

```bash
docker compose -f /opt/redash/compose.yaml start
```

**Reiniciar tudo:**

```bash
docker compose -f /opt/redash/compose.yaml restart
```

### Forma curta (após configurar o projeto padrão)

Execute uma vez (opcional):

```bash
/opt/redash/redash_make_default.sh
```

Isso adiciona ao `~/.profile` ou `~/.bashrc`:

```bash
export COMPOSE_PROJECT_NAME=redash
export COMPOSE_FILE=/opt/redash/compose.yaml
```

Depois de relogar (ou `source ~/.profile` / `source ~/.bashrc`):

```bash
docker compose up -d      # subir
docker compose down       # baixar
docker compose stop       # parar
docker compose start      # subir de novo
docker compose ps         # ver status
docker compose logs -f    # ver logs
```

### Comandos úteis

| Comando | O que faz |
|---------|-----------|
| `docker compose -f /opt/redash/compose.yaml ps` | Lista containers e status |
| `docker compose -f /opt/redash/compose.yaml logs -f server` | Logs do serviço web |
| `docker compose -f /opt/redash/compose.yaml pull` | Atualiza imagens antes de subir |

### Primeira subida com `--dont-start`

Se a instalação foi feita com `--dont-start`, na **primeira** subida execute:

```bash
docker compose -f /opt/redash/compose.yaml run --rm server create_db
docker compose -f /opt/redash/compose.yaml up -d
```

### `down` vs `stop`

| Comando | Efeito |
|---------|--------|
| `stop` | Para os containers; eles permanecem criados |
| `down` | Para e remove containers da rede Compose; **volume do Postgres permanece** |

---

## Remoção completa

Para desinstalar o Redash por completo (apaga volumes e imagens):

```bash
docker compose -f /opt/redash/compose.yaml down --volumes --rmi all
```

Remova também, se existirem, estas linhas de `~/.profile` e `~/.bashrc`:

```bash
export COMPOSE_PROJECT_NAME=redash
export COMPOSE_FILE=/opt/redash/compose.yaml
```

Apague a pasta de instalação:

```bash
sudo rm -fr /opt/redash
```

---

## Referências

- [Upgrade Guide](https://redash.io/help/open-source/admin-guide/how-to-upgrade)
- [Imagens Docker no Docker Hub](https://hub.docker.com/r/redash/redash/tags)
- README principal do repositório: `README.md`
