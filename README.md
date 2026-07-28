# petnet-logs

Serviço de logging da aplicação PetNet. API REST construída com PHP 8.2 + Slim Framework, banco de dados MySQL e containerização via Docker.

---

## Stack

- **PHP 8.2** com Apache
- **Slim Framework 4** — roteamento e middlewares
- **PDO / MySQL 8.4** — persistência
- **firebase/php-jwt** — validação de JWT
- **respect/validation** — validação de payload
- **Docker + Docker Compose** — ambiente local e produção
- **Traefik** — reverse proxy com TLS automático em produção

---

## Estrutura

```
petnet-logs/
├── public/
│   └── index.php               # Entrypoint da aplicação
├── src/
│   ├── config/
│   │   └── env.php             # Carregamento das variáveis de ambiente
│   ├── db/
│   │   └── Connection.php      # Singleton PDO — gerencia conexão com o banco
│   ├── controllers/
│   │   └── LogController.php
│   ├── services/
│   │   └── LogService.php
│   ├── repositories/
│   │   └── LogRepository.php
│   ├── middleware/
│   │   ├── AuthMiddleware.php  # Token estático (POST /logs)
│   │   └── JwtMiddleware.php   # JWT de sessão (GET /logs)
│   └── routes/
│       └── logs.php
├── database/
│   └── migration.sql
├── Dockerfile
├── docker-compose.yml          # Ambiente local
├── docker-compose.prod.yml     # Produção (Traefik)
├── deploy.sh                   # Script de deploy na VPS
└── .env.example
```

---

## Endpoints

| Método | Rota        | Middleware      | Descrição                    |
|--------|-------------|-----------------|------------------------------|
| GET    | `/logs`     | `JwtMiddleware` | Lista todos os logs          |
| GET    | `/logs/{id}`| `JwtMiddleware` | Retorna um log pelo ID       |
| POST   | `/logs`     | `AuthMiddleware`| Cria um novo log             |

### Autenticação

**GET** — requer cookie `token` contendo um JWT válido com `type === "MANAGER"` (mesmo segredo usado pelo backend Node.js).

**POST** — requer header `Authorization` com o valor exato de `LOG_API_TOKEN` (token estático compartilhado com o backend).

### Payload — POST `/logs`

```json
{
  "entity":      "Pet",
  "action":      "DELETE",
  "status":      "success",
  "responsible": "12345678901",
  "details":     "Pet id=42 removido pelo manager.",
  "created_at":  "2025-01-15 10:30:00"
}
```

| Campo         | Tipo     | Obrigatório | Regras                        |
|---------------|----------|-------------|-------------------------------|
| `entity`      | string   | sim         | 1–50 caracteres               |
| `action`      | string   | sim         | 1–50 caracteres               |
| `status`      | string   | sim         | 1–20 caracteres               |
| `created_at`  | datetime | sim         | formato `Y-m-d H:i:s`        |
| `responsible` | string   | não         | até 11 caracteres (CPF)       |
| `details`     | string   | não         | texto livre                   |

---

## Ambiente local

### Pré-requisitos

- Docker e Docker Compose instalados

### Setup

```bash
cp .env.example .env
# Preencha as variáveis no .env
```

```bash
docker compose up -d
```

A API ficará disponível em `http://localhost:8080`.

---

## Variáveis de ambiente

| Variável          | Descrição                                                   |
|-------------------|-------------------------------------------------------------|
| `DB_HOST`         | Host do banco (usar `db` dentro do Docker)                  |
| `DB_NAME`         | Nome do banco de dados                                      |
| `DB_USER`         | Usuário do banco                                            |
| `DB_PASSWORD`     | Senha do usuário                                            |
| `DB_ROOT_PASSWORD`| Senha do root MySQL                                         |
| `LOG_API_TOKEN`   | Token estático para autenticar o POST `/logs`               |
| `JWT_SECRET`      | Segredo JWT — deve ser o mesmo do backend Node.js           |
| `FRONTEND_URLS`   | Origens permitidas pelo CORS, separadas por vírgula         |
| `LOGS_API_HOST`   | Domínio exposto pelo Traefik em produção                    |
| `APP_PORT`        | Porta do host para o Traefik acessar (padrão: `8081`)       |

---

## Banco de dados

A migration é executada automaticamente pelo MySQL na primeira inicialização do container (`docker-entrypoint-initdb.d`).

```sql
CREATE TABLE IF NOT EXISTS logs (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    entity      VARCHAR(50)  NOT NULL,
    action      VARCHAR(50)  NOT NULL,
    status      VARCHAR(20)  NOT NULL,
    responsible VARCHAR(11)  NULL,
    details     TEXT         NULL,
    created_at  DATETIME     NOT NULL
) DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```