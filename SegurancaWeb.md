# Guia Essencial de Segurança Web, Autenticação e Resiliência

Este documento reúne respostas técnicas aprofundadas sobre os pilares mais críticos da segurança e arquitetura de aplicações modernas: gestão de identidade, delegação de acesso, criptografia de credenciais, políticas de proteção de navegadores e controle de fluxo de tráfego.

---

## 11. Sessão com Cookie ou JWT?

A decisão entre sessões baseadas em cookies com estado (*stateful*) ou tokens assinados como JWT (*stateless*) define a forma como a identidade do usuário é validada e impacta diretamente a escalabilidade, revogabilidade e vetores de ataque da aplicação.

### Visão Técnica Detalhada

#### **Sessão com Cookie (Stateful)**
* **Como funciona:** O servidor autentica o usuário, cria um registro de sessão em um armazenamento centralizado em memória (ex.: Redis) e devolve um identificador opaco e aleatório (*Session ID*) armazenado em um cookie HTTP.
* **Segurança:** O cookie deve ser configurado com as diretivas de proteção:
  * `HttpOnly`: Impede que scripts maliciosos (XSS) leiam o cookie via JavaScript (`document.cookie`).
  * `Secure`: Garante que o cookie só trafegue por conexões criptografadas HTTPS.
  * `SameSite=Strict` ou `Lax`: Mitiga drasticamente ataques de CSRF (*Cross-Site Request Forgery*).
* **Vantagens:**
  * **Revogação Instantânea:** Se o usuário fizer logout, for banido ou tiver o dispositivo furtado, basta deletar a chave no Redis e a sessão é invalidada imediatamente.
  * **Payload Mínimo:** Apenas um ID de 32 a 64 bytes trafega na rede.
* **Desvantagens:** Exige infraestrutura centralizada de alta disponibilidade (cluster Redis) para checar o estado a cada requisição.

#### **JWT - JSON Web Token (Stateless)**
* **Como funciona:** O servidor gera um token codificado em Base64 contendo um cabeçalho, um payload com reivindicações (*claims* como `user_id`, `roles`, `exp`) e uma assinatura digital criptográfica (HMAC-SHA256 ou RSA/ECDSA).
* **Como valida:** Qualquer serviço que possua a chave secreta ou chave pública valida a autenticidade da assinatura de forma independente, sem realizar consultas a bancos de dados ou Redis.
* **Vantagens:**
  * **Desacoplamento e Microsserviços:** Permite que 10 microsserviços diferentes validem a identidade do usuário localmente sem gargalos em um banco de sessões compartilhado.
* **Desvantagens:**
  * **Dificuldade de Revogação:** Uma vez assinado, o JWT é matematicamente válido até seu timestamp de expiração (`exp`). Revogá-lo antes exige criar uma lista de bloqueio (*denylist/blacklist*) no Redis, o que anula o benefício *stateless*.
  * **Tamanho do Payload:** Trafega centenas de bytes em cada requisição HTTP nos cabeçalhos (`Authorization: Bearer <token>`).
  * **Risco de Armazenamento:** Se for guardado no `localStorage` do navegador, fica 100% exposto a qualquer vulnerabilidade de XSS.

---

### A Recomendação da Engenharia Sênior (Padrão Híbrido)
Em aplicações corporativas modernas, utiliza-se o melhor dos dois mundos:
1. **Access Token (JWT):** Tempo de vida ultracurto (ex.: 10 a 15 minutos), trafegando no cabeçalho ou em memória, minimizando a janela de risco caso seja interceptado.
2. **Refresh Token:** Tempo de vida longo (ex.: 7 a 30 dias), opaco, armazenado em um cookie seguro `HttpOnly` com rotação automática (*Refresh Token Rotation*) e persistido no banco/Redis para controle de revogação.

### Tabela Comparativa

| Critério | Sessão com Cookie (Stateful) | JWT Puro (Stateless) | Padrão Híbrido Recomendado |
| :--- | :--- | :--- | :--- |
| **Armazenamento de Estado** | Servidor (Redis/DB) | Nenhum (apenas no cliente) | Híbrido (Refresh Token no server) |
| **Revogação Imediata** | ✅ Imediata e trivial | ❌ Complexa / Não nativa | ✅ Quase imediata (máx. 10-15 min) |
| **Escalabilidade em Microsserviços** | Depende de Redis central | ✅ Excelente (descentralizada) | ✅ Excelente |
| **Vulnerabilidade Predominante** | CSRF (protegido por `SameSite`) | XSS (se salvo em `localStorage`) | Protegido com `HttpOnly` |

💡 **Analogia:**
* **Sessão com Cookie:** A ficha de metal do guarda-volumes de um clube. Você recebe uma ficha física com o número "84". A cada pedido, você apresenta a ficha; o atendente vai até o armário do clube consultar suas coisas. Se você perder a ficha, o atendente cancela aquele número no livro de registros na mesma hora.
* **JWT:** Um passaporte oficial com carimbo, foto e selo holográfico infalsificável emitido pelo governo. Qualquer autoridade na fronteira de qualquer país consegue olhar o selo e atestar sua identidade sem precisar telefonar para o país emissor. Contudo, se alguém roubar seu passaporte carimbado, poderá usá-lo em qualquer posto de fiscalização até que ele atinja a data de validade impressa nele.

---

## 12. Como Funciona o OAuth 2.0?

O **OAuth 2.0** (RFC 6749) é um framework de **autorização delegada** que permite que uma aplicação de terceiros acesse recursos protegidos em nome de um usuário, sem que o usuário precise fornecer sua senha a essa aplicação.

> **Distinção Fundamental:** OAuth 2.0 **não é** um protocolo de autenticação (*"quem é você?"*), mas de autorização (*"o que você tem permissão para fazer?"*). Para autenticação e identidade, utiliza-se a camada construída sobre ele: o **OpenID Connect (OIDC)**, que introduz o `id_token`.

### Os 4 Papéis do OAuth 2.0
1. **Resource Owner (Usuário):** O dono dos dados (ex.: você).
2. **Client (Aplicação):** O sistema que deseja acessar seus dados (ex.: um app de calendário de terceiros).
3. **Authorization Server:** O provedor que autentica o usuário e emite tokens (ex.: Google Identity, Auth0, Keycloak).
4. **Resource Server (API):** O serviço que guarda os dados protegidos (ex.: Google Calendar API).

---

### O Fluxo Mais Seguro: *Authorization Code Grant com PKCE*
Utilizado para aplicações web modernas (SPAs), mobile e backends para evitar o vazamento do código de autorização:

```
[ Usuário ]          [ Aplicação / Cliente ]         [ Servidor de Autorização ]       [ API de Recursos ]
     |                          |                                   |                          |
     |---- 1. Clica em Login -->|                                   |                          |
     |                          |-- 2. Gera PKCE (verifier/challenge)|                         |
     |                          |-- 3. Redireciona p/ tela de login |                          |
     |                          |      com scopes e challenge ----->|                          |
     |<--- 4. Exibe consentimento ----------------------------------|                          |
     |---- 5. Aprova permissão ------------------------------------>|                          |
     |                          |<-- 6. Devolve Code de 1 uso ------|                          |
     |                          |                                   |                          |
     |                          |-- 7. Troca Code + Verifier pelo   |                          |
     |                          |      Access Token --------------->|                          |
     |                          |<-- 8. Retorna Access Token -------|                          |
     |                          |                                                              |
     |                          |-- 9. Requisição com 'Authorization: Bearer <token>' -------->|
     |                          |<-- 10. Devolve os dados solicitados -------------------------|
```

### O que são Scopes (Escopos)?
Os escopos delimitam o raio de acesso do token emitido. Uma aplicação de fotos pode solicitar apenas `photos.readonly`. Mesmo que o invasor capture o token, ele não poderá apagar suas fotos nem ler seus e-mails.

💡 **Analogia da Chave de Manobrista (*Valet Key*):**
Em muitos veículos modernos, existe uma "chave de manobrista" separada da chave mestra do carro. Essa chave especial liga o motor e permite que o manobrista estacione o carro, mas **não destranca o porta-malas nem o porta-luvas**. Você nunca entrega a chave mestra com o controle do cofre ao manobrista; você entrega uma chave de autorização restrita apenas para a finalidade de estacionar.

---

## 13. Como Guardar Senha (bcrypt ou Argon2)?

### A Regra Fundamental de Segurança
**NUNCA** armazene senhas em texto plano e **NUNCA** utilize algoritmos de hash criptográficos rápidos genéricos como MD5, SHA-1, SHA-256 ou SHA-512.

* **O perigo das funções SHA-256 para senhas:** Funções da família SHA foram projetadas para integridade de arquivos e assinaturas digitais, sendo calculadas em nanossegundos. Uma GPU moderna comum (como uma RTX 4090) consegue calcular **dezenas de bilhões de hashes SHA-256 por segundo**, permitindo que ataques de força bruta (*brute force*) e *rainbow tables* quebrem senhas vazadas em poucas horas.

---

### O que usar: Funções de Derivação de Chaves Adaptativas (KDF)
Para senhas, usamos algoritmos que possuem dois pilares:
1. **Salting Automático Obrigatório:** Um valor aleatório gerado para cada usuário adicionado à senha antes do hash, garantindo que duas senhas idênticas ("123456") gerem strings de hash completamente diferentes, inviabilizando ataques pré-computados (*Rainbow Tables*).
2. **Custo de Processamento Configurável (*Work Factor*):** Introduz uma lentidão proposital ajustável (ex.: 200 a 500 ms por verificação) para tornar ataques em escala economicamente inviáveis.

---

### Comparativo: bcrypt vs. Argon2

#### **1. bcrypt (Criado em 1999)**
* **Mecanismo:** Baseado no algoritmo de cifra simétrica Blowfish.
* **Característica:** É um algoritmo puramente focado em **CPU (*CPU-bound*)**.
* **Fator de Custo:** Configura-se o número de iterações ($2^{\text{cost}}$). O padrão atual da indústria é um custo $\ge 12$.
* **Limitação Histórica:** Possui um limite intrínseco de entrada de **72 bytes**. Qualquer caractere após o 72º byte é ignorado (pode ser contornado gerando previamente um pré-hash SHA-256 da senha antes de passá-la ao bcrypt).
* **Veredito:** Maduro, estável e amplamente suportado em qualquer linguagem de programação.

#### **2. Argon2 (Vencedor da *Password Hashing Competition* - PHC 2015)**
* **Mecanismo:** É o padrão contemporâneo ouro recomendado pelo **OWASP**.
* **Inovação (*Memory-Hardness*):** Além de consumir tempo de CPU, o Argon2 exige **alocação deliberada de grandes blocos de memória RAM** (ex.: 64 MB por tentativa de hash) e paralelismo de threads.
* **Por que é superior contra invasores modernos:** Placas de vídeo (GPUs) e chips especializados (ASICs) possuem milhares de núcleos de processamento, mas pouquíssima memória dedicada por núcleo. Ao exigir 64 MB de RAM por hash, o invasor é incapaz de rodar milhões de tentativas em paralelo, neutralizando a vantagem de farms de GPUs.
* **Variantes:**
  * `Argon2d`: Focado em resistência a GPUs (usado em criptomoedas).
  * `Argon2i`: Otimizado contra ataques de canal lateral (*side-channel timing attacks*).
  * **`Argon2id` (Recomendado):** Versão híbrida que combina as proteções de ambos.

### Decisão de Engenharia Sênior
* **Novos Projetos:** Adote **Argon2id** (com parâmetros recomendados pelo OWASP: Memória: 64 MB, Iterações: 3, Paralelismo: 4).
* **Projetos Legados rodando bcrypt:** Mantenha o bcrypt com fator de custo 12 a 14; ele continua perfeitamente seguro e resistente para tráfego web de produção.

💡 **Analogia da Fechadura Pesada com Carga Física:**
* **SHA-256:** Uma fechadura tão leve que um robô consegue girar a chave 50 bilhões de vezes por segundo até acertar o segredo.
* **bcrypt:** Uma fechadura enferrujada propositalmente que demora 0,5 segundo para dar uma volta completa.
* **Argon2id:** Uma fechadura pesada que demora 0,5 segundo e que, para cada tentativa de giro, obriga quem está tentando a segurar uma barra de halteres de 50 kg nos braços (memória RAM). Um morador legítimo consegue levantar o peso uma vez para entrar em casa (fazer login); mas uma quadrilha de ladrões tentando testar 1 bilhão de combinações simultâneas não tem como colocar 1 bilhão de pessoas segurando barras pesadas sem que a estrutura inteira desmorone.

---

## 14. O que é CORS (Cross-Origin Resource Sharing)?

**CORS** é um mecanismo de segurança implementado exclusivamente pelos **navegadores web** para flexibilizar a rígida política de mesma origem (**Same-Origin Policy - SOP**).

### O que é uma "Origem"?
Uma origem é a combinação estrita de 3 elementos: **Esquema (Protocolo) + Domínio + Porta**.

| URL Requisitada de `https://app.com` | Mesma Origem? | Motivo |
| :--- | :---: | :--- |
| `https://app.com/api/usuarios` | ✅ Sim | Mesmo protocolo, domínio e porta (443) |
| `http://app.com/api/usuarios` | ❌ Não | Protocolo diferente (`http` vs `https`) |
| `https://api.app.com/usuarios` | ❌ Não | Subdomínio diferente (`api.app.com` vs `app.com`) |
| `https://app.com:8080/usuarios` | ❌ Não | Porta diferente (`8080` vs `443`) |

Se a origem for diferente, o navegador **bloqueia por padrão o acesso do script do frontend à resposta do servidor**, a menos que o servidor explicitamente conceda permissão via cabeçalhos CORS.

---

### O Mecanismo de Preflight (`OPTIONS`)
Quando uma requisição não é considerada "simples" (por exemplo, utiliza métodos como `PUT`, `DELETE`, `PATCH`, ou envia cabeçalhos como `Authorization` ou `Content-Type: application/json`), o navegador primeiro dispara uma requisição preliminar invisível usando o método **HTTP `OPTIONS`** (*Preflight*):

```
Frontend (origem: https://meuapp.com)            Backend (origem: https://api.empresa.com)
            |                                                     |
            |--- 1. OPTIONS /pedidos/123 ------------------------>|
            |    Origin: https://meuapp.com                       |
            |    Access-Control-Request-Method: DELETE            |
            |    Access-Control-Request-Headers: Authorization    |
            |                                                     |
            |<-- 2. Resposta do Preflight (204 No Content) -------|
            |    Access-Control-Allow-Origin: https://meuapp.com  |
            |    Access-Control-Allow-Methods: GET, POST, DELETE  |
            |    Access-Control-Allow-Headers: Authorization      |
            |    Access-Control-Max-Age: 86400 (Cache de 24h)     |
            |                                                     |
            |--- 3. DELETE /pedidos/123 (Requisição Real) ------->|
            |<-- 4. 200 OK com os dados --------------------------|
```

Se o servidor responder sem o cabeçalho `Access-Control-Allow-Origin` apropriado, o navegador emite o famoso erro no console:
> *`Access to fetch at ... from origin ... has been blocked by CORS policy`*

### O Mito Mais Comum de Segurança
> ❌ *"CORS serve para proteger minha API contra ataques de hackers."*

**FALSO.** Ferramentas como cURL, Postman, scripts em Python ou bots automatizados **ignoram completamente o CORS**, pois não são executados dentro de navegadores web. 

O CORS protege o **usuário final**: impede que um site malicioso (`site-falso.com`) que você abriu em uma aba do navegador execute chamadas em segundo plano para o site do seu banco (`banco.com`) aproveitando os seus cookies de autenticação salvos e roube suas informações financeiras via JavaScript.

💡 **Analogia da Alfândega do Aeroporto:**
O navegador é o fiscal alfandegário do aeroporto. Um passageiro estrangeiro (o script de outro domínio) tenta desembarcar com uma encomenda no país. O fiscal para o passageiro e envia uma mensagem de rádio (*OPTIONS preflight*) ao Ministério da Fazenda do país de destino: *"Esse cidadão do domínio X tem autorização diplomática para desembarcar com essa carga?"*. Se o ministério devolver a carta de autorização (*Access-Control-Allow-Origin*), o fiscal libera a passagem. Se o ministério negar ou não responder, o fiscal bloqueia a carga na alfândega ali mesmo.

---

## 15. Como Limitar Requisições (Rate Limiting)?

**Rate Limiting** (limitação de taxa) é a estratégia de controle de tráfego que restringe o número de requisições que um cliente pode enviar a uma API dentro de uma janela temporal pré-estabelecida.

### Por que é indispensável em produção?
1. **Proteção contra Ataques de Negação de Serviço (DoS/DDoS):** Evita a saturação de CPU, conexões de banco e pools de threads.
2. **Combate a Força Bruta (*Brute Force*):** Bloqueia tentativas automatizadas de adivinhar senhas em rotas como `/login` ou `/recuperar-senha`.
3. **Controle de Custos e Infraestrutura:** Protege rotas caras que invocam APIs externas pagas (ex.: chamadas a modelos LLM, envio de SMS, gateways de pagamento).
4. **Combate a *Web Scraping* Abusivo:** Impede que concorrentes clonem bases de dados inteiras via extração massiva.

---

### Principais Algoritmos de Rate Limiting

#### 1. Token Bucket (Balde de Fichas) — Mais Utilizado
* **Como funciona:** Um balde possui capacidade máxima de $B$ fichas e recebe novas fichas a uma taxa constante de $R$ fichas por segundo.
* **Operação:** Cada requisição retira 1 ficha. Se o balde tiver fichas, a requisição passa; se estiver vazio, a requisição é rejeitada.
* **Vantagem:** Suporta **rajadas controladas de tráfego (*bursts*)**, permitindo que o cliente gaste as fichas acumuladas rapidamente e depois volte à velocidade nominal.

#### 2. Leaky Bucket (Balde Furado)
* **Como funciona:** As requisições entram em uma fila (balde) e saem para processamento por um orifício inferior a uma vazão estritamente constante.
* **Vantagem:** Suaviza picos de tráfego (*traffic shaping*), garantindo que o backend receba um fluxo sempre estável e homogêneo. Se a fila encher, novas requisições transbordam e são descartadas.

#### 3. Fixed Window Counter (Janela Fixa)
* **Como funciona:** Divide o tempo em blocos rígidos (ex.: no máximo 100 requisições entre 14:00 e 14:01).
* **Problema:** Vulnerável a rajadas na fronteira das janelas. Se um cliente fizer 100 requisições às 14:00:59 e mais 100 requisições às 14:01:01, o sistema processará 200 requisições em um intervalo de apenas 2 segundos.

#### 4. Sliding Window Counter (Janela Deslizante)
* **Como funciona:** Calcula uma média ponderada entre as requisições da janela anterior e da janela atual com base no minuto corrente, eliminando o problema de pico nas bordas da janela fixa com consumo mínimo de memória.

---

### Padrão de Resposta HTTP (RFC 6585 e Standards)
Quando o cliente excede a cota permitida, a API deve responder imediatamente com o status **`429 Too Many Requests`** e enviar cabeçalhos informativos:

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
Retry-After: 30
RateLimit-Limit: 100
RateLimit-Remaining: 0
RateLimit-Reset: 1711200030

{
  "error": "Too Many Requests",
  "message": "Você excedeu o limite de requisições. Tente novamente em 30 segundos."
}
```

### Onde Implementar na Arquitetura?
Em arquiteturas de alta escala, o rate limiting **não deve ser executado pelo código da aplicação** (Node.js, Java, Python), pois processar a conexão já consome recursos valiosos. Deve ser delegado à camada de borda:
* **API Gateways / Proxies Reversos:** Kong, Nginx, Envoy, Traefik.
* **Camada de Nuvem / Edge:** Cloudflare, AWS WAF, Google Cloud Armor.
* **Armazenamento de Contadores Distribuídos:** **Redis**, utilizando comandos atômicos incrementais com expiração (`INCR` e `EXPIRE`) ou scripts em Lua para operações de janela deslizante sem *race conditions*.

💡 **Analogia da Catraca da Balada:**
Imagine uma casa noturna exclusiva que possui uma catraca de controle de fluxo na entrada. Cada frequentador recebe uma cota de 10 fichas por hora para pedir drinks no bar. A cada drink pedido, o barman retira uma ficha. Quando suas fichas daquela hora acabam, o segurança educadamente impede você de pedir mais e aponta para o relógio na parede: *"Aguarde 15 minutos até a próxima rodada horária de fichas ser liberada"* (Status 429 com `Retry-After: 900`).
