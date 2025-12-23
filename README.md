# V360 — Proposta e Arquitetura

Resumo curto  
Aplicação web interativa que recebe perguntas do cliente, enfileira processamento assíncrono (Sidekiq + Redis), consulta base fiscal (PostgreSQL) e serviços de IA (OpenAI), e retorna resultado em tempo real via WebSocket (ActionCable).

## Visão geral dos componentes

![Diagrama de Arquitetura](/docs/diagrama-de-arquitetura.png)

- Client (React / HTML5 / JS)
    - Envia perguntas via HTTPS e escuta respostas via WSS (ActionCable).
- Rails API
    - Endpoints REST que recebem pedidos e enfileiram jobs.
    - Canal WebSocket (ActionCable) para notificações em tempo real.
- Redis
    - Fila de tarefas (Sidekiq), cache e mecanismo Pub/Sub para notificar a API.
- Sidekiq Workers
    - Processamento assíncrono: chama OpenAI, consulta PostgreSQL, persiste resultados.
- PostgreSQL
    - Persistência da base fiscal V360 e resultados.
- Serviços externos
    - OpenAI (LLM) — interpretação e geração de texto.
    - SEFAZ / Gov — validação/consulta de notas fiscais.

## Fluxo de dados (mapeado aos diagramas)

![Diagrama de Sequência](/docs/diagrama-de-sequencia.png)

1. Front‑end envia pergunta (HTTP POST) ao Rails API.  
2. Rails API enfileira um job no Redis (Sidekiq) e responde 202 Accepted ao cliente.  
3. Sidekiq (worker) consome o job do Redis.  
4. Worker consulta a API de IA (OpenAI) para interpretar/gerar resposta.  
5. Worker consulta PostgreSQL para dados fiscais necessários.  
6. Ao concluir, o Worker publica o resultado em Redis (Pub/Sub).  
7. Rails (assinando o Pub/Sub) recebe notificação e envia resposta final ao cliente via WebSocket (ActionCable).

## Endpoints e comportamento esperado
- POST /api/v1/questions
    - Recebe payload da pergunta, valida, cria registro/trace e enfileira job → retorna 202.
- ActionCable /cable
    - Canal para envio de resposta final por usuário/room.
- Jobs Sidekiq
    - Job: QuestionProcessingJob
        - Input: question_id, context
        - Responsabilidades: chamar OpenAI, consultar DB, persistir resultado, publicar evento.

## Protocolos e integrações

![Diagrama de Blocos](/docs/diagrama-de-blocos.png)

- Comunicação cliente ↔ API: HTTPS / WSS
- API ↔ Redis: enfileirar/dequeue (Sidekiq), Pub/Sub para notificações
- Worker ↔ OpenAI: HTTPS REST (rate limiting + retries exponenciais)
- Worker ↔ SEFAZ (Gov): chamadas HTTP seguras para validações fiscais
- Persistência: PostgreSQL via pool de conexões

## Segurança
- Sempre HTTPS, WSS com certificados (ACM/ALB).
- Autenticação/Autorização: JWT ou Devise + Pundit (RBAC).
- Segredos em AWS Secrets Manager / Parameter Store.
- Sanitização e validação de inputs antes de enviar para LLM e SEFAZ.
- Rate limiting e circuit breaker para chamadas à OpenAI e SEFAZ.

## Resiliência e escalabilidade
- Sidekiq workers horizontalmente escaláveis; jobs idempotentes.
- Redis como fila e Pub/Sub com monitoramento (ElastiCache).
- PostgreSQL em RDS Multi-AZ, backups e read replicas para leitura intensiva.
- Auto Scaling para aplicações (ECS/Fargate/EC2) e workers.
- Estratégia de retry com backoff e dead letter queue para falhas persistentes.

## Observabilidade & Operação
- Logs estruturados (JSON) com correlação (request_id / job_id).
- Métricas: Sidekiq queue size, Redis hits/misses, DB latency, LLM latency/usage.
- Tracing distribuído (OWASP/Zipkin/Datadog) entre API → Worker → OpenAI.
- Dashboards e alertas para filas longas, erros 5xx e latência de LLM.

## Deploy / Infra (sugestão AWS)
- Rails API: ECS/Fargate ou EC2 + ALB, AutoScaling, SSL (ACM).
- Sidekiq: serviço/container separado com escala independente.
- PostgreSQL: RDS (Multi-AZ) + read replica.
- Redis: ElastiCache (clustered) ou managed Redis.
- Secrets: Secrets Manager; Configs: SSM Parameter Store.
- CI/CD: pipelines (GitHub Actions / CodePipeline) com migrações controladas.

## Boas práticas operacionais
- Tornar jobs idempotentes e com limites de tempo.
- Cache de respostas LLM quando aplicável.
- Limitar e agrupar chamadas para OpenAI (batching quando possível).
- Validar e normalizar dados fiscais antes de persistir.
- Testes end‑to‑end para fluxo assíncrono (enfileirar → processar → notificar).

## Variantes e extensões
- Adição de cache semântico para respostas LLM.
- Pipeline de pré‑processamento para anonimizar dados sensíveis antes de enviar ao LLM.
- Mecanismo de fallback/compactação de respostas quando OpenAI indisponível.

## Variáveis de ambiente essenciais
- DATABASE_URL
- REDIS_URL
- RAILS_ENV, RAILS_MASTER_KEY, SECRET_KEY_BASE
- OPENAI_API_KEY
- SEFAZ_API_KEY / SEFAZ_BASE_URL
- AWS credentials / REGION

Observação final  
A documentação mapeia os diagramas fornecidos: arquitetura de camadas (Client → Rails → Redis/Sidekiq → Postgres/OpenAI/SEFAZ) e o fluxo assíncrono com pub/sub + WebSocket para retorno em tempo real. Ajustes operacionais e de segurança devem ser adaptados à política de dados e requisitos de compliance da organização.