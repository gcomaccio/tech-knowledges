# 🧠 Tech Knowledges — Guia Definitivo de Arquitetura e Engenharia de Software

Repositório central de conhecimento técnico avançado cobrindo os fundamentos essenciais, decisões arquiteturais e padrões de produção para engenharia de software moderna, backend e sistemas distribuídos.

Cada tópico foi construído combinando **profundidade técnica de nível sênior**, **especificações oficiais (RFCs)**, **diagramas práticos** e **analogias do mundo real** para fixação intuitiva dos conceitos.

---

## 📚 Estrutura dos Módulos

```
tech-knowledges/
├── 🌐 APIs.md                   # Métodos HTTP, REST, Idempotência e Versionamento
├── 🗄️ BancoDeDados.md           # Paginação, Problema N+1, Índices, ACID e SQL Injection
├── 🛡️ SegurancaWeb.md           # Sessões vs JWT, OAuth 2.0, Hashing de Senhas, CORS e Rate Limiting
├── ⚙️ ArquiteturaSistemas.md   # Redis, Mensageria, REST/GraphQL/gRPC, Webhooks e Observabilidade
└── 📖 README.md                 # Índice mestre e visão holística do repositório
```

---

## 🗺️ Índice Geral de Questões (1 a 20)

### [🌐 Módulo 1: APIs HTTP e Arquitetura REST](APIs.md)
Guia de padrões arquiteturais de comunicação cliente-servidor sob os padrões das RFCs do IETF.

* **[1. GET, POST, PUT, PATCH e DELETE: quando usar cada um?](APIs.md#1-get-post-put-patch-e-delete-quando-usar-cada-um)**
  * *Conceito:* Semântica de verbos HTTP, métodos seguros (*safe*) vs. não seguros, idempotência e códigos de status recomendados.
  * 💡 *Analogia:* Consultar o cardápio (GET), preencher ficha de matrícula (POST), substituir a mobília da sala (PUT), retocar a pintura da parede (PATCH) e triturar papel (DELETE).
* **[2. O que é uma API Stateless?](APIs.md#2-o-que-é-uma-api-stateless)**
  * *Conceito:* Requisições autocontidas sem sessões na memória do servidor, viabilizando escalabilidade horizontal sem *sticky sessions*.
  * 💡 *Analogia:* Uma máquina automática de café expresso ou pedágio sem cancela que processa cada transação via aproximação de cartão/token.
* **[3. 401 Unauthorized ou 403 Forbidden?](APIs.md#3-401-unauthorized-ou-403-forbidden)**
  * *Conceito:* Autenticação (*"Quem é você?"*) vs. Autorização (*"Você tem permissão para isso?"*), e a obrigatoriedade do cabeçalho `WWW-Authenticate`.
  * 💡 *Analogia da Portaria:* 401 é ser barrado na catraca por estar sem crachá; 403 é passar a catraca com crachá de estagiário e tentar abrir o cofre da diretoria.
* **[4. O que é Idempotência? Qual método não é idempotente?](APIs.md#4-o-que-é-idempotência-qual-método-não-é-idempotente)**
  * *Conceito:* $f(f(x)) = f(x)$. Por que `POST` não é idempotente e como aplicar Chaves de Idempotência (`Idempotency-Key`) em transações financeiras.
  * 💡 *Analogia:* Apertar o botão do 10º andar no elevador 1 ou 30 vezes (idempotente) vs. a alavanca de uma máquina caça-níquel cobrando fichas novas (não idempotente).
* **[5. Como versionar uma API?](APIs.md#5-como-versionar-uma-api)**
  * *Conceito:* Quebra de contrato (*Breaking Changes*) vs. mudanças aditivas. URI Versioning, Custom Headers, Content Negotiation (Accept) e política de `Sunset` (RFC 8594).
  * 💡 *Analogia:* Duas agências bancárias físicas lado a lado (URI) vs. sentar na mesma mesa de restaurante e pedir o cardápio especial de 2026 anotado no bilhete (Header).

---

### [🗄️ Módulo 2: Engenharia de Bancos de Dados & SQL](BancoDeDados.md)
Guia de modelagem, otimização de consultas, isolamento transacional e defesa contra ataques estruturais.

* **[6. Paginação por Offset ou por Cursor?](BancoDeDados.md#6-paginação-por-offset-ou-por-cursor)**
  * *Conceito:* O gargalo $O(N)$ do `LIMIT/OFFSET` em páginas profundas e anomalias de *data drift* vs. a estabilidade e performance $O(1)$ do Keyset Pagination via B-Tree.
  * 💡 *Analogia:* Folhear um livro manualmente desde a folha 1 até a folha 500 (Offset) vs. usar uma fita marcadora de páginas para abrir exatamente no ponto onde parou (Cursor).
* **[7. O que é o Problema N+1?](BancoDeDados.md#7-o-que-é-o-problema-n1)**
  * *Conceito:* Disparo de $1 + N$ queries consecutivas gerado por *lazy loading* desatento em ORMs. Soluções com `JOIN`, *batch fetching* (`IN (...)`) e *DataLoader*.
  * 💡 *Analogia:* Um garçom que anota os pedidos de uma mesa com 10 pessoas e vai 10 vezes consecutivas até a cozinha para trazer um copo por vez ($1 + 10$ viagens).
* **[8. Quando um Índice Ajuda e Quando Atrapalha?](BancoDeDados.md#8-quando-um-índice-ajuda-e-quando-atrapalha)**
  * *Conceito:* Árvores B-Tree e índices cobertores acelerando filtros e ordenações vs. o custo de rebalanceamento contínuo em tabelas de alta escrita e saturação de Buffer Pool.
  * 💡 *Analogia da Enciclopédia:* O índice remissivo ajuda a achar um verbete em segundos; mas ter que reescrever 20 livros de índices diferentes a cada nova folha escrita emperra todo o trabalho.
* **[9. O que é uma Transação ACID?](BancoDeDados.md#9-o-que-é-uma-transação-acid)**
  * *Conceito:* Atomicidade (tudo ou nada), Consistência (constraints válidas), Isolamento (mitigação de *dirty reads/phantoms*) e Durabilidade (garantia via WAL/Redo Log).
  * 💡 *Analogias:* A cabine de teletransporte integral (Atomicidade), regras oficiais do xadrez (Consistência), cabines blindadas de banco (Isolamento) e contratos esculpidos em pedra (Durabilidade).
* **[10. Como evitar SQL Injection?](BancoDeDados.md#10-como-evitar-sql-injection)**
  * *Conceito:* O perigo da interpolação de strings não sanitizadas e a separação física absoluta de código e dados por meio de *Prepared Statements* / Consultas Parametrizadas.
  * 💡 *Analogia:* Um carteiro que lê um bilhete que diz "Queime o galpão" e executa a ordem (SQLi) vs. o carteiro que transporta uma caixa lacrada sem executar nada do conteúdo interno (Prepared Statement).

---

### [🛡️ Módulo 3: Segurança Web & Autenticação](SegurancaWeb.md)
Guia de proteção perimetral, criptografia de credenciais, delegação segura e contenção de tráfego.

* **[11. Sessão com Cookie ou JWT?](SegurancaWeb.md#11-sessão-com-cookie-ou-jwt)**
  * *Conceito:* Stateful (Redis + cookies `HttpOnly`, `Secure`, `SameSite`) com revogação instantânea vs. Stateless (JWT descentralizado). Padrão híbrido corporativo (Access Token curto + Refresh Token rotativo).
  * 💡 *Analogia:* A ficha plástica do guarda-volumes de um clube (Cookie) vs. um passaporte internacional oficial carimbado com selo holográfico (JWT).
* **[12. Como funciona o OAuth 2.0?](SegurancaWeb.md#12-como-funciona-o-oauth-20)**
  * *Conceito:* Autorização delegada (RFC 6749) vs. Autenticação (OIDC). O fluxo moderno e seguro de *Authorization Code Grant com PKCE*.
  * 💡 *Analogia da Chave de Manobrista:* A chave de carro que apenas liga o motor e manobra o veículo, sem destrancar o porta-luvas nem o cofre pessoal.
* **[13. Como guardar senha (bcrypt ou Argon2)?](SegurancaWeb.md#13-como-guardar-senha-bcrypt-ou-argon2)**
  * *Conceito:* Por que SHA-256 é perigoso para senhas. A superioridade do **Argon2id** (padrão ouro OWASP) por exigir alocação física de memória RAM (*memory-hard*), neutralizando farms de GPUs e ASICs.
  * 💡 *Analogia:* Uma fechadura com retardo de tempo (bcrypt) vs. uma fechadura que, além do tempo, exige levantar uma barra de 50 kg de chumbo para cada tentativa de giro (Argon2id).
* **[14. O que é CORS (Cross-Origin Resource Sharing)?](SegurancaWeb.md#14-o-que-é-cors-cross-origin-resource-sharing)**
  * *Conceito:* Mecanismo exclusivo de navegadores derivado da *Same-Origin Policy* (SOP). O papel das requisições preliminares `OPTIONS` (*preflight*) e desmistificação do uso defensivo.
  * 💡 *Analogia da Alfândega:* O fiscal do aeroporto que impede o desembarque de uma carga e envia um rádio (*OPTIONS*) ao Ministério do país de destino para confirmar a autorização prévia.
* **[15. Como limitar requisições (Rate Limiting)?](SegurancaWeb.md#15-como-limitar-requisições-rate-limiting)**
  * *Conceito:* Prevenção contra DoS, força bruta e custos abusivos. Algoritmos *Token Bucket*, *Leaky Bucket* e *Sliding Window*. Status `429 Too Many Requests` e implementação em gateways com Redis.
  * 💡 *Analogia da Balada:* O controle de fluxo com catraca que entrega 10 fichas por hora para cada frequentador, bloqueando novos pedidos até a próxima janela horária.

---

### [⚙️ Módulo 4: Arquitetura de Sistemas, Performance & Observabilidade](ArquiteturaSistemas.md)
Guia de orquestração assíncrona, protocolos de integração de alta performance e telemetria em produção.

* **[16. Onde colocar cache com Redis?](ArquiteturaSistemas.md#16-onde-colocar-cache-com-redis)**
  * *Conceito:* Estratégias *Cache-Aside*, *Write-Through*, travamento distribuído (*Redlock*) e mitigação de *Cache Stampede*, *Penetration* e *Avalanche*.
  * 💡 *Analogia da Mesa:* Deixar os documentos mais consultados abertos diretamente em cima da sua mesa (Redis) em vez de ir ao arquivo de aço no subsolo (banco) a cada minuto.
* **[17. Quando mandar o processamento para uma fila?](ArquiteturaSistemas.md#17-quando-mandar-o-processamento-para-uma-fila)**
  * *Conceito:* Desacoplamento de tarefas pesadas (> 200 ms), integrações com terceiros instáveis, absorção de rajadas (*load leveling*) e padrão HTTP `202 Accepted`.
  * 💡 *Analogia da Padaria vs. Fast-Food:* O atendente que trava o balcão para assar um bolo por 40 min vs. o fast-food que entrega a senha do pedido em 5s e despacha o preparo para a cozinha (fila).
* **[18. REST, GraphQL ou gRPC?](ArquiteturaSistemas.md#18-rest-graphql-ou-grpc)**
  * *Conceito:* Matriz comparativa entre recursos JSON com cache nativo (REST), consultas flexíveis para frontends dinâmicos (GraphQL) e RPC binário sobre HTTP/2 com Protocol Buffers (gRPC).
  * 💡 *Analogias:* Combo fechado da praça de alimentação (REST) vs. buffet a quilo por encomenda personalizada (GraphQL) vs. telégrafo militar binário direto entre bases (gRPC).
* **[19. Como processar um Webhook uma vez só?](ArquiteturaSistemas.md#19-como-processar-um-webhook-uma-vez-só)**
  * *Conceito:* Garantia de entrega *At-Least-Once*, validação de assinatura HMAC, trava atômica de `event_id` no banco/Redis e resposta `200 OK` para interrupção de retentativas.
  * 💡 *Analogia da Portaria:* O porteiro que confere o código de rastreio único de uma encomenda e descarta pacotes duplicados trazidos por engano no mesmo dia.
* **[20. Como monitorar a API com Logs, Métricas e Traces?](ArquiteturaSistemas.md#20-como-monitorar-a-api-com-logs-métricas-e-traces)**
  * *Conceito:* Os 3 Pilares da Observabilidade com OpenTelemetry: Métricas (4 Sinais Dourados), Logs estruturados em JSON e Tracing distribuído com propagação de contexto W3C (`traceparent`).
  * 💡 *Analogia da UTI:* O monitor cardíaco em tempo real (Métricas), o prontuário clínico detalhado pela enfermagem (Logs) e a tomografia com contraste que revela o vaso sanguíneo obstruído (Traces).

---

## 🎯 Matriz Resumo das 20 Perguntas

| # | Pergunta Principal | Área | Tópico-Chave de Engenharia | Analogia Central |
| :-: | :--- | :--- | :--- | :--- |
| **01** | Métodos HTTP | APIs | Semântica, Safety e Idempotência | Cardápio, matrícula e triturador |
| **02** | API Stateless | APIs | Desacoplamento e Escalabilidade Horizontal | Máquina de café de autoatendimento |
| **03** | 401 vs. 403 | APIs | Autenticação vs. Autorização | Crachá inexistente vs. porta da diretoria |
| **04** | Idempotência | APIs | Tolerância a falhas e Idempotency-Key | Botão do elevador vs. caça-níquel |
| **05** | Versionamento de APIs | APIs | Quebra de contrato e cabeçalho `Sunset` | Lojas físicas lado a lado vs. bilhete na comanda |
| **06** | Offset vs. Cursor | Banco de Dados | Complexidade $O(N)$ vs. $O(1)$ em paginação | Folhear página por página vs. marcador de livro |
| **07** | Problema N+1 | Banco de Dados | Eager Loading, JOINs e DataLoader | Garçom trazendo 1 copo por viagem |
| **08** | Índices em Banco | Banco de Dados | B-Tree, Seletividade e Custo de Escrita | Índice remissivo de enciclopédia |
| **09** | Transações ACID | Banco de Dados | Integridade relacional e WAL | Teletransporte e regras de xadrez |
| **10** | SQL Injection | Banco de Dados | *Prepared Statements* e Binding de Dados | Carteiro com ordem explícita vs. envelope lacrado |
| **11** | Cookie vs. JWT | Segurança | Stateful vs. Stateless e Revogação | Ficha de guarda-volumes vs. passaporte |
| **12** | OAuth 2.0 | Segurança | Autorização delegada e fluxo PKCE | Chave de manobrista (*Valet Key*) |
| **13** | bcrypt vs. Argon2 | Segurança | KDFs *Memory-Hard* e resistência a GPUs | Fechadura pesada levantando chumbo |
| **14** | CORS | Segurança | *Same-Origin Policy* e Preflight `OPTIONS` | Fiscal de alfândega consultando ministério |
| **15** | Rate Limiting | Segurança | Controle de vazão e *Token Bucket* | Catraca de balada com cota de fichas |
| **16** | Cache com Redis | Arquitetura | *Cache-Aside* e mitigação de *Stampede* | Documentos na mesa vs. arquivo no subsolo |
| **17** | Filas Assíncronas | Arquitetura | Desacoplamento e absorção de rajadas | Balcão de padaria travado vs. fast-food |
| **18** | REST, GraphQL ou gRPC | Arquitetura | JSON vs. Schema SDL vs. Protobuf binário | Combo fechado vs. buffet a quilo vs. telégrafo |
| **19** | Idempotência em Webhook | Arquitetura | Deduplicação e entrega *At-Least-Once* | Porteiro conferindo código de rastreamento |
| **20** | Logs, Métricas e Traces | Arquitetura | OpenTelemetry e os 3 pilares | Monitor da UTI, prontuário e contraste vascular |

---

## 🛠️ Tecnologias e Padrões Mencionados

* **Protocolos e Padrões:** HTTP/1.1, HTTP/2, RFC 9110 (HTTP Semantics), RFC 5789 (PATCH), RFC 6749 (OAuth 2.0), RFC 7636 (PKCE), RFC 8594 (Sunset Header), W3C Trace Context.
* **Persistência e Cache:** PostgreSQL, MySQL, Redis, Índices B-Tree, Write-Ahead Logging (WAL).
* **Segurança e Criptografia:** OWASP Top 10, Argon2id, bcrypt, HMAC-SHA256, SameSite Cookies, CORS Preflight.
* **Mensageria e Protocolos:** RabbitMQ, Apache Kafka, AWS SQS, Protocol Buffers (gRPC), GraphQL.
* **Observabilidade:** OpenTelemetry (OTel), Prometheus, Grafana Loki, Jaeger, Grafana Tempo.
