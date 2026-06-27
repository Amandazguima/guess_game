Aqui está um exemplo de um arquivo `README.md` para o seu jogo:

---

# Jogo de Adivinhação com Flask

Este é um simples jogo de adivinhação desenvolvido utilizando o framework Flask. O jogador deve adivinhar uma senha criada aleatoriamente, e o sistema fornecerá feedback sobre o número de letras corretas e suas respectivas posições.

## Funcionalidades

- Criação de um novo jogo com uma senha fornecida pelo usuário.
- Adivinhe a senha e receba feedback se as letras estão corretas e/ou em posições corretas.
- As senhas são armazenadas  utilizando base64.
- As adivinhações incorretas retornam uma mensagem com dicas.
  
## Requisitos

- Python 3.8+ - 3.12
- Flask
- Um banco de dados local (ou um mecanismo de armazenamento configurado em `current_app.db`)
- node 18.17.0

## Instalação

1. Clone o repositório:

   ```bash
   git clone https://github.com/fams/guess_game.git
   cd guess-game
   ```

2. Crie um ambiente virtual e ative-o:

   ```bash
   python3 -m venv venv
   source venv/bin/activate  # Linux/Mac
   venv\Scripts\activate  # Windows
   ```

3. Instale as dependências:

```bash
docker compose down            # para containers, mantém volumes
docker compose down -v         # também remove o volume postgres_data
```

---

## URLs de acesso

Após `docker compose up -d --build`, a aplicação está disponível em:

| O quê                        | URL                                  |
|------------------------------|--------------------------------------|
| **Frontend (interface web)** | **`http://localhost:8080/`**         |
| API — health check direto    | `http://localhost:5050/health`       |
| API — health check via NGINX | `http://localhost:8080/health`       |
| API — criar jogo             | `POST http://localhost:8080/api/create` |
| API — submeter palpite       | `POST http://localhost:8080/api/guess/<game_id>` |
| Postgres (acesso direto, ex.: psql) | `localhost:5433` (user=`postgres`, pass=`***`, db=`postgres`) |

### Exemplo rápido com `curl`

```bash
# 1. Criar um jogo (senha "banana")
curl -X POST http://localhost:8080/api/create \
     -H "Content-Type: application/json" \
     -d '{"password":"banana"}'
# Resposta: {"game_id":"ABC12345"}

# 2. Submeter palpite correto
curl -X POST http://localhost:8080/api/guess/ABC12345 \
     -H "Content-Type: application/json" \
     -d '{"guess":"banana"}'
# Resposta: {"result":"Correct"}
```

> **Observação sobre o NGINX:** O bloco `location /api/` do `nginx.conf` faz `proxy_pass http://guessgame_backend/` (com barra final). Isso **remove** o prefixo `/api` antes de enviar ao backend. Por isso o backend expõe rotas em `/create` e `/guess/<id>`, não em `/api/create`. Esse comportamento é intencional e documentado no `nginx.conf`.

---

## Atualização de componentes

O design com três serviços isolados permite atualizar cada parte **de forma independente**, sem rebuildar todo o ambiente:

### Atualizar somente o backend

```bash
docker compose up -d --build backend
```

Útil, por exemplo, quando você altera o código Python mas o frontend e o banco continuam iguais.

### Atualizar somente o frontend

```bash
docker compose up -d --build frontend
```

Útil quando há mudanças no React/NGINX. O `Dockerfile.frontend` é multi-stage (Node 18-alpine para build, NGINX alpine para servir), então apenas a etapa de build é refeita se o código React muda.

### Atualizar o banco de dados (ex.: mudar versão do Postgres)

Edite `docker-compose.yml` (por exemplo, `postgres:15-alpine` → `postgres:16-alpine`) e:

```bash
docker compose up -d --build postgres
```

> **Atenção:** mudar a versão do Postgres **não** migra os dados automaticamente. Faça backup antes:
> ```bash
> docker compose exec postgres pg_dump -U postgres postgres > backup.sql
> ```

### Atualizar a configuração do NGINX

Edite `nginx.conf` e:

```bash
docker compose up -d --build frontend
```

### Rebuild completo (forçar tudo do zero)

```bash
docker compose down -v
docker compose up -d --build
```

---

## Resiliência

A configuração foi desenhada para sobreviver a falhas comuns:

| Cenário                                | Comportamento                                                                |
|----------------------------------------|------------------------------------------------------------------------------|
| Container do backend cai               | Reinicia automaticamente (`restart: unless-stopped`)                         |
| Container do frontend cai              | Reinicia automaticamente                                                     |
| Container do postgres cai               | Reinicia automaticamente; **dados preservados** pelo volume `postgres_data`   |
| Postgres reinicia                      | Backend precisa ser reiniciado também (psycopg2 não reconecta sozinho) — o healthcheck do compose sinaliza isso; basta `docker compose restart backend` |
| Imagem base desatualizada              | `docker compose pull && docker compose up -d --build`                        |

### Healthchecks implementados

- **postgres**: `pg_isready -U postgres -d postgres` a cada 10s
- **backend**: `GET http://localhost:5000/health` a cada 15s
- **frontend**: depende do backend estar healthy (`depends_on: condition: service_healthy`)

Graças a esses healthchecks, a ordem de inicialização é **sempre**: postgres → backend → frontend. Você não verá o backend tentar conectar ao banco antes dele estar pronto.

---

## Decisões de design

### Por que Postgres e não SQLite?

A aplicação suporta três backends de persistência (SQLite, Postgres, DynamoDB), selecionados via `FLASK_DB_TYPE`. Para um ambiente containerizado de produção-like, **Postgres** é o padrão:

- **Volume nomeado (`postgres_data`)**: garante que os dados sobrevivam a `docker compose down` (somente `down -v` apaga).
- **Imagem oficial `postgres:15-alpine`**: leve (~80 MB), com suporte a `healthcheck` nativo via `pg_isready`.

### Por que NGINX como proxy reverso?

- Permite servir o build do React **e** fazer proxy da API no mesmo host/porta.
- O bloco `upstream guessgame_backend` no `nginx.conf` está pronto para **balanceamento de carga** entre múltiplos backends. Para escalar:
  1. Edite `docker-compose.yml` adicionando réplicas: `backend: { ..., deploy: { replicas: 3 } }` (em formato swarm) **ou** duplique os serviços manualmente.
  2. Edite `nginx.conf` adicionando mais `server backend_N:5000;` ao `upstream`.
  3. `docker compose up -d --build`.

### Por que dois Dockerfiles separados?

- `Dockerfile.backend` e `Dockerfile.frontend` são **independentes** — uma mudança no backend não invalida o cache de build do frontend (e vice-versa), acelerando o ciclo de desenvolvimento.
- O `Dockerfile.frontend` é **multi-stage**: usa `node:18-alpine` para o build do React e descarta essa camada no resultado final, que é só `nginx:alpine + arquivos estáticos`. Isso mantém a imagem de produção pequena.

### Por que CMD `flask run` no backend?

O `run.py` chama `app.run(debug=True)`, que por padrão escuta em `127.0.0.1:5000` — inacessível de outros containers. Para preservar o código original sem alterá-lo, o `Dockerfile.backend` usa `CMD ["flask", "run", "--host=0.0.0.0", "--port=5000"]`, que respeita as variáveis `FLASK_RUN_HOST` e `FLASK_RUN_PORT` já definidas no Dockerfile. O resultado é o backend escutando em `0.0.0.0:5000`, acessível dentro da rede do compose.

### Por que variáveis de ambiente em vez de volumes de configuração?

Todas as configurações (credenciais do banco, tipo de persistência, host) são injetadas via `environment:` no `docker-compose.yml`. Isso:

- Facilita mover o ambiente entre máquinas (basta copiar o `docker-compose.yml`).
- Permite override por container sem editar imagens: `FLASK_DB_TYPE=sqlite docker compose up -d backend`.

---

## Solução de problemas

### Containers sobem mas `/health` retorna 502 via NGINX

Geralmente é timing — o NGINX subiu antes do backend estar pronto. Verifique:

```bash
docker compose ps                 # todos devem estar "Up"
docker compose logs backend       # procure "Running on http://0.0.0.0:5000"
docker compose restart backend    # força reconexão
```

### Porta 80, 5000 ou 5432 já em uso no host (macOS)

O `docker-compose.yml` já publica em portas alternativas (8080, 5050, 5433) para evitar conflitos com AirPlay (5000) e outros serviços do sistema. Se ainda assim houver conflito, edite o bloco `ports:` do serviço afetado.

### Backend perde conexão após restart do Postgres

O driver `psycopg2` não reconecta automaticamente após o servidor cair. Solução:

```bash
docker compose restart backend
```

### Como ver logs em tempo real

```bash
docker compose logs -f                # todos os serviços
docker compose logs -f backend        # só o backend
docker compose logs --tail=100 backend # últimas 100 linhas
```

### Como executar comandos dentro de um container

```bash
docker compose exec backend bash
docker compose exec postgres psql -U postgres -d postgres
```

---

## Uso sem Docker (legado)

As instruções originais do projeto continuam funcionando — use-as se preferir rodar Python/React diretamente na máquina (sem containers).

### Backend

```bash
python3 -m venv venv
source venv/bin/activate            # Linux/Mac
# ou: venv\Scripts\activate        # Windows

pip install -r requirements.txt

# Configure o banco via start-backend.sh ou exportando variáveis:
export FLASK_APP="run.py"
export FLASK_DB_TYPE="sqlite"
export FLASK_DB_PATH="./guess_game.db"

./start-backend.sh &
```

⚠️ **Cuidado (Windows/WSL):** o arquivo `start-backend.sh` pode estar com fim de linha CRLF. Verifique com `vim -b start-backend.sh`. Se estiver CRLF, converta:

```bash
sed -i 's/\r$//' start-backend.sh
```

### Frontend

```bash
cd frontend
npm install
npm start
```

O frontend estará disponível em `http://localhost:3000` por padrão (configurável em `frontend/package.json`).

---

## Licença

Este projeto faz parte do repositório demo guess-game. Consulte o repositório original para informações de licença.