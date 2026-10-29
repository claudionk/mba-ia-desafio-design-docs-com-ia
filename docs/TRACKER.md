# Tracker de Rastreabilidade — Sistema de Webhooks de Notificação de Pedidos

Cada linha liga um item registrado nos documentos à sua origem: a transcrição da reunião (`TRANSCRICAO`, localizada por `[hh:mm] Nome`) ou o código do repositório (`CODIGO`, localizada pelo caminho do arquivo).

Itens marcados nos documentos como **"identificada na revisão do desenho"** ou **"a validar"** são lacunas encontradas na análise. Eles **não** são decisões da reunião: a localização aponta para a fala ou o arquivo que originou a dúvida.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Atlas, MaxDistribuição e Nova Cargo pediram notificação de mudança de status; hoje fazem polling em GET /orders | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Atlas pode migrar para concorrente se não houver entrega até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Restrição | "Tempo real" para os clientes = abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-04 | docs/PRD.md | Contexto | OMS não tem hoje mecanismo de notificação, eventos ou fila; mudança de status só em changeStatus | CODIGO | src/modules/orders/order.service.ts |
| PRD-PERS-01 | docs/PRD.md | Público-alvo | Usuários do OMS representam o cliente e cadastram webhooks pela API com JWT | TRANSCRICAO | [09:32] Marcos |
| PRD-PERS-02 | docs/PRD.md | Público-alvo | Roles existentes ADMIN e OPERATOR | CODIGO | src/middlewares/auth.middleware.ts |
| PRD-CEN-01 | docs/PRD.md | Cenário | Cliente filtra só SHIPPED e DELIVERED | TRANSCRICAO | [09:33] Marcos |
| PRD-CEN-02 | docs/PRD.md | Cenário | Cliente com manutenção planejada de 2 h | TRANSCRICAO | [09:16] Diego |
| PRD-CEN-03 | docs/PRD.md | Cenário | Cliente já vazou secret em log de aplicação | TRANSCRICAO | [09:22] Diego |
| PRD-OBJ-01 | docs/PRD.md | Métrica | Latência p95 < 10 s entre mudança de status e 1ª tentativa | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Métrica | 100% das mudanças de status assinadas com evento registrado (atomicidade) | TRANSCRICAO | [09:06] Diego |
| PRD-OBJ-03 | docs/PRD.md | Métrica | Janela de ≈ 15 h coberta por retentativas | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-04 | docs/PRD.md | Métrica | 3 de 3 clientes solicitantes em produção até o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-05 | docs/PRD.md | Métrica | Entrega em ≤ 3 sprints, incluindo revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook com URL, customerId (no body, não do JWT) e lista de status | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-01b | docs/PRD.md | Restrição | customerId passado no body ou no path; não vem do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Editar webhook (PATCH), incluindo estado ativo | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04b | docs/PRD.md | Requisito Funcional | Configuração guarda url + secret + customer_id + estado ativo | TRANSCRICAO | [09:21] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtro de eventos por lista de status por endpoint | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Toda mudança de status gera notificação para webhooks inscritos | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Conteúdo da notificação: event_id, tipo, timestamp, pedido, from/to, customer, total_cents, sem itens | TRANSCRICAO | [09:43] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Notificação assinada para o cliente validar origem e integridade | TRANSCRICAO | [09:19] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Rotação de secret pela API com grace period de 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Retry automático com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | DLQ com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ por ADMIN, com auditoria de quem executou | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas (sucesso/falha, payload, resposta, tempo) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | Recusar URL não-https com erro de validação | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Polling de 2 s; latência mínima de 2 s aceita | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Status e evento na mesma transação; sem status sem evento | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Cliente lento/offline não pode travar nem desfazer mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | At-least-once com dedup pelo event_id | TRANSCRICAO | [09:26] Larissa |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Ordem só por order_id e com single-worker | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s por chamada HTTP | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Limite de 64 KB, erro se ultrapassar (sem truncar) | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256, secret por endpoint, TLS obrigatório | TRANSCRICAO | [09:22] Sofia |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Sem infraestrutura nova: mesmo MySQL e mesma stack | TRANSCRICAO | [09:07] Diego |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | CRUD por qualquer role autenticada; replay só ADMIN | TRANSCRICAO | [09:37] Sofia |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo | E-mail ao cliente após falhas repetidas adiado para próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo | Dashboard visual descartado (projeto do time de frontend) | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Webhooks inbound descartados (só outbound) | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Arquivamento de entregues (~30 dias) fora do escopo | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-06 | docs/PRD.md | Fora de escopo | Ordenação global/múltiplos workers adiados | TRANSCRICAO | [09:13] Diego |
| PRD-OOS-07 | docs/PRD.md | Fora de escopo | Clientes nunca pediram ordenação global | TRANSCRICAO | [09:14] Marcos |
| PRD-OOS-08 | docs/PRD.md | Fora de escopo | Exactly-once descartado | TRANSCRICAO | [09:25] Diego |
| PRD-OOS-09 | docs/PRD.md | Fora de escopo | Itens do pedido fora do payload | TRANSCRICAO | [09:43] Diego |
| PRD-DEC-01 | docs/PRD.md | Trade-off | Tabela de decisões e trade-offs consolidada a partir dos ADRs | TRANSCRICAO | [09:48] Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança da Sofia (≥ 2 dias úteis) antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Documentação no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Documentar integração via API no portal | TRANSCRICAO | [09:40] Marcos |
| PRD-DEP-04 | docs/PRD.md | Dependência | Confirmação de prazo com a Atlas | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-05 | docs/PRD.md | Dependência | Sessão de revisão do design com Bruno e Diego antes de codar | TRANSCRICAO | [09:50] Larissa |
| PRD-DEP-06 | docs/PRD.md | Dependência | Endpoint existente PATCH /orders/:id/status | CODIGO | src/modules/orders/order.routes.ts |
| PRD-RISK-01 | docs/PRD.md | Risco | Atraso faz a Atlas migrar; prazo fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Cliente não deduplica; responsabilidade jogada ao cliente | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-03 | docs/PRD.md | Risco | Vazamento de secret | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Rajada de chamadas sem rate limiting | TRANSCRICAO | [09:38] Diego |
| PRD-RISK-05 | docs/PRD.md | Risco | Indisponibilidade > 15 h vai para DLQ sem aviso automático | TRANSCRICAO | [09:17] Marcos |
| PRD-RISK-06 | docs/PRD.md | Risco | Processo de envio parado acumula eventos (processo separado) | TRANSCRICAO | [09:11] Diego |
| PRD-CA-01 | docs/PRD.md | Critério de aceitação | Revisão de segurança aprovada antes do deploy | TRANSCRICAO | [09:49] Sofia |
| PRD-TEST-01 | docs/PRD.md | Estratégia de teste | Testes de integração no padrão Vitest + supertest existente | CODIGO | tests/orders.test.ts |
| PRD-TEST-02 | docs/PRD.md | Estratégia de teste | Integração no order.service e testes ponta a ponta | TRANSCRICAO | [09:46] Larissa |
| PRD-CRON-01 | docs/PRD.md | Cronograma | Estimativa de 3 sprints com revisão da Sofia no fim | TRANSCRICAO | [09:47] Larissa |
| RFC-META-01 | docs/RFC.md | Metadado | Larissa abre o doc de design e marca revisão com Bruno e Diego | TRANSCRICAO | [09:50] Larissa |
| RFC-TLDR-01 | docs/RFC.md | Decisão | Resumo da proposta consolidado na fala de fechamento | TRANSCRICAO | [09:48] Larissa |
| RFC-CTX-01 | docs/RFC.md | Contexto | Transação de changeStatus atualiza orders, insere history e movimenta estoque | CODIGO | src/modules/orders/order.service.ts |
| RFC-PROP-01 | docs/RFC.md | Decisão | Webhook outbound apenas | TRANSCRICAO | [09:03] Sofia |
| RFC-PROP-02 | docs/RFC.md | Decisão | Nenhuma infraestrutura nova; outbox no MySQL existente resolve | TRANSCRICAO | [09:07] Diego |
| RFC-PROP-03 | docs/RFC.md | Decisão | Headers X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo síncrono no service de orders | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-01b | docs/RFC.md | Alternativa descartada | "Síncrono está fora de questão" | TRANSCRICAO | [09:06] Diego |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams exigiria subir mais infra | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger MySQL para notificar worker (sem LISTEN/NOTIFY) | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | 3 tentativas cobrem pouco | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-04b | docs/RFC.md | Alternativa descartada | Retry indefinido deixa evento pendurado | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Exactly-once exige coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| RFC-Q-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Diego |
| RFC-Q-02 | docs/RFC.md | Questão em aberto | E-mail em falhas repetidas: próxima fase | TRANSCRICAO | [09:37] Marcos |
| RFC-Q-03 | docs/RFC.md | Questão em aberto | Escalar para múltiplos workers (particionar por order_id ou lock) | TRANSCRICAO | [09:13] Diego |
| RFC-Q-04 | docs/RFC.md | Questão em aberto | Endurecer permissões do CRUD no futuro | TRANSCRICAO | [09:37] Sofia |
| RFC-Q-05 | docs/RFC.md | Questão em aberto | Arquivamento de linhas entregues após ~30 dias | TRANSCRICAO | [09:08] Diego |
| RFC-Q-06 | docs/RFC.md | Questão em aberto | Leitura de "5 tentativas" × 5 intervalos (~15 h) | TRANSCRICAO | [09:17] Diego |
| RFC-Q-07 | docs/RFC.md | Questão em aberto (revisão) | Assinatura durante o grace period de 24 h | TRANSCRICAO | [09:21] Sofia |
| RFC-Q-08 | docs/RFC.md | Questão em aberto (revisão) | X-Timestamp fora do material assinado (HMAC só do corpo) | TRANSCRICAO | [09:44] Diego |
| RFC-Q-09 | docs/RFC.md | Questão em aberto (revisão) | Ordem por pedido quando há retry pendente | TRANSCRICAO | [09:12] Diego |
| RFC-Q-10 | docs/RFC.md | Questão em aberto (revisão) | Retenção de histórico/DLQ após DELETE do webhook | TRANSCRICAO | [09:33] Bruno |
| RFC-Q-11 | docs/RFC.md | Questão em aberto (revisão) | Criação de pedido (create) não passa por changeStatus e não gera evento | CODIGO | src/modules/orders/order.service.ts |
| RFC-Q-12 | docs/RFC.md | Questão em aberto (revisão) | SSRF: além de https, bloquear destinos internos? | TRANSCRICAO | [09:23] Sofia |
| RFC-IMP-01 | docs/RFC.md | Impacto | Um passo a mais na transação do changeStatus | TRANSCRICAO | [09:41] Bruno |
| RFC-IMP-02 | docs/RFC.md | Impacto | Novo processo para implantar (npm run worker) | TRANSCRICAO | [09:11] Larissa |
| RFC-RISK-01 | docs/RFC.md | Risco | Prazo fim de novembro com 3 sprints | TRANSCRICAO | [09:46] Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox em MySQL decidido | TRANSCRICAO | [09:08] Larissa |
| ADR-001-CTX | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | changeStatus roda em prisma.$transaction com estoque, update e history | CODIGO | src/modules/orders/order.service.ts |
| ADR-001-D1 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Insert na webhook_outbox na mesma transação SQL | TRANSCRICAO | [09:06] Diego |
| ADR-001-D2 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Índices em status e created_at; batch pequeno | TRANSCRICAO | [09:08] Diego |
| ADR-001-ALT1 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa | Síncrono travaria status de outros pedidos e não pode dar rollback | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT2 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa | Redis Cluster é overengineering para time pequeno | TRANSCRICAO | [09:07] Diego |
| ADR-001-CTX2 | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | Infra atual só sobe MySQL 8.0 | CODIGO | docker-compose.yml |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em polling de 2 s, registrado como decisão | TRANSCRICAO | [09:10] Larissa |
| ADR-002-D1 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Polling em loop a cada 2 s | TRANSCRICAO | [09:09] Diego |
| ADR-002-D2 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Processo separado; se a API reinicia não perde o worker | TRANSCRICAO | [09:11] Diego |
| ADR-002-D3 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Entry-point src/worker.ts (novo) e script npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-002-D4 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Mesmo banco e mesmo Prisma; PrismaClient separado por processo | TRANSCRICAO | [09:30] Bruno |
| ADR-002-D5 | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Single-worker; ordem implícita por order_id | TRANSCRICAO | [09:12] Diego |
| ADR-002-D6 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Lógica em webhook.worker.ts/webhook.processor.ts (novo) dentro do módulo | TRANSCRICAO | [09:28] Bruno |
| ADR-002-ALT1 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa | Trigger do banco para ser reativo | TRANSCRICAO | [09:09] Bruno |
| ADR-002-CODE1 | docs/adrs/ADR-002-worker-separado-em-polling.md | Padrão existente | Entry-point com bootstrap e SIGINT/SIGTERM | CODIGO | src/server.ts |
| ADR-002-CODE2 | docs/adrs/ADR-002-worker-separado-em-polling.md | Padrão existente | createPrismaClient / prisma exportado | CODIGO | src/config/database.ts |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-D1 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Backoff exponencial com teto e DLQ | TRANSCRICAO | [09:15] Diego |
| ADR-003-D2 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Tabela webhook_dead_letter separada com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| ADR-003-D3 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | POST /admin/webhooks/dead-letter/:id/replay recoloca como pendente | TRANSCRICAO | [09:18] Diego |
| ADR-003-D4 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Replay exige ADMIN reaproveitando requireRole | TRANSCRICAO | [09:36] Larissa |
| ADR-003-ALT1 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa | 3 tentativas (proposta do Bruno) | TRANSCRICAO | [09:16] Bruno |
| ADR-003-ALT2 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa | Marcar failed na própria outbox × tabela separada | TRANSCRICAO | [09:17] Larissa |
| ADR-003-CODE | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Padrão existente | requireRole(...roles) | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-004 | docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-D1 | docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md | Decisão | Assinatura no header X-Signature | TRANSCRICAO | [09:20] Sofia |
| ADR-004-D2 | docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md | Decisão | Secret única por endpoint, não global | TRANSCRICAO | [09:21] Sofia |
| ADR-004-D3 | docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md | Restrição | TLS obrigatório, validado no schema Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-004-D4 | docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md | Decisão | Revisão do código de HMAC e geração de secret | TRANSCRICAO | [09:46] Sofia |
| ADR-004-ALT1 | docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md | Alternativa | Algoritmo: SHA-256 é padrão de mercado | TRANSCRICAO | [09:20] Bruno |
| ADR-004-CODE | docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md | Padrão existente | Node ≥ 20 (crypto nativo); sem dependência nova | CODIGO | package.json |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id para dedup no cliente | TRANSCRICAO | [09:26] Larissa |
| ADR-005-D1 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Cliente pode receber o mesmo evento duas vezes | TRANSCRICAO | [09:24] Diego |
| ADR-005-D2 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | UUID gerado na entrada da outbox, enviado em X-Event-Id | TRANSCRICAO | [09:25] Diego |
| ADR-005-D3 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | IDs em UUID seguindo o padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-005-CODE | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Padrão existente | @default(uuid()) @db.Char(36) em todos os models | CODIGO | prisma/schema.prisma |
| ADR-005-CON | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Trade-off | Joga responsabilidade de dedup para o cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso máximo: AppError, Pino, error middleware, módulos, Zod, códigos | TRANSCRICAO | [09:30] Larissa |
| ADR-006-D1 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Módulo src/modules/webhooks com controller/service/repository/routes/schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-D2 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Códigos WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| ADR-006-D3 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Prefixo WEBHOOK_ para tudo do módulo | TRANSCRICAO | [09:29] Larissa |
| ADR-006-D4 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Pino sem nada novo; error middleware sem mudança | TRANSCRICAO | [09:29] Bruno |
| ADR-006-D5 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Função publishWebhookEvent(tx, order, fromStatus, toStatus) | TRANSCRICAO | [09:41] Bruno |
| ADR-006-ALT1 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Alternativa | Injetar repository inteiro × função pura recebendo tx | TRANSCRICAO | [09:41] Diego |
| ADR-006-CODE1 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Padrão existente | AppError com statusCode e errorCode | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-CODE2 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Padrão existente | InvalidStatusTransitionError, InsufficientStockError, NotFoundError (código fixo) | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-CODE3 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Padrão existente | Error middleware trata AppError, ZodError e Prisma P2002/P2025 | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-CODE4 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Restrição | validate() converte ZodError em VALIDATION_ERROR | CODIGO | src/middlewares/validate.middleware.ts |
| ADR-006-CODE5 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Padrão existente | Composição de dependências em buildControllers | CODIGO | src/app.ts |
| ADR-007 | docs/adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md | Decisão | Snapshot do payload na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-007-D1 | docs/adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md | Decisão | Snapshot confirmado como decidido | TRANSCRICAO | [09:52] Bruno |
| ADR-007-D2 | docs/adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md | Decisão | Payload JSON enxuto, sem items | TRANSCRICAO | [09:43] Diego |
| ADR-007-D3 | docs/adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md | Requisito Não Funcional | 64 KB como teto | TRANSCRICAO | [09:24] Diego |
| ADR-007-D4 | docs/adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md | Decisão | Filtro de eventos na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| ADR-007-ALT1 | docs/adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md | Alternativa | Guardar só order_id e renderizar no envio | TRANSCRICAO | [09:51] Bruno |
| ADR-007-ALT2 | docs/adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md | Alternativa | Truncar × errar em payload grande | TRANSCRICAO | [09:23] Sofia |
| ADR-007-CODE | docs/adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md | Padrão existente | Enum OrderStatus e campos orderNumber, customerId, totalCents | CODIGO | prisma/schema.prisma |
| FDD-OT-01 | docs/FDD.md | Objetivo técnico | Zero dependências/infra novas | TRANSCRICAO | [09:29] Bruno |
| FDD-DADOS-01 | docs/FDD.md | Modelo de dados | Tabela webhooks: url, secret, customer_id, ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-02 | docs/FDD.md | Modelo de dados | Outbox com status pendente/processando/falhou/entregue | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-03 | docs/FDD.md | Modelo de dados | webhook_deliveries sustenta o histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-04 | docs/FDD.md | Modelo de dados | webhook_dead_letter com payload, motivo, timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-05 | docs/FDD.md | Modelo de dados | IDs UUID Char(36) | TRANSCRICAO | [09:51] Larissa |
| FDD-DADOS-06 | docs/FDD.md | Modelo de dados | orderId sem FK porque pedidos PENDING/CANCELLED podem ser excluídos | CODIGO | src/modules/orders/order.service.ts |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Insert na outbox dentro do changeStatus; falha = rollback | TRANSCRICAO | [09:40] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Se nenhum webhook assina o status, nada é inserido | TRANSCRICAO | [09:34] Bruno |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Worker lê pendentes mais antigos em batch pequeno e marca | TRANSCRICAO | [09:08] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Recuperação de PROCESSING no boot (reenvio aceito pelo at-least-once) | TRANSCRICAO | [09:24] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | Agenda de retry 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-06 | docs/FDD.md | Fluxo | Mover para DLQ e replay recolocando na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-07 | docs/FDD.md | Fluxo | Replay audita quem executou | TRANSCRICAO | [09:36] Sofia |
| FDD-FLUXO-08 | docs/FDD.md | Fluxo | Rotação: antiga válida 24 h em paralelo | TRANSCRICAO | [09:21] Sofia |
| FDD-FLUXO-09 | docs/FDD.md | Fluxo | Verificação de 64 KB no worker (erro na transação desfaria a mudança de status) | TRANSCRICAO | [09:04] Bruno |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | E1 POST /webhooks (url, events, secret na resposta) | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | E2 GET /webhooks?customerId — listar webhooks do customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | E3 PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | E4 DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | E5 POST /webhooks/:id/rotate-secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | E6 GET /webhooks/:id/deliveries (últimos 100) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | E7 POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Restrição | JWT é de usuário operador, não do cliente | TRANSCRICAO | [09:32] Bruno |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Header X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | X-Webhook-Id aceito | TRANSCRICAO | [09:45] Diego |
| FDD-CONTRATO-11 | docs/FDD.md | Contrato | X-Timestamp para o cliente detectar replay attack | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-12 | docs/FDD.md | Contrato | Payload enxuto mantido | TRANSCRICAO | [09:44] Bruno |
| FDD-CONTRATO-13 | docs/FDD.md | Padrão existente | Listagem com paginated() | CODIGO | src/shared/http/response.ts |
| FDD-CONTRATO-14 | docs/FDD.md | Padrão existente | customerId no body/query como em createOrderSchema/listOrdersQuerySchema | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-CONTRATO-15 | docs/FDD.md | Restrição | Nenhuma transição leva a PENDING → não é status assinável | CODIGO | src/modules/orders/order.status.ts |
| FDD-CONTRATO-16 | docs/FDD.md | Contrato | order_number no formato ORD-NNNNNN | CODIGO | src/modules/orders/order.service.ts |
| FDD-ERRO-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND / WEBHOOK_INVALID_URL / WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE (erro, não truncar) | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-03 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT após 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-04 | docs/FDD.md | Erro | WEBHOOK_MAX_ATTEMPTS_EXCEEDED → falha permanente → DLQ | TRANSCRICAO | [09:15] Diego |
| FDD-ERRO-05 | docs/FDD.md | Erro | FORBIDDEN no replay por não-ADMIN | TRANSCRICAO | [09:36] Sofia |
| FDD-ERRO-06 | docs/FDD.md | Erro | WEBHOOK_OUTBOX_INSERT_FAILED provoca rollback | TRANSCRICAO | [09:41] Diego |
| FDD-ERRO-07 | docs/FDD.md | Restrição | NotFoundError fixa NOT_FOUND → subclasse própria para WEBHOOK_NOT_FOUND | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERRO-08 | docs/FDD.md | Restrição | Error middleware só loga erros que não são AppError | CODIGO | src/middlewares/error.middleware.ts |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout 10 s; cliente lento = falha e retry | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Fallback por e-mail adiado | TRANSCRICAO | [09:37] Larissa |
| FDD-RES-03 | docs/FDD.md | Resiliência | Shutdown gracioso espelhando o server | CODIGO | src/server.ts |
| FDD-RES-04 | docs/FDD.md | Resiliência | Indisponibilidade de 2 h justifica a janela longa | TRANSCRICAO | [09:16] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs com Pino existente | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | redactPaths a estender com secret/assinatura | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Motivação: secret vazada em log de cliente | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Log de auditoria do replay | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-05 | docs/FDD.md | Observabilidade | Métrica de latência ponta a ponta contra SLA de 10 s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-06 | docs/FDD.md | Observabilidade | Métrica de volume por cliente para decidir rate limiting | TRANSCRICAO | [09:39] Diego |
| FDD-OBS-07 | docs/FDD.md | Observabilidade | Correlação por requestId (X-Request-Id) | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-DEP-01 | docs/FDD.md | Dependência | Node ≥ 20, prisma 5.22, zod, pino, uuid já instalados | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Dependência | Worker precisa de DATABASE_URL e JWT_SECRET (validados na carga) | CODIGO | src/config/env.ts |
| FDD-DEP-03 | docs/FDD.md | Dependência | Revisão de segurança antes de subir | TRANSCRICAO | [09:49] Sofia |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus chama publishWebhookEvent após orderStatusHistory.create | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Montar /webhooks e /admin/webhooks em buildApiRouter | CODIGO | src/routes/index.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Instanciar módulo em buildControllers | CODIGO | src/app.ts |
| FDD-INT-04 | docs/FDD.md | Integração | authenticate + requireRole('ADMIN') como em user.routes.ts | CODIGO | src/modules/users/user.routes.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Worker reutiliza prisma (nova instância por processo) | CODIGO | src/config/database.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Novas classes exportadas no barrel de erros | CODIGO | src/shared/errors/index.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Novos models e relação Customer.webhooks | CODIGO | prisma/schema.prisma |
| FDD-INT-08 | docs/FDD.md | Integração | beforeEach limpa tabelas novas antes de customers | CODIGO | tests/setup.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Helper createTestWebhook ao lado de createTestCustomer | CODIGO | tests/helpers/factories.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Script npm run worker | TRANSCRICAO | [09:11] Larissa |
| FDD-INT-11 | docs/FDD.md | Integração | Mesmo banco, mesma stack, processo diferente | TRANSCRICAO | [09:11] Diego |
| FDD-INT-12 | docs/FDD.md | Integração | Mesmo Prisma client/banco precisa ser usado pelo worker | TRANSCRICAO | [09:11] Bruno |
| FDD-CAT-01 | docs/FDD.md | Critério de aceite técnico | Rollback do insert na outbox mantém status inalterado | TRANSCRICAO | [09:41] Diego |
| FDD-CAT-02 | docs/FDD.md | Critério de aceite técnico | Transição inválida não gera evento (cenário PENDING → SHIPPED) | CODIGO | tests/orders.test.ts |
| FDD-CAT-03 | docs/FDD.md | Critério de aceite técnico | Histórico limitado a 100 itens | TRANSCRICAO | [09:34] Marcos |
| FDD-CAT-04 | docs/FDD.md | Critério de aceite técnico | Revisão de segurança aprovada | TRANSCRICAO | [09:46] Sofia |
| FDD-RISK-01 | docs/FDD.md | Risco | Crescimento da outbox sem arquivamento | TRANSCRICAO | [09:08] Diego |
| FDD-RISK-02 | docs/FDD.md | Risco | Inversão de ordem em retries; clientes não exigem ordem global | TRANSCRICAO | [09:14] Marcos |
| FDD-RISK-03 | docs/FDD.md | Risco | Perda de garantia de ordem ao escalar | TRANSCRICAO | [09:13] Bruno |
| FDD-RISK-04 | docs/FDD.md | Risco | Performance da tabela com muitos eventos | TRANSCRICAO | [09:07] Bruno |
| FDD-RISK-05 | docs/FDD.md | Risco | Secret vazada (secret por endpoint) | TRANSCRICAO | [09:21] Sofia |
| PRD-CA-02 | docs/PRD.md | Critério de aceitação | Notificação recebida em menos de 10 s | TRANSCRICAO | [09:02] Marcos |
| PRD-CA-03 | docs/PRD.md | Critério de aceitação | Status não assinado ou webhook inativo não geram notificação | TRANSCRICAO | [09:34] Bruno |
| PRD-CA-04 | docs/PRD.md | Critério de aceitação | Mudança de status que falha não gera notificação | TRANSCRICAO | [09:06] Diego |
| PRD-CA-05 | docs/PRD.md | Critério de aceitação | Assinatura válida e mesmo identificador em reenvios | TRANSCRICAO | [09:25] Diego |
| PRD-CA-06 | docs/PRD.md | Critério de aceitação | Retentativas 1m/5m/30m/2h/12h e depois DLQ | TRANSCRICAO | [09:48] Larissa |
| PRD-CA-07 | docs/PRD.md | Critério de aceitação | ADMIN reprocessa; OPERATOR recebe acesso negado | TRANSCRICAO | [09:36] Larissa |
| PRD-CA-08 | docs/PRD.md | Critério de aceitação | Secret antiga verificável por 24 h após rotação | TRANSCRICAO | [09:22] Sofia |
| PRD-CA-09 | docs/PRD.md | Critério de aceitação | URL http recusada | TRANSCRICAO | [09:23] Sofia |
| PRD-CA-10 | docs/PRD.md | Critério de aceitação | Histórico com até 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-TEST-03 | docs/PRD.md | Estratégia de teste | Testes unitários com Vitest já configurado | CODIGO | vitest.config.ts |
| PRD-TEST-04 | docs/PRD.md | Estratégia de teste | Validação com os clientes após comunicação do prazo | TRANSCRICAO | [09:47] Marcos |
| PRD-TEST-05 | docs/PRD.md | Estratégia de teste | Medir impacto pós-lançamento antes de decidir e-mail | TRANSCRICAO | [09:37] Larissa |
| RFC-PROP-04 | docs/RFC.md | Princípio | Ordem por pedido basta; ordenação global não é objetivo | TRANSCRICAO | [09:14] Marcos |
| RFC-RISK-02 | docs/RFC.md | Risco | Worker parado acumula eventos | TRANSCRICAO | [09:11] Diego |
| RFC-RISK-03 | docs/RFC.md | Risco | Rajadas sem rate limiting | TRANSCRICAO | [09:38] Diego |
| RFC-RISK-04 | docs/RFC.md | Risco | Cliente não deduplica; mitigação no portal | TRANSCRICAO | [09:26] Marcos |
| RFC-RISK-05 | docs/RFC.md | Risco | Vazamento de secret | TRANSCRICAO | [09:22] Diego |
| RFC-NEXT-01 | docs/RFC.md | Próximo passo | Agendar revisão de segurança antes de subir | TRANSCRICAO | [09:49] Sofia |
| FDD-DADOS-07 | docs/FDD.md | Modelo de dados | Uma linha de outbox por webhook inscrito (X-Webhook-Id por cadastro) | TRANSCRICAO | [09:44] Sofia |
| FDD-DADOS-08 | docs/FDD.md | Modelo de dados | events como Json porque o provider é MySQL | CODIGO | prisma/schema.prisma |
| FDD-ERRO-09 | docs/FDD.md | Erro | WEBHOOK_INVALID_EVENT_FILTER para lista de status inválida | TRANSCRICAO | [09:33] Marcos |
| FDD-ERRO-10 | docs/FDD.md | Erro | WEBHOOK_INACTIVE (estado ativo do cadastro) | TRANSCRICAO | [09:21] Sofia |
| FDD-ERRO-11 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND / ALREADY_REPLAYED no replay | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-12 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_HTTP_ERROR / NETWORK_ERROR (cliente offline) → retry | TRANSCRICAO | [09:14] Larissa |
| FDD-ERRO-13 | docs/FDD.md | Padrão existente | Reuso de UNAUTHORIZED, FORBIDDEN, NOT_FOUND, VALIDATION_ERROR | CODIGO | src/shared/errors/http-errors.ts |
| FDD-RES-05 | docs/FDD.md | Resiliência | Reenvio após crash aceito pela garantia at-least-once | TRANSCRICAO | [09:26] Larissa |
| FDD-RES-06 | docs/FDD.md | Resiliência | Cliente lento não pode travar os demais (paralelismo por webhook) | TRANSCRICAO | [09:04] Bruno |
| FDD-RES-07 | docs/FDD.md | Resiliência | Payload > 64 KB não é enviado | TRANSCRICAO | [09:24] Larissa |
| FDD-RES-08 | docs/FDD.md | Resiliência | Backoff sem jitter por haver um único worker | TRANSCRICAO | [09:12] Diego |
| FDD-OBS-08 | docs/FDD.md | Observabilidade | Nomes de evento de log em snake_case (server_started) | CODIGO | src/server.ts |
| FDD-OBS-09 | docs/FDD.md | Observabilidade | Alerta de worker parado (processo separado) | TRANSCRICAO | [09:11] Diego |
| FDD-OBS-10 | docs/FDD.md | Observabilidade | Sem OpenTelemetry: nada de lib nova | TRANSCRICAO | [09:29] Bruno |
| FDD-INT-13 | docs/FDD.md | Integração | Schemas do módulo aplicados via validate() | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-14 | docs/FDD.md | Integração | Error middleware pega erros do módulo sem mudança | TRANSCRICAO | [09:29] Bruno |
| FDD-INT-15 | docs/FDD.md | Integração | Máquina de estados de pedidos sem alteração | CODIGO | src/modules/orders/order.status.ts |
| FDD-CAT-05 | docs/FDD.md | Critério de aceite técnico | Envio com os 5 headers e assinatura verificável | TRANSCRICAO | [09:44] Diego |
| FDD-CAT-06 | docs/FDD.md | Critério de aceite técnico | Agenda de retry e 6ª falha para DLQ | TRANSCRICAO | [09:17] Larissa |
| FDD-CAT-07 | docs/FDD.md | Critério de aceite técnico | X-Event-Id idêntico entre tentativas e replay | TRANSCRICAO | [09:25] Diego |
| FDD-CAT-08 | docs/FDD.md | Critério de aceite técnico | Payload > 64 KB vai para DLQ sem envio | TRANSCRICAO | [09:24] Larissa |
| FDD-CAT-09 | docs/FDD.md | Critério de aceite técnico | URL http → 400 | TRANSCRICAO | [09:23] Sofia |
| FDD-CAT-10 | docs/FDD.md | Critério de aceite técnico | Replay: OPERATOR 403, ADMIN 202 + log de auditoria | TRANSCRICAO | [09:36] Sofia |
| FDD-CAT-11 | docs/FDD.md | Critério de aceite técnico | Rotação válida com as duas secrets por 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-CAT-12 | docs/FDD.md | Critério de aceite técnico | Secret ausente de listagens e logs | TRANSCRICAO | [09:22] Diego |
| FDD-CAT-13 | docs/FDD.md | Critério de aceite técnico | Suíte existente continua verde | CODIGO | vitest.config.ts |
| FDD-CAT-14 | docs/FDD.md | Critério de aceite técnico | Worker reiniciado reprocessa PROCESSING | TRANSCRICAO | [09:11] Diego |
| PRD-OOS-04b | docs/PRD.md | Fora de escopo | Pergunta de escopo que originou a decisão "só outbound" | TRANSCRICAO | [09:02] Sofia |
| PRD-OOS-02b | docs/PRD.md | Fora de escopo | Proposta de dashboard visual para o cliente (depois descartada) | TRANSCRICAO | [09:39] Marcos |
| PRD-OOS-01b | docs/PRD.md | Fora de escopo | E-mail registrado como "futuro" pelo PM | TRANSCRICAO | [09:38] Marcos |
| PRD-NFR-01b | docs/PRD.md | Requisito Não Funcional | PM valida polling de 2 s como suficiente | TRANSCRICAO | [09:10] Marcos |
| PRD-NFR-11b | docs/PRD.md | Requisito Não Funcional | PM pergunta se o CRUD pode ser de qualquer role autenticada | TRANSCRICAO | [09:36] Marcos |
| PRD-FR-08b | docs/PRD.md | Requisito Funcional | PM levanta a definição do formato do payload | TRANSCRICAO | [09:43] Marcos |
| PRD-FR-09b | docs/PRD.md | Requisito Funcional | PM levanta a definição dos headers do request | TRANSCRICAO | [09:44] Marcos |
| PRD-DEP-04b | docs/PRD.md | Dependência | PM atualiza os clientes sobre o plano no mesmo dia | TRANSCRICAO | [09:49] Marcos |
| PRD-CTX-03b | docs/PRD.md | Restrição | Pergunta sobre tempo real × delay aceitável | TRANSCRICAO | [09:01] Bruno |
| FDD-RES-01b | docs/FDD.md | Resiliência | Segurança levanta o timeout da chamada HTTP do worker | TRANSCRICAO | [09:42] Sofia |
| FDD-RES-01c | docs/FDD.md | Resiliência | Segurança aprova timeout de 10 s | TRANSCRICAO | [09:42] Sofia |
| ADR-001-CTX3 | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | Questão inicial: disparo síncrono no service de orders × fila/outbox | TRANSCRICAO | [09:03] Larissa |
| ADR-002-D7 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Pergunta: worker usa o mesmo PrismaClient ou um separado? | TRANSCRICAO | [09:29] Diego |
| ADR-002-D8 | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Questão de ordering entre PAID → PROCESSING → SHIPPED | TRANSCRICAO | [09:12] Larissa |
| ADR-003-D5 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Cinco tentativas aceitas pela Tech Lead | TRANSCRICAO | [09:16] Larissa |
| ADR-003-D6 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Pergunta sobre quem reprocessa a DLQ (origem do endpoint admin) | TRANSCRICAO | [09:18] Bruno |
| ADR-003-D7 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Endpoint admin de replay registrado | TRANSCRICAO | [09:19] Larissa |
| ADR-003-D8 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Pergunta se o replay exige role ADMIN do JWT | TRANSCRICAO | [09:35] Larissa |
| ADR-005-D4 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Pergunta: como o cliente diferencia duplicatas? | TRANSCRICAO | [09:25] Bruno |
| ADR-005-D5 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Pergunta: id auto incremental ou UUID na outbox | TRANSCRICAO | [09:51] Diego |
| ADR-007-D5 | docs/adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md | Decisão | Pergunta: filtrar na inserção da outbox ou no envio | TRANSCRICAO | [09:34] Diego |
| ADR-007-D6 | docs/adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md | Decisão | Snapshot na inserção confirmado | TRANSCRICAO | [09:52] Diego |
| RFC-TLDR-02 | docs/RFC.md | Decisão | Resumo final confirmado pela Plataforma | TRANSCRICAO | [09:49] Diego |
| RFC-TLDR-03 | docs/RFC.md | Decisão | Resumo final confirmado por Pedidos | TRANSCRICAO | [09:49] Bruno |
