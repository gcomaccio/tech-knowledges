# Guia Essencial de Arquitetura de APIs REST

Este documento reúne respostas técnicas e aprofundadas para as principais perguntas conceituais de engenharia de software sobre APIs HTTP/REST, combinando rigor técnico, referências às RFCs oficiais e analogias do dia a dia para fixação intuitiva.

---

## 1. GET, POST, PUT, PATCH e DELETE: quando usar cada um?

Os métodos HTTP (verbos) definem a semântica da ação que o cliente deseja executar sobre um recurso identificado por uma URI.

### Métodos em Detalhes

#### **GET**
* **Quando usar:** Exclusivamente para recuperação e leitura de informações.
* **Características:** É um método **seguro** (*safe*) e **idempotente**. Não deve produzir efeitos colaterais perceptíveis no estado do servidor (logs e métricas não contam como alteração de estado do recurso). O corpo da requisição (*body*) normalmente não deve carregar semântica.
* **Exemplo:** `GET /pedidos/123`
* 💡 **Analogia:** Consultar o cardápio em um restaurante ou olhar as horas no relógio de rua. Olhar não gasta comida nem altera a hora do relógio; apenas traz informação.

#### **POST**
* **Quando usar:** Para criar um recurso subordinado a uma coleção ou disparar processos e ações no servidor que não se encaixam puramente no modelo CRUD (ex.: processar pagamento, autenticar usuário).
* **Características:** **Não é seguro** e **não é idempotente**. Cada envio pode criar um novo recurso com identificador próprio gerado pelo servidor.
* **Exemplo:** `POST /pedidos` (o servidor gera o pedido `123` e retorna status `201 Created` com o header `Location: /pedidos/123`).
* 💡 **Analogia:** Entregar uma ficha de matrícula preenchida na secretaria de uma faculdade. Cada vez que você entrega uma nova ficha, uma nova inscrição diferente com novo número de protocolo é gerada.

#### **PUT**
* **Quando usar:** Para substituição integral de um recurso existente ou criação quando a URI completa já é conhecida e definida pelo cliente.
* **Características:** **Não é seguro**, mas é **idempotente**. O payload enviado representa o novo estado completo do recurso; campos omitidos no payload devem ser redefinidos para os valores padrão ou removidos.
* **Exemplo:** `PUT /usuarios/42` com `{ "nome": "Ana", "email": "ana@email.com" }`. Se o usuário tinha telefone cadastrado e o campo foi omitido, o telefone deixa de existir.
* 💡 **Analogia:** Substituir toda a mobília de uma sala por um jogo novo de móveis. Se você encomendou apenas sofá e TV, a mesa antiga foi retirada e a sala fica sem mesa.

#### **PATCH**
* **Quando usar:** Para atualização parcial de um recurso.
* **Características:** Modifica apenas os campos explicitamente informados no payload, preservando todo o restante do estado do recurso (conforme RFC 5789).
* **Exemplo:** `PATCH /usuarios/42` com `{ "email": "novo@email.com" }`. Nome, telefone e endereço permanecem intactos.
* 💡 **Analogia:** Chamar um pintor para retocar apenas uma parede arranhada do quarto, sem tirar nem trocar nenhum móvel do lugar.

#### **DELETE**
* **Quando usar:** Para solicitar a remoção de um recurso específico.
* **Características:** **Não é seguro**, mas é **idempotente**. Após a execução com sucesso, o recurso não deve mais existir no servidor.
* **Exemplo:** `DELETE /pedidos/123`
* 💡 **Analogia:** Jogar um documento no triturador de papel. Depois de picotado, tentar passá-lo pelo triturador de novo não altera o fato de que o documento já foi destruído.

### Matriz Comparativa

| Método | Finalidade Semântica | Seguro (*Safe*) | Idempotente | Body na Requisição | Status Comum de Sucesso |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **GET** | Leitura de recurso | Sim | Sim | Não recomendado | `200 OK` |
| **POST** | Criação / Processamento | Não | Não | Sim | `201 Created` / `200 OK` |
| **PUT** | Substituição integral | Não | Sim | Sim | `200 OK` / `204 No Content` |
| **PATCH** | Atualização parcial | Não | Condicional* | Sim | `200 OK` / `204 No Content` |
| **DELETE** | Exclusão de recurso | Não | Sim | Raramente | `204 No Content` / `200 OK` |

*\*Pela RFC 5789, PATCH não é intrinsecamente idempotente, embora mutações diretas de campos normalmente se comportem como tal.*

---

## 2. O que é uma API Stateless?

### Conceito
Uma API é considerada **Stateless** (sem estado de sessão no servidor) quando **cada requisição HTTP enviada pelo cliente contém, em si mesma, todas as informações necessárias para que o servidor possa compreendê-la e processá-la**.

O servidor **não armazena contexto de sessão daquele cliente em sua memória local** entre uma requisição e outra.

### Como funciona na prática?
* Em arquiteturas tradicionais com estado (*stateful*), o servidor guarda um `session_id` em memória (como sessões PHP ou Tomcat) e o cliente só envia um cookie de identificação.
* Em APIs REST *Stateless*, a identidade, permissões e dados de contexto transitam em cada requisição através de cabeçalhos padronizados (ex.: `Authorization: Bearer <JWT>`, headers de paginação, etc.).

### Por que isso é vital em engenharia de software?
1. **Escalabilidade Horizontal Infinita:** Se o cluster tem 50 instâncias da API atrás de um Load Balancer, a requisição 1 pode cair no nó A e a requisição 2 no nó B sem necessidade de replicação de memória ou *sticky sessions*.
2. **Resiliência a Falhas:** Se o servidor que atendeu a requisição anterior reiniciar, a próxima requisição do cliente não sofrerá perda de sessão.
3. **Cache e Desacoplamento:** Requisições autocontidas facilitam a aplicação de proxies de cache (como CDN e Redis).

💡 **Analogia:** Uma máquina automática de café expresso ou um pedágio sem cancela. A máquina não precisa se lembrar quem você é; você se aproxima, aproxima o cartão de crédito com o pagamento (token) e aperta o botão do café desejado. Cada café é uma transação independente e autocontida.

---

## 3. 401 Unauthorized ou 403 Forbidden?

Ambos são códigos de erro do cliente (`4xx`), mas tratam de etapas completamente distintas do ciclo de segurança: **Autenticação** versus **Autorização**.

```
                   +---------------------------+
                   |  Requisição do Cliente    |
                   +-------------+-------------+
                                 |
                     [ O cliente se identificou? ]
                                 |
                   +-------------+-------------+
                  NÃO                         SIM
                   |                           |
         +---------v---------+       [ Tem permissão para o recurso? ]
         |  401 UNAUTHORIZED |                 |
         +-------------------+       +---------+---------+
                                    NÃO                 SIM
                                     |                   |
                           +---------v-------+   +-------v-------+
                           |  403 FORBIDDEN  |   |    200 OK     |
                           +-----------------+   +---------------+
```

### 401 Unauthorized (Não Autenticado)
* **Significado real:** *"Quem é você?"*
* **Quando ocorre:** O cliente não enviou credenciais, enviou um token inválido, expirado ou com assinatura corrompida.
* **Exigência da RFC 9110:** A resposta HTTP **deve** incluir o cabeçalho `WWW-Authenticate` indicando como o cliente pode se autenticar (ex.: `WWW-Authenticate: Bearer realm="api"`).
* **Ação corretiva:** O cliente precisa fazer login ou renovar seu token (*refresh token*).

### 403 Forbidden (Não Autorizado / Proibido)
* **Significado real:** *"Eu sei quem você é, mas você não tem direito de acessar este recurso."*
* **Quando ocorre:** As credenciais foram validadas com sucesso (a identidade do usuário é conhecida), porém as regras de negócio, permissões ou papéis (RBAC - *Role-Based Access Control*) proíbem o acesso.
* **Ação corretiva:** Reenviar as mesmas credenciais não resolverá; o usuário precisará solicitar elevação de privilégios a um administrador.

💡 **Analogia da Portaria do Prédio Corporativo:**
* **401 Unauthorized:** Você tenta entrar no prédio corporativo sem crachá ou com o crachá do seu emprego anterior. O recepcionista barra na catraca: *"Você precisa se cadastrar na recepção primeiro."*
* **403 Forbidden:** Você passou pela catraca com seu crachá válido de analista júnior, mas tenta passar o cartão na porta blindada da sala da diretoria financeira. A luz fica vermelha e apita: *"Sei quem você é, mas seu crachá não tem permissão para abrir esta sala."*

---

## 4. O que é Idempotência? Qual método não é idempotente?

### Definição
Originada da matemática ($f(f(x)) = f(x)$), no contexto de APIs HTTP, **uma operação é idempotente quando executá-la uma única vez ou executá-la múltiplas vezes consecutivas produz exatamente o mesmo efeito colateral no estado do servidor**.

> **Atenção:** Idempotência refere-se ao **estado final dos dados no servidor**, e não necessariamente ao status code retornado.

### Classificação dos Métodos

| Idempotente? | Métodos | Comportamento |
| :--- | :--- | :--- |
| **Sim** | `GET`, `HEAD`, `OPTIONS`, `TRACE` | São seguros e nunca alteram dados. |
| **Sim** | `PUT` | Substitui o recurso. Se você enviar o mesmo payload 10 vezes, o recurso terá exatamente o mesmo estado após a 1ª e após a 10ª vez. |
| **Sim** | `DELETE` | Na 1ª requisição remove o registro (`204 No Content`). Nas seguintes pode retornar `404 Not Found`, mas o estado final do servidor não muda: o recurso continua não existindo. |
| **NÃO** | **POST** | **É o método não idempotente por excelência.** Enviar a mesma requisição 3 vezes criará 3 registros ou disparará 3 cobranças diferentes. |
| **Condicional** | `PATCH` | Se alterar campos absolutos (`{ "status": "PAGO" }`), comporta-se como idempotente. Mas se usar instruções incrementais (`{ "saldo": "+10" }`), cada chamada acumulará efeitos. |

### Como lidar com a não idempotência do POST em sistemas críticos?
Em operações financeiras ou de checkout, utiliza-se o padrão de **Chave de Idempotência (*Idempotency-Key*)**:
O cliente gera um UUID único no cabeçalho:
```http
POST /v1/cobrancas
Idempotency-Key: 7b29a8f4-6e01-4471-a477-9d7a26f8d05e
```
Se a conexão oscilar e o cliente reexecutar a chamada, o servidor detecta a chave já processada no Redis/Banco e devolve a resposta cacheada da primeira chamada sem duplicar a cobrança.

💡 **Analogia:**
* **Idempotente:** O botão do elevador para o 10º andar. Se você apertar 1 vez ou apertar 30 vezes impacientemente, o efeito no elevador é o mesmo: ele irá para o 10º andar.
* **Não idempotente:** O botão de dispensar refrigerante de uma máquina de vendas ou a roleta de um cassino. Cada vez que você aciona, uma nova moeda é consumida e uma nova lata é liberada.

---

## 5. Como versionar uma API?

O versionamento é a salvaguarda de contratos de comunicação, permitindo que uma API evolua sem quebrar integrações legadas (*Breaking Changes*).

### Quando criar uma nova versão?
Apenas quando houver **quebra de contrato**, por exemplo:
* Remoção ou renomeação de endpoints ou campos de resposta.
* Mudança no tipo de dado de um campo (ex.: de `number` para `object`).
* Mudança na semântica de uma regra de negócio existente.

> Se for apenas adicionar um campo opcional novo na resposta, **não** crie uma nova versão; pratique a tolerância a aditivos (*Additive Changes*).

---

### Principais Estratégias de Versionamento

#### 1. Versionamento por URI (*Path Versioning*) — Mais popular
A versão é prefixada no próprio caminho do endpoint.
* **Exemplo:** `https://api.exemplo.com/v1/usuarios` e `https://api.exemplo.com/v2/usuarios`
* **Vantagens:** Fácil de entender, fácil de testar direto no navegador e simplifica o roteamento em API Gateways (como Kong, Apigee, AWS API Gateway).
* **Desvantagens:** Quebra o princípio estrito do REST de que uma URI deve representar um recurso perene, não a sua representação temporal.

#### 2. Versionamento por Cabeçalho Customizado (*Custom Headers*)
A versão é enviada em um header HTTP proprietário.
* **Exemplo:**
  ```http
  GET /usuarios/42 HTTP/1.1
  Host: api.exemplo.com
  X-API-Version: 2
  ```
  *(Utilizado amplamente por plataformas como a Stripe, com base em datas: `Stripe-Version: 2024-04-10`)*
* **Vantagens:** Mantém as URIs limpas e canônicas.
* **Desvantagens:** Requer clientes configurados com cabeçalhos adequados e dificulta testes rápidos via browser.

#### 3. Versionamento por Negociação de Conteúdo (*Accept Header / MIME Types*)
A versão é solicitada no cabeçalho padrão de tipo de mídia aceito.
* **Exemplo:**
  ```http
  GET /usuarios/42 HTTP/1.1
  Accept: application/vnd.minhaempresa.v2+json
  ```
* **Vantagens:** É a abordagem mais aderente à especificação conceitual do REST e HATEOAS.
* **Desvantagens:** Mais complexa para implementar, documentar (Swagger/OpenAPI) e depurar.

#### 4. Versionamento por Query Parameters
A versão é informada como parâmetro de consulta.
* **Exemplo:** `https://api.exemplo.com/usuarios?version=2`
* **Vantagens:** Simples de executar testes pontuais.
* **Desvantagens:** Interfere negativamente em camadas de cache e é considerada um anti-padrão pela maioria dos times de plataforma.

---

### Boas Práticas de Engenharia (Visão Sênior)
1. **Política de Ciclo de Vida (*Deprecation & Sunset*):** Utilize o cabeçalho padronizado da RFC 8594 `Sunset` informando aos clientes a data limite em que uma versão antiga será desligada:
   ```http
   Sunset: Wed, 11 Nov 2026 00:00:00 GMT
   Deprecation: true
   ```
2. **Suporte Limitado de Versões:** Mantenha no máximo 2 versões ativas simultaneamente (ex.: `v(atual)` e `v(anterior)`). Manter muitas versões vivas fragmenta o código e onera os testes de regressão.

💡 **Analogia:**
* **URI Versioning:** Ter duas agências bancárias físicas na mesma rua: a "Agência Tradicional (v1)" e a "Agência Digital (v2)". O cliente decide fisicamente em qual porta entrar.
* **Header Versioning:** Ir a um restaurante tradicional, sentar na mesma mesa, mas informar ao garçom no bilhete: *"Quero o prato do cardápio especial de 2026"*. O ambiente e a mesa são os mesmos, mas a cozinha consulta a nota do pedido para saber qual receita preparar.
