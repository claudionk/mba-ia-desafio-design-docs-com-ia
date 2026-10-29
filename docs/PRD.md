# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Stakeholders** | Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |
| **Status** | Aprovado para implementação |
| **Data** | 2026-10-29 |
| **Prazo-alvo** | Fim de novembro/2026 (3 sprints, incluindo revisão de segurança) |
| **Documentos relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

---

## 1. Resumo e contexto da feature

O OMS passará a **avisar automaticamente os sistemas dos clientes B2B quando o status de um pedido mudar**, por meio de webhooks HTTP (chamadas que a nossa plataforma faz para uma URL cadastrada pelo cliente). Cada cliente cadastra um ou mais endpoints, escolhe de quais mudanças de status quer saber e passa a receber notificações assinadas em poucos segundos, sem precisar consultar a nossa API repetidamente.

O pedido veio formalmente de três clientes B2B — **Atlas Comercial, MaxDistribuição e Nova Cargo** (Marcos, [09:00]). Hoje o OMS não tem nenhum mecanismo de notificação externa.

## 2. Problema e motivação

- **Integração lenta e cara para o cliente:** sem notificações, os clientes fazem polling em `GET /orders` "de tempos em tempos" para descobrir mudanças (Marcos, [09:00]).
- **Risco comercial:** a Atlas indicou que pode migrar para um concorrente se a solução não for entregue até o fim do trimestre (Marcos, [09:00]).
- **Expectativa clara de latência:** para os clientes, "tempo real" é qualquer coisa abaixo de 10 segundos. O que importa é não precisar atualizar manualmente (Marcos, [09:02]).

## 3. Público-alvo e cenários de uso

**Personas**

| Persona | Necessidade |
| --- | --- |
| **Integrador do cliente B2B** (ex.: time de TI da Atlas) | Receber mudanças de status em < 10 s, verificar que a chamada é legítima e deduplicar reenvios. |
| **Usuário do OMS que representa o cliente** (role `OPERATOR` ou `ADMIN`) | Cadastrar, editar e remover webhooks, rotacionar a secret e consultar o histórico de entregas pela API (Marcos, [09:32]). |
| **Administrador da plataforma** (role `ADMIN`) | Reprocessar eventos que falharam definitivamente, com auditoria (Sofia, [09:36]). |

**Cenários**

1. **Acompanhar expedição:** a Nova Cargo cadastra um webhook só para `SHIPPED` e `DELIVERED`. Quando um pedido é despachado, o sistema dela recebe o evento em segundos e dispara o próprio fluxo logístico (Marcos, [09:33]).
2. **Cliente em manutenção planejada:** o endpoint da MaxDistribuição fica 2 h fora do ar. Os eventos do período são retentados automaticamente e entregues quando o endpoint volta (Diego, [09:16]).
3. **Secret exposta:** a Atlas descobre que a secret vazou em log. Ela pede uma nova secret pela API e tem 24 h para atualizar seus sistemas, sem perder notificações (Sofia, [09:21]; Diego, [09:22]).
4. **Falha definitiva:** depois de ~15 h de tentativas, um evento vai para a fila de falhas. Um admin corrige a causa com o cliente e reprocessa o evento (Diego, [09:18]).
5. **Diagnóstico:** o integrador consulta as últimas 100 entregas do seu webhook para entender por que não recebeu um evento (Marcos, [09:34]).

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta |
| --- | --- | --- | --- |
| OBJ-01 | Notificar mudanças de status em "tempo real" | Latência entre a mudança de status e a 1ª tentativa de entrega (p95) | **< 10 s** (Marcos, [09:02]); o desenho prevê ≤ 2 s de espera do polling (Diego, [09:09]) |
| OBJ-02 | Não perder notificações | % de mudanças de status (para status assinados) com evento registrado | **100%** — atomicidade com a mudança de status (Diego, [09:06]) |
| OBJ-03 | Tolerar indisponibilidade do cliente | Janela coberta por retentativas automáticas | **≈ 15 h** (5 retentativas: 1m/5m/30m/2h/12h — Diego, [09:17]) |
| OBJ-04 | Reter os clientes que pediram a feature | Clientes solicitantes com webhook em produção | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) até o fim de novembro (Marcos, [09:45]) |
| OBJ-05 | Entregar no prazo comprometido | Sprints até produção | **≤ 3 sprints**, incluindo revisão de segurança (Larissa, [09:46]) |

## 5. Escopo

### 5.1 Incluso

- Notificação outbound de **mudança de status de pedido** (`order.status_changed`).
- API para o cliente **gerenciar webhooks**: cadastrar, listar, editar, remover, rotacionar secret e ver histórico de entregas.
- **Filtro por status** em cada webhook.
- **Retentativas automáticas** e **fila de falhas definitivas (DLQ)** com **reprocessamento manual por admin**.
- **Assinatura** de cada notificação e identificador único de evento para deduplicação.

### 5.2 Fora de escopo

| Item | Situação | Origem |
| --- | --- | --- |
| **E-mail avisando o cliente quando o webhook falha repetidamente** | **Adiado** para a próxima fase, depois de medir impacto | Marcos [09:37]; Larissa [09:37] |
| **Dashboard visual / painel do cliente** para ver webhooks | **Descartado** desta feature — é projeto separado do time de frontend; só endpoints de API | Marcos [09:39]; Larissa [09:40] |
| **Rate limiting de envio por cliente** | **Adiado** — observar em produção e decidir depois | Diego [09:38]–[09:39]; Larissa [09:39] |
| **Webhooks de entrada** (cliente enviando para o OMS) | **Descartado** — os clientes só querem receber | Sofia [09:02] (pergunta de escopo); Marcos [09:02]; Sofia [09:03] |
| **Arquivamento de eventos entregues** (~30 dias) | **Fora do escopo** desta feature | Diego [09:08] |
| **Ordenação global de eventos** e múltiplos workers | **Adiado** — só ordem por pedido, com worker único | Diego [09:12]–[09:13]; Marcos [09:14] |
| **Entrega exactly-once** | **Descartado** — at-least-once com deduplicação pelo cliente | Diego [09:25] |
| **Itens do pedido na notificação** | **Descartado** — o cliente consulta `GET /orders/:id` se precisar | Diego [09:43] |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| PRD-FR-01 | O sistema deve permitir **cadastrar** um webhook informando a URL, o cliente (`customerId` no corpo da requisição, não derivado do token) e a lista de status a receber. | Marcos [09:31]; Larissa [09:32] |
| PRD-FR-02 | A **secret** do webhook deve ser **gerada pela plataforma** e devolvida ao cliente na criação. | Marcos [09:31] |
| PRD-FR-03 | O sistema deve permitir **listar** os webhooks de um cliente. | Bruno [09:33] |
| PRD-FR-04 | O sistema deve permitir **editar** um webhook (URL, status assinados, ativo/inativo). | Bruno [09:33]; Sofia [09:21] |
| PRD-FR-05 | O sistema deve permitir **remover** um webhook. | Bruno [09:33] |
| PRD-FR-06 | Cada webhook deve receber **apenas** os status que assinou (ex.: só `SHIPPED` e `DELIVERED`). | Marcos [09:33]; Bruno [09:34] |
| PRD-FR-07 | Toda **mudança de status** de pedido deve gerar uma notificação para os webhooks ativos do cliente que assinam o novo status. | Marcos [09:00]; Bruno [09:40] |
| PRD-FR-08 | A notificação deve conter identificador do evento, tipo, data/hora, pedido (id e número), status anterior e novo, cliente e valor total — **sem os itens**. | Diego [09:43] |
| PRD-FR-09 | Cada notificação deve ser **assinada** para que o cliente verifique origem e integridade, e trazer identificador do evento, do webhook e horário de envio. | Sofia [09:19]–[09:20]; Diego [09:44]; Sofia [09:44] |
| PRD-FR-10 | O cliente deve poder **solicitar uma nova secret**; a anterior continua válida por **24 h**. | Sofia [09:21] |
| PRD-FR-11 | Notificações que falharem devem ser **retentadas automaticamente** com intervalos crescentes (1m, 5m, 30m, 2h, 12h). | Diego [09:15], [09:17] |
| PRD-FR-12 | Esgotadas as tentativas, o evento deve ir para uma **fila de falhas (DLQ)** com payload, motivo e data. | Diego [09:18] |
| PRD-FR-13 | Um **ADMIN** deve poder **reprocessar** um evento da DLQ, e a ação deve ser **auditada** (quem e quando). | Diego [09:18]; Sofia [09:36]; Larissa [09:36] |
| PRD-FR-14 | O cliente deve poder consultar o **histórico das últimas 100 entregas** de um webhook: sucesso/falha, payload, resposta e tempo de resposta. | Marcos [09:34] |
| PRD-FR-15 | O cadastro e a edição devem **recusar URLs que não sejam `https`**. | Sofia [09:23] |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| PRD-NFR-01 | **Latência:** notificação em < 10 s; espera máxima de 2 s até o evento ser apanhado. | Marcos [09:02]; Diego [09:09]; Larissa [09:10] |
| PRD-NFR-02 | **Consistência:** a mudança de status e o registro do evento acontecem juntos ou não acontecem. | Diego [09:06]; Bruno [09:40] |
| PRD-NFR-03 | **Isolamento:** um cliente lento ou fora do ar não pode atrasar nem impedir mudanças de status. | Bruno [09:04] |
| PRD-NFR-04 | **Garantia de entrega:** at-least-once; o cliente deduplica pelo identificador do evento. | Diego [09:24]–[09:25]; Larissa [09:26] |
| PRD-NFR-05 | **Ordenação:** garantida apenas por pedido e enquanto houver um único processador. | Diego [09:12]; Larissa [09:13] |
| PRD-NFR-06 | **Timeout:** o cliente tem 10 s para responder; depois disso, a tentativa conta como falha. | Diego [09:42] |
| PRD-NFR-07 | **Tamanho:** notificações acima de 64 KB não são enviadas (erro, sem truncamento). | Sofia [09:23]; Diego [09:24]; Larissa [09:24] |
| PRD-NFR-08 | **Segurança:** HMAC-SHA256, secret exclusiva por endpoint, TLS obrigatório. | Sofia [09:20]–[09:23] |
| PRD-NFR-09 | **Disponibilidade:** o envio roda em processo separado da API; reinícios da API não interrompem entregas. | Diego [09:11] |
| PRD-NFR-10 | **Sem infraestrutura nova:** mesmo banco MySQL e mesma stack. | Diego [09:07], [09:11]; Bruno [09:29] |
| PRD-NFR-11 | **Permissões:** gestão de webhooks por qualquer usuário autenticado; reprocessamento só por `ADMIN`. | Sofia [09:36]–[09:37] |

## 8. Decisões e trade-offs principais

| Decisão | Ganho | Custo aceito | ADR |
| --- | --- | --- | --- |
| Registrar o evento numa tabela do próprio banco, na mesma transação da mudança de status | Nenhum evento perdido; nenhuma infraestrutura nova | Latência mínima de até 2 s | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Processo separado que consulta a fila a cada 2 s | Atende < 10 s com simplicidade | Consultas constantes ao banco; vazão de um único processo | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 retentativas em ~15 h e depois DLQ | Cobre manutenções longas sem eventos eternos | Atraso de até ~15 h; reprocessamento manual | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| HMAC-SHA256 com secret por endpoint e rotação | Vazamento isolado; padrão de mercado | Gestão de segredos mais complexa | [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) |
| At-least-once com identificador de evento | Simplicidade e robustez | Cliente precisa deduplicar | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| Construir como um módulo igual aos demais | Velocidade e consistência | Herda limitações dos padrões atuais | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) |
| Notificação enxuta, congelada no momento da mudança | Contrato simples e fiel ao momento do evento | Detalhes exigem uma consulta extra pelo cliente | [ADR-007](adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md) |

## 9. Dependências

| Dependência | Tipo | Origem |
| --- | --- | --- |
| **Revisão de segurança da Sofia** (≥ 2 dias úteis, antes do deploy), com foco em HMAC e geração de secret | Interna — bloqueante para release | Sofia [09:46], [09:49] |
| **Documentação no portal do desenvolvedor** (como integrar, verificar assinatura e deduplicar) | Interna — Produto (Marcos) | Marcos [09:26], [09:40] |
| **Confirmação de prazo com a Atlas** e comunicação aos três clientes | Externa — Produto | Marcos [09:47], [09:49] |
| **Clientes implementam** endpoint `https`, verificação HMAC e deduplicação | Externa | Sofia [09:20]; Diego [09:25] |
| **Fluxo de mudança de status existente** (`PATCH /orders/:id/status`) | Técnica — código existente | Bruno [09:40] |
| **Sessão de revisão do design** com Bruno e Diego antes de codar | Interna | Larissa [09:50] |

## 10. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| R-01 | **Atraso na entrega** além do fim de novembro leva a Atlas a migrar para o concorrente | Média | Alto | Escopo enxuto (e-mail, dashboard e rate limiting fora); estimativa de 3 sprints com a revisão de segurança já incluída; Marcos confirma o prazo com a Atlas (Marcos [09:00], [09:47]; Larissa [09:46]) |
| R-02 | **Cliente não trata duplicatas** e processa a mesma mudança duas vezes | Média | Médio | Identificador de evento em toda notificação; destaque no portal do desenvolvedor (Diego [09:25]; Marcos [09:26]) |
| R-03 | **Vazamento de secret** de um cliente | Média | Alto | Secret por endpoint; rotação com 24 h de transição; nunca logar a secret; revisão da Sofia (Sofia [09:21]; Diego [09:22]) |
| R-04 | **Rajadas** de notificações sobrecarregam o endpoint do cliente (sem rate limiting) | Média | Médio | Monitorar volume por cliente e decidir na próxima fase (Diego [09:38]–[09:39]) |
| R-05 | **Indisponibilidade longa** do cliente (> 15 h) faz eventos caírem na DLQ sem aviso automático | Baixa | Médio | Reprocessamento manual por admin; aviso por e-mail avaliado na próxima fase (Diego [09:18]; Larissa [09:37]) |
| R-06 | **Processo de envio parado** atrasa todas as notificações | Baixa | Alto | Eventos ficam guardados e saem quando o processo volta; alerta sobre a idade do evento pendente mais antigo (FDD §9) |

## 11. Critérios de aceitação

1. Ao mudar o status de um pedido para um status assinado, o webhook do cliente recebe a notificação em **menos de 10 s** (PRD-FR-07, PRD-NFR-01).
2. Mudanças para status **não** assinados ou de webhooks **inativos** não geram notificação (PRD-FR-06).
3. Se a mudança de status falhar (ex.: transição inválida, estoque insuficiente), **nenhuma** notificação é gerada (PRD-NFR-02).
4. Toda notificação traz assinatura válida e identificador de evento; reenvios do mesmo evento trazem o **mesmo** identificador (PRD-FR-09, PRD-NFR-04).
5. Um endpoint fora do ar recebe retentativas em 1m, 5m, 30m, 2h e 12h; depois disso, o evento aparece na DLQ (PRD-FR-11, PRD-FR-12).
6. Um `ADMIN` consegue reprocessar um evento da DLQ; um `OPERATOR` recebe acesso negado; a ação fica registrada (PRD-FR-13).
7. Após a rotação, notificações são verificáveis com a secret nova e com a antiga por 24 h (PRD-FR-10).
8. URL `http://` é recusada no cadastro e na edição (PRD-FR-15).
9. O histórico mostra até as últimas 100 entregas, com resultado, payload, resposta e tempo (PRD-FR-14).
10. Revisão de segurança aprovada pela Sofia antes do deploy.

## 12. Estratégia de testes e validação

| Nível | O que valida | Como |
| --- | --- | --- |
| **Unitário** | Assinatura HMAC, cálculo de backoff, limite de 64 KB, montagem da notificação | Vitest (já usado no projeto) |
| **Integração / API** | CRUD de webhooks, permissões (ADMIN × OPERATOR), validação de `https`, histórico de entregas | Vitest + supertest com MySQL, no padrão de `tests/orders.test.ts` |
| **Integração com pedidos** | Evento criado junto com a mudança de status; rollback não deixa evento; status não assinado não gera evento | Testes sobre `PATCH /orders/:id/status` |
| **Ponta a ponta** | Envio real a um endpoint de teste: entrega, retentativa após falha, DLQ e reprocessamento | Servidor HTTP local simulando o cliente (Larissa [09:46]: "testes ponta a ponta") |
| **Segurança** | Geração de secret, HMAC, rotação, ausência de secret em logs e respostas | Revisão de código da Sofia (≥ 2 dias úteis) |
| **Validação com clientes** | Integração real da Atlas, MaxDistribuição e Nova Cargo | Acompanhamento do Marcos após o go-live; histórico de entregas como evidência |
| **Pós-lançamento** | Latência p95 < 10 s, taxa de sucesso, volume por cliente (insumo para rate limiting/e-mail) | Métricas do FDD §9 |

## 13. Cronograma estimado

| Sprint | Entrega | Origem |
| --- | --- | --- |
| 1 | Modelagem de outbox e DLQ | Larissa [09:46] |
| 2 | Worker e retry | Larissa [09:46] |
| 3 | CRUD de configuração e histórico de entregas (½), integração com pedidos e testes ponta a ponta (½), HMAC/schemas/validações, revisão de segurança da Sofia | Larissa [09:46]; Sofia [09:46] |
