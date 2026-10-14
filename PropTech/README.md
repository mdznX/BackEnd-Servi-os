# PropTech - Microsserviço de Serviços

Microsserviço responsável por gerenciar áreas comuns, reservas, ocorrências, manutenções e avisos do sistema de condomínio PropTech.

## Stack

- **NestJS** (TypeScript) - Framework backend
- **Prisma ORM** - ORM para MySQL
- **MySQL** - Banco de dados (local no Windows via MySQL Installer, XAMPP ou MySQL Workbench)
- **JWT + Passport** - Autenticação (validação de token emitido pelo MS Moradores)
- **Swagger** - Documentação da API
- **class-validator / class-transformer** - Validação de DTOs

## Pré-requisitos

- Node.js 18+
- MySQL 8.0+ (instalado localmente no Windows — MySQL Installer, XAMPP ou MySQL Workbench)
- npm

## Configuração (PowerShell)

> **Importante:** Todos os comandos abaixo são para **PowerShell** no Windows.

1. Clone o repositório e instale as dependências:

```powershell
npm install
```

2. Configure as variáveis de ambiente copiando o arquivo de exemplo:

```powershell
Copy-Item .env.example .env
```

3. Edite o `.env` com suas configurações (use seu editor preferido, ex: `code .env` ou `notepad .env`):

```env
DATABASE_URL="mysql://root:SUASENHA@localhost:3306/condominio_servicos"
JWT_SECRET="sua-chave-secreta-super-segura"
PORT=3002
NODE_ENV=development
```

> **Nota:** Substitua `SUASENHA` pela senha do seu usuário `root` do MySQL local.

## Banco de Dados

### Opção 1: Usando migrações Prisma (recomendado para desenvolvimento)

```powershell
npx prisma generate
npx prisma migrate dev
npx prisma db seed
```

### Opção 2: Se já tem o banco criado via SQL (`Servico_otimizado.sql`)

O arquivo `Servico_otimizado.sql` contém **triggers e stored procedures** que usam a sintaxe `DELIMITER` do MySQL.

> **⚠️ IMPORTANTE:** Scripts com `DELIMITER` **devem ser executados no MySQL Workbench** (ou outro client GUI que suporte `DELIMITER`), **não no terminal** (`mysql -u root -p < arquivo.sql` falhará).

As regras de negócio (validação de sobreposição de reservas, datas de manutenção, soft delete, etc.) **já estão implementadas no código NestJS**. Portanto, **triggers e procedures do SQL são opcionais** — o Prisma + código da aplicação já garantem a integridade.

Se quiser usar apenas o schema do SQL (tabelas, índices, FKs) sem triggers/procedures:
1. Rode o script no MySQL Workbench (apenas a parte de `CREATE TABLE` / `CREATE INDEX` / `ALTER TABLE`).
2. Depois sincronize o Prisma:

```powershell
npx prisma generate
npx prisma migrate dev
npx prisma db seed
```

## Executando o Projeto

### Desenvolvimento

```powershell
npm run start:dev
```

O servidor iniciará em `http://localhost:3002`

### Produção

```powershell
npm run build
npm run start:prod
```

## Documentação da API (Swagger)

Acesse `http://localhost:3002/docs` para visualizar e testar a API.

## Autenticação

Este microsserviço **não faz login**. Ele apenas **valida tokens JWT** emitidos pelo MS Moradores.

### Payload esperado no token:

```json
{
  "sub": "user-uuid",
  "morador_id": 1,
  "perfil": "MORADOR" | "SINDICO"
}
```

### Headers necessários:

```
Authorization: Bearer <seu-jwt-token>
```

### Perfis:

- **MORADOR**: Acesso às próprias reservas/ocorrências, visualização de áreas ativas e avisos visíveis
- **SINDICO**: Acesso total (CRUD completo em todos os recursos + dashboard)

### Token de desenvolvimento

Para testar sem o MS Moradores, use o endpoint (apenas em `NODE_ENV !== production`):

```
GET /dev/token?perfil=SINDICO&morador_id=1
```

Retorna um token JWT válido para testes.

## Endpoints Principais

### Áreas Comuns (`/area_comum`)
| Método | Rota | Perfil | Descrição |
|--------|------|--------|-----------|
| GET | `/` | MORADOR, SINDICO | Lista com filtros `?status=ativas|inativas&busca=nome&page=1&limit=10` |
| GET | `/:id` | MORADOR, SINDICO | Detalhes da área |
| POST | `/` | SINDICO | Criar área |
| PATCH | `/:id` | SINDICO | Atualizar área |
| DELETE | `/:id` | SINDICO | Soft delete (status = 0) |

### Reservas (`/reserva`)
| Método | Rota | Perfil | Descrição |
|--------|------|--------|-----------|
| GET | `/` | MORADOR, SINDICO | Lista (MORADOR vê só suas, SINDICO vê todas) - filtro `?tipo=PENDENTE|CONFIRMADA|CANCELADA` |
| GET | `/:id` | MORADOR, SINDICO | Detalhes da reserva |
| POST | `/` | MORADOR, SINDICO | Criar reserva (morador_id vem do token) |
| PATCH | `/:id/confirmar` | SINDICO | Confirmar reserva (só se PENDENTE) |
| PATCH | `/:id/cancelar` | MORADOR, SINDICO | Cancelar (SINDICO qualquer, MORADOR só suas; só PENDENTE/CONFIRMADA) |

**Regras de negócio:**
- Horário fim > horário início
- Não permite sobreposição na mesma área/data com reservas PENDENTE/CONFIRMADA ativas
- Área deve existir e estar ativa

### Ocorrências (`/ocorrencia`)
| Método | Rota | Perfil | Descrição |
|--------|------|--------|-----------|
| GET | `/` | MORADOR, SINDICO | Lista com filtro `?tipo=ABERTA|EM_ANDAMENTO|FINALIZADA` (inclui manutenções) |
| GET | `/:id` | MORADOR, SINDICO | Detalhes com manutenções |
| POST | `/` | MORADOR, SINDICO | Criar (tipo inicia ABERTA, morador_id do token) |
| PATCH | `/:id` | SINDICO | Atualizar status/tipo |
| DELETE | `/:id` | SINDICO | Soft delete |

### Manutenções (`/manutencao`) - Apenas SINDICO
| Método | Rota | Descrição |
|--------|------|-----------|
| GET | `/` | Lista com filtro `?tipo=PENDENTE|EM_EXECUCAO|CONCLUIDA` (inclui título da ocorrência) |
| GET | `/:id` | Detalhes |
| POST | `/` | Criar |
| PATCH | `/:id` | Atualizar |
| DELETE | `/:id` | Soft delete |

**Regra:** `data_conclusao` não pode ser anterior a `data_solicitacao`

### Avisos (`/avisos`)
| Método | Rota | Perfil | Descrição |
|--------|------|--------|-----------|
| GET | `/` | MORADOR, SINDICO | MORADOR: só ativos, não expirados, visíveis. SINDICO: todos com filtro `?filtro=ativos|expirados` |
| GET | `/:id` | MORADOR, SINDICO | Detalhes |
| POST | `/` | SINDICO | Criar |
| PATCH | `/:id` | SINDICO | Atualizar/expirar |
| DELETE | `/:id` | SINDICO | Soft delete |

### Dashboard (`/dashboard`) - Apenas SINDICO
| Método | Rota | Descrição |
|--------|------|-----------|
| GET | `/` | Retorna: `total_areas_ativas`, `total_areas_inativas`, `total_reservas_mes`, `total_ocorrencias_abertas`, `total_manutencoes_pendentes`, `total_avisos_ativos` |

## Paginação

Todas as listagens suportam paginação:
```
?page=1&limit=10
```

Resposta padrão:
```json
{
  "data": [...],
  "total": 100,
  "page": 1,
  "limit": 10
}
```

## Soft Delete

Todas as operações de DELETE realizam **soft delete** (atualizam `*_status = 0`). Registros com status 0 não aparecem nas listagens padrão.

## Tratamento de Erros

- **400 Bad Request**: Dados inválidos, regras de negócio violadas
- **401 Unauthorized**: Token ausente ou inválido
- **403 Forbidden**: Perfil insuficiente para a operação
- **404 Not Found**: Recurso não encontrado
- **409 Conflict**: Conflito (ex: nome de área duplicado, sobreposição de reserva)

## Variáveis de Ambiente

| Variável | Descrição | Obrigatório |
|----------|-----------|-------------|
| `DATABASE_URL` | URL de conexão MySQL | Sim |
| `JWT_SECRET` | Chave secreta para validar JWT | Sim |
| `PORT` | Porta do servidor (padrão: 3002) | Não |
| `NODE_ENV` | Ambiente (development/production) | Não |

## Estrutura do Projeto

```
src/
├── main.ts                 # Bootstrap da aplicação
├── app.module.ts           # Módulo raiz
├── prisma/                 # Prisma Service & Module
├── common/
│   └── dto/pagination.dto.ts
└── modules/
    ├── auth/               # JWT Strategy, Guards, Decorators
    ├── area_comum/
    ├── reserva/
    ├── ocorrencia/
    ├── manutencao/
    ├── avisos/
    └── dashboard/
```

## Scripts Úteis

```powershell
# Gerar Prisma Client
npm run prisma:generate

# Criar migração
npm run prisma:migrate

# Popular banco com dados de exemplo
npm run prisma:seed

# Reset completo do banco (apaga tudo e recria via migrations + seed)
npm run db:reset

# Abrir Prisma Studio
npm run prisma:studio

# Build para produção
npm run build
```

## Licença

MIT