# Projeto Gerador de XLSX e PDF

## Tecnologias Utilizadas

- **RabbitMQ**: mensageria para orquestrar o processamento assíncrono.
- **AWS S3**: armazenamento seguro dos relatórios gerados.
- **AWS API Gateway**: servir a aplicação com um gestor de gateway.
- **AWS Lambda**: servir a aplicação BFF de forma elástica (para manter o serviço sempre disponível e escalável).
- **PostgreSQL**: persistência dos metadados da solicitação e status de geração.
- **Docker**: containers para manter a execução padronizada, controlada e estável do Worker.
- **Kubernetes**: orquestrar os containers docker e fazer autoscailing Horizontal.
- **Cloudwatch**: logs em nuvem com dashboard de processamento e erros.
---

### BFF (Backend For Frontend)

- Expõe os endpoints REST:

  - `POST /generate`
    - Recebe os dados do relatório.
    - Insere os dados do relatório no banco (gerando assim o requestId).
    - Retorna:
        - HTTP status: 201.
        - Body: requestId, status (pending) e data de criação.

  - `GET /status/{requestId}`
    - Consulta o Redis (depois no PostgreSQL caso não encontre) e retorna:
      - Dados do relatório, status queued, processing, done ou error, URL do relatório como null, data de criação, data de atualização, enum de tipo de erro e mensagem de erro (caso tenha sido error, senão null).
      - Caso done, retorna dados do relatório, status (done), URL do relatório, data de criação e data de atualização.

---

### ⚙️ Worker

- Escutam mensagens da fila do RabbitMQ.
- Para cada mensagem:
  1. Verifica no PostgreSQL se o status ainda é queued ou error. Se não for, ignora para manter a idempotência.
  2. Atualiza status no banco para processing.
  3. Gera o relatório em formato XLSX ou PDF, conforme for especificado.
  4. Faz upload do arquivo para o bucket S3.
  5. Atualiza o status para done e salva a URL no banco.
  6. Em caso de erro, atualiza o status para error.
  7. Atualiza o registro usando o requestId no Redis.

---

## Idempotência
- O banco de dados é consultado antes de qualquer operação de geração.
- O worker valida o status antes de iniciar o processamento.
- Com isso, mesmo em casos de retry, duplicações são evitadas.

---

## Tolerância a Falhas e Retries

- RabbitMQ configurado com retry exponencial.
- Mensagens que falham várias vezes são enviadas para uma Dead Letter Queue (DLQ).
- Logs e alertas monitoram falhas recorrentes.
- Workers são idempotentes e seguros para reexecuções.

---

## Escalabilidade

- AWS Lambda lida de forma elástica (sempre disponível) e eficiente das requisições ao BFF.
- Horizontal scaling com múltiplos pods/workers processando filas simultaneamente (caso necessário).
- RabbitMQ distribui as mensagens entre workers ativos.
- AWS S3 lida com armazenamento de forma escalável e eficiente.
- PostgreSQL com índices garante performance nas consultas por requestId indexado.

---

## Estrutura do Banco de Dados

Tabela: `report_requests`

| Campo         | Tipo        | Descrição                              |
|---------------|-------------|----------------------------------------|
| requestId     | UUID        | Identificador único da requisição      |
| status        | ENUM        | queued, processing, done, error        |
| error_enum    | ENUM        | <tipos comuns de erros>                |
| error_message | TEXT        | Mensagens dos tipos comuns de erros    |
| created_at    | TIMESTAMP   | Data da requisição                     |
| updated_at    | TIMESTAMP   | Atualização mais recente               |

---

## Benefícios da Arquitetura
- Alta disponibilidade e resiliência.
- Processamento paralelo e escalável.
- Geração segura e controlada de arquivos.
- Evita duplicidade com controle total via banco de dados.
