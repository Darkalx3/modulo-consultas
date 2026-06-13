# Módulo Consultas (G4)

O módulo de consultas G4 faz parte do projeto integrador de 2026, realizado em conjunto com as matérias de Engenharia de Software, Banco de Dados e Programação I.

Ele é responsável pela coordenação entre **paciente**, **horário** e **estado do atendimento**, sendo a fonte confiável de verdade sobre as consultas para todo o ecossistema. O módulo garante que só seja possível agendar consultas para pacientes que existem (validados junto ao G1) e para horários efetivamente livres (validados junto ao G3), além de controlar o ciclo de vida de cada consulta (`agendada`, `realizada` ou `cancelada`) segundo regras explícitas.

O G4 consome os módulos **G1 (Pacientes)**, **G2 (Profissionais)** e **G3 (Agenda)** e fornece dados para os módulos **G5 (Prontuário)**, **G7 (Exames)**, **G10 (Faturamento)**, **G11 (Triagem)** e **G13 (Autorização)** via API REST.

**Repositório:** `git@github.com:Darkalx3/modulo-consultas.git`

---

## Tecnologias

Este é um projeto **somente backend** (sem frontend). A stack utilizada é:

- **Node.js** — runtime principal do backend, responsável por expor a API REST do módulo.
- **PostgreSQL** — banco de dados relacional onde as consultas são persistidas. Seguindo o RNF07 (minimização de dados / LGPD), o banco armazena apenas referências lógicas (`pacienteId`, `horarioId`) e **não** duplica dados pessoais.
- **Docker / Docker Compose** — orquestração do ambiente, subindo a aplicação Node.js e o banco PostgreSQL em containers, de forma que o projeto rode em qualquer máquina sem configuração manual de dependências.

### Estrutura em camadas

O backend segue uma arquitetura em camadas com portas e adaptadores (decisão de arquitetura DA01):

- **Controller (API REST):** traduz requisições HTTP em chamadas de aplicação.
- **Service (aplicação):** orquestra as regras de negócio e a integração com os gateways.
- **Domínio (`Consulta` / `StatusConsulta`):** concentra a máquina de estados do ciclo de vida da consulta.
- **Gateways (G1, G2, G3) e Repository:** isolam a integração externa e a persistência.

### Convenção de nomenclatura

Todo o código e a API seguem o padrão **PascalCase** como convenção de nomenclatura. Isso vale para nomes de classes, métodos, rotas e campos expostos pela API, mantendo a consistência em todo o módulo (por exemplo: `ConsultaController`, `BuscarPorId`, `StatusConsulta`).

---

## Como rodar (ambiente Docker)

Pré-requisitos: **Docker** e **Docker Compose** instalados.

1. Clone o repositório:

   ```bash
   git clone git@github.com:Darkalx3/modulo-consultas.git
   cd modulo-consultas
   ```

2. Crie um arquivo `.env` na raiz (use `.env.example` como base) com as variáveis de ambiente, por exemplo:

   ```env
   PORT=3000
   DATABASE_URL=postgres://postgres:postgres@db:5432/consultas
   POSTGRES_USER=postgres
   POSTGRES_PASSWORD=postgres
   POSTGRES_DB=consultas
   ```

3. Suba os containers:

   ```bash
   docker compose up --build
   ```

4. A API ficará disponível em `http://localhost:3000`. O endpoint de verificação de saúde pode ser acessado em `GET /health`.

Para parar e remover os containers:

```bash
docker compose down
```

---

## Documentação da API

Todos os recursos são versionados por prefixo de rota (`/v1`). A API trafega JSON (`application/json`) e utiliza códigos de status HTTP semanticamente corretos. Exceto o *health check*, todos os endpoints exigem token de autenticação válido (sem token → `401`; perfil sem permissão → `403`).

### Endpoints do módulo G4

| Método | Rota | Descrição |
|--------|------|-----------|
| `POST` | `/v1/consultas` | Agenda uma nova consulta (valida paciente em G1, valida e reserva horário em G3). |
| `GET` | `/v1/consultas/{id}` | Recupera os dados completos de uma consulta pelo identificador. |
| `GET` | `/v1/consultas?{filtros}` | Lista consultas com filtros, ordenadas por data. |
| `PATCH` | `/v1/consultas/{id}/status` | Altera o estado da consulta (`realizada` ou `cancelada`). |
| `GET` | `/v1/horarios?{filtros}` | Busca horários disponíveis (consome G3). |
| `GET` | `/v1/profissionais?{filtro}` | Consulta profissionais por especialidade (consome G2). |
| `GET` | `/health` | Verificação de saúde do serviço (não exige autenticação). |

**Filtros disponíveis em `GET /v1/consultas`:** `pacienteId`, `status` (`agendada` / `realizada` / `cancelada`), `especialidade`, `dataInicio` e `dataFim`.

### Exemplos

**Agendar consulta**

```http
POST /v1/consultas
Content-Type: application/json
Authorization: Bearer <token>

{
  "pacienteId": "uuid-do-paciente",
  "horarioId": "uuid-do-horario"
}
```

Resposta `201 Created`:

```json
{
  "id": "uuid-da-consulta",
  "pacienteId": "uuid-do-paciente",
  "horarioId": "uuid-do-horario",
  "profissionalId": "uuid-do-profissional",
  "dataHora": "2026-07-01T14:00:00Z",
  "status": "AGENDADA"
}
```

**Concluir consulta**

```http
PATCH /v1/consultas/{id}/status
Content-Type: application/json

{ "status": "realizada" }
```

**Cancelar consulta** (libera o horário em G3)

```http
PATCH /v1/consultas/{id}/status
Content-Type: application/json

{ "status": "cancelada" }
```

### Códigos de status utilizados

| Código | Significado no módulo |
|--------|-----------------------|
| `200` | Operação de leitura ou alteração concluída com sucesso. |
| `201` | Consulta criada com sucesso. |
| `401` | Token de autenticação ausente, inválido ou expirado. |
| `403` | Perfil sem permissão para a operação. |
| `404` | Consulta não encontrada. |
| `409` | Conflito (horário ocupado, concorrência ou transição de estado inválida). |
| `422` | Dados inválidos (campo obrigatório ausente ou paciente inexistente). |
| `503` | Dependência indisponível (G1 ou G3 fora do ar). |

---

## Organização de commits e branches

### Commits

O projeto adota um padrão simples de mensagens de commit, com um prefixo indicando o tipo da alteração:

- **`add`** — adição de algo novo (arquivo, funcionalidade, endpoint).
- **`update`** — alteração ou melhoria de algo já existente.
- **`fix`** — correção de bug ou de comportamento incorreto.

Cada commit deve descrever de forma curta e objetiva o que foi alterado, no tempo presente. Exemplos:

```
add: endpoint de agendamento de consulta
update: validação de paciente no gateway do G1
fix: liberação de horário ao cancelar consulta
```

### Branches

O repositório trabalha com **duas branches principais**:

- **`developer`** — branch de desenvolvimento. É onde todo o trabalho do dia a dia acontece e onde as novas alterações são integradas primeiro. Os commits são feitos aqui (ou em branches de feature que partem da `developer` e voltam para ela).
- **`main`** — branch estável, que representa o estado pronto/entregável do projeto. Não recebe commits diretos.

### Processo de lançamento (PR para a `main`)

Todo lançamento para a `main` deve seguir o processo padrão de **Pull Request (PR)**:

1. As alterações são desenvolvidas e validadas na branch `developer`.
2. Quando o conjunto de mudanças está pronto para ser lançado, abre-se um **PR de `developer` → `main`**.
3. O PR é revisado por outro integrante do grupo antes de ser aprovado.
4. Após a aprovação, o PR é mesclado (merge) na `main`.

Dessa forma, a `main` se mantém sempre estável e nenhuma alteração chega a ela sem passar por revisão.

---

## Sobre a implementação (Programação I)

Nosso grupo **não cursa a matéria de Programação I** dentro do projeto integrador. Por esse motivo, toda a parte que envolve programação (código do backend, integração com o banco e configuração do ambiente) foi desenvolvida com o **Claude Code**, apenas para simular um cenário real de implementação a partir da modelagem e dos requisitos definidos nas demais entregas.

---

## Contato dos integrantes

- Alexsandro Krzjzaniack — alexsandrokrzj@gmail.com
- Gabriel Zini Turri — gabriel.turrri@gmail.com
- Luis Guilherme Ferronato Khon — lkohn1002@gmail.com
