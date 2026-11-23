# Guia Essencial de Engenharia de Bancos de Dados e SQL

Este documento reúne respostas técnicas aprofundadas para questões cruciais de bancos de dados relacionais, modelagem, otimização e persistência de dados. O conteúdo equilibra rigor de engenharia, mecanismos internos de SGBDs e analogias intuitivas para cada tópico.

---

## 6. Paginação por Offset ou por Cursor?

A estratégia de paginação define como a aplicação divide e entrega grandes volumes de dados em fatias gerenciáveis. A escolha entre **Offset** e **Cursor (Keyset)** impacta diretamente a escalabilidade do banco e a consistência das leituras.

### Comparativo Técnico

#### **Paginação por Offset (`LIMIT / OFFSET`)**
* **Como funciona:** O cliente informa o número da página ou a quantidade de itens a saltar.
  ```sql
  -- Busca a página 50.000 (tamanho 20)
  SELECT * FROM pedidos ORDER BY id ASC LIMIT 20 OFFSET 1000000;
  ```
* **O gargalo oculto de performance:** Para entregar 20 linhas pulando 1.000.000, o motor do banco de dados (PostgreSQL, MySQL, etc.) precisa escanear em memória e descartar 1.000.000 de registros antes de selecionar os 20 desejados. O custo de I/O e CPU é linear: **$O(N)$**.
* **Problema da Inconsistência (*Data Drift*):** Se novos registros forem inseridos ou deletados enquanto o usuário navega, registros podem aparecer duplicados ou ser sumariamente omitidos entre as páginas.
* **Quando usar:** UIs administrativas simples com pequenas tabelas (< 10.000 linhas) onde o usuário precisa saltar aleatoriamente para páginas específicas (ex.: "Ir para a página 42").

#### **Paginação por Cursor / Keyset**
* **Como funciona:** O cliente não envia o número da página; ele envia o ponteiro (*cursor*) do último item que visualizou na consulta anterior (geralmente uma chave sequencial indexada, como `id` ou `created_at, id`).
  ```sql
  -- Busca os próximos 20 itens a partir do último visto
  SELECT * FROM pedidos 
  WHERE id > 1000000 
  ORDER BY id ASC 
  LIMIT 20;
  ```
* **Por que é superior em escala:** O banco utiliza diretamente o índice B-Tree para saltar até a chave informada em tempo logarítmico **$O(\log N)$** e percorre apenas as 20 linhas seguintes em tempo constante **$O(1)$**, consumindo o mesmo tempo na página 1 ou na página 10.000.
* **Imunidade a anomalias:** Itens novos inseridos no início ou no fim da tabela não deslocam o cursor do leitor.
* **Quando usar:** Feeds infinitos (*infinite scroll*), APIs de alto tráfego, listagens volumosas e integrações de mensageria/eventos.

### Matriz Comparativa

| Critério | Offset Pagination | Cursor / Keyset Pagination |
| :--- | :--- | :--- |
| **Complexidade em páginas profundas** | $O(N)$ — Degrada rapidamente | $O(1)$ — Constante e escalável |
| **Pulo direto para páginas arbitrárias** | ✅ Sim (ex.: "Página 15") | ❌ Não (apenas próxima / anterior) |
| **Estabilidade contra inserções concorrentes** | ❌ Vulnerável a duplicações/saltos | ✅ Totalmente consistente |
| **Complexidade da consulta SQL** | Muito simples | Requer ordenação determinística e índice |

💡 **Analogia:**
* **Offset:** Um leitor que, para ler a página 500 de um livro, é obrigado a começar da página 1 e folhear 499 páginas uma a uma até chegar nela. Se alguém arrancar uma folha do início do livro, todas as contagens futuras de páginas ficam erradas.
* **Cursor:** Usar um marcador de páginas (fitilho). Você abre o livro exatamente no ponto onde parou com um único movimento imediato, sem importar se já leu 10 páginas ou 10.000 páginas.

---

## 7. O que é o Problema N+1?

O **Problema N+1** é um dos gargalos de desempenho mais frequentes em aplicações que utilizam mapeadores objeto-relacional (ORMs como Hibernate, Prisma, Entity Framework, TypeORM, SQLAlchemy) e APIs GraphQL.

### Como ele acontece na prática?
Ocorre quando o código executa **1 consulta inicial** para buscar uma coleção de registros pai e, subsequentemente, dispara **$N$ consultas adicionais individuais** para carregar os relacionamentos filhos de cada um desses registros.

```
+-------------------------------------------------------+
|  1. Consulta Pai: SELECT * FROM usuarios LIMIT 10;     |
+-------------------------------------------------------+
  |
  +--> 2. Para cada usuário, dispara uma query filha (N vezes):
         SELECT * FROM pedidos WHERE usuario_id = 1;
         SELECT * FROM pedidos WHERE usuario_id = 2;
         ...
         SELECT * FROM pedidos WHERE usuario_id = 10;
  |
  Total de chamadas de rede ao banco: 1 + 10 = 11 queries!
```

Se a listagem inicial retornar 500 usuários, a aplicação fará **501 requisições** ao banco de dados para montar uma única tela.

### Sintomas no Sistema
* Latência de rede multiplicada (*Network Round-trip Overhead*).
* Esgotamento prematuro do pool de conexões do banco de dados (*Connection Pool Exhaustion*).
* Picos de utilização de CPU e I/O tanto no servidor da aplicação quanto no SGBD.

### Estratégias de Resolução

1. **Eager Loading com JOIN (Carregamento Antecipado):**
   Instruir o ORM a buscar os dados relacionados em uma única consulta com `JOIN`:
   ```sql
   SELECT u.*, p.* 
   FROM usuarios u 
   LEFT JOIN pedidos p ON p.usuario_id = u.id;
   ```
2. **Batch Fetching / Cláusula IN:**
   O ORM busca os pais em uma query e todos os filhos correspondentes em uma segunda query em lote:
   ```sql
   SELECT * FROM usuarios LIMIT 10;
   SELECT * FROM pedidos WHERE usuario_id IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
   ```
   *(Reduz de $N + 1$ para apenas $2$ queries).*
3. **Padrão DataLoader (Essencial em GraphQL):**
   Agrupa e remove duplicidades de IDs durante o ciclo da requisição (*batching + caching*), postergando a resolução dos campos até consolidar as consultas.

💡 **Analogia:**
* **Problema N+1:** Um garçom atende uma mesa com 10 clientes. Ele anota os pratos de todos em uma única passada (1 ida). Na hora de servir as bebidas, em vez de colocar os 10 copos em uma bandeja e levar de uma vez, ele vai até a cozinha, pega o copo da 1ª pessoa, volta e entrega. Depois vai até a cozinha pegar o da 2ª pessoa, volta e entrega... repetindo essa viagem 10 vezes seguidas ($1 + 10$ viagens).
* **Solução:** O garçom utiliza uma bandeja e traz os pedidos de todos os 10 clientes de uma só vez (1 viagem com JOIN / Batch).

---

## 8. Quando um Índice Ajuda e Quando Atrapalha?

Um índice de banco de dados é uma estrutura de dados auxiliar (predominantemente **B-Tree**, **Hash**, ou **GIN/GiST**) mantida separadamente das páginas de dados brutos da tabela para acelerar a localização física dos registros.

```
           Árvore B-Tree (Índice)                  Tabela Principal (Heap)
                 [ 50 ]                                  [Linha 1: Ana, 25]
                /      \                                 [Linha 2: Beto, 50]
           [ 25 ]      [ 75 ]        == Aponta para ==>  [Linha 3: Carlos, 75]
           /    \      /    \                            [Linha 4: Diana, 12]
        [12]   [30]  [60]   [90]
```

### Quando o Índice AJUDA?
1. **Filtros com Alta Seletividade (`WHERE`):** Colunas que possuem grande diversidade de valores únicos (alta cardinalidade, como CPF, e-mail, UUIDs, chaves primárias). O índice reduz o universo de busca de milhões de linhas para pouquíssimas leituras de nós.
2. **Junções de Tabelas (`JOIN`):** Índices em chaves estrangeiras (*Foreign Keys*) transformam *Hash Joins* ou *Nested Loops* caríssimos em buscas indexadas ultra velozes.
3. **Ordenação e Agrupamento (`ORDER BY` e `GROUP BY`):** O índice B-Tree armazena as chaves previamente ordenadas. O banco entrega o resultado ordenado diretamente sem precisar alocar memória para *filesort* ou ordenação temporária em disco.
4. **Índice Cobertor (*Covering Index / Index-Only Scan*):** Quando todos os campos solicitados no `SELECT` já existem no próprio índice, o banco sequer acessa o arquivo principal da tabela (*Heap*).

---

### Quando o Índice ATRAPALHA?
1. **Sobrecarga Severa em Escritas (`INSERT`, `UPDATE`, `DELETE`):**
   * Toda vez que um registro é inserido ou removido, **todas as árvores de índices daquela tabela precisam ser reequilibradas e reescritas**.
   * Em tabelas transacionais de alto volume de gravação (logs, eventos de telemetria, streaming financeiro), ter muitos índices transforma operações de $1 \text{ ms}$ em dezenas de milissegundos por saturação de I/O de disco.
2. **Consumo Excessivo de Armazenamento e Memória RAM:**
   * Índices ocupam espaço em disco (às vezes superando o tamanho da própria tabela).
   * Para serem eficazes, os nós quentes dos índices devem caber na memória RAM (*Buffer Pool / Shared Buffers*). Índices gigantes expulsam dados úteis da RAM.
3. **Colunas com Baixa Cardinalidade / Seletividade:**
   * Criar índice em colunas como `status_ativo (true/false)` ou `sexo (M/F)` costuma ser inútil. O otimizador de consultas avaliará que 50% dos dados atendem ao filtro e preferirá fazer um *Full Table Scan* (leitura sequencial rápida), ignorando o índice que continuará consumindo recursos de escrita inutilmente.

💡 **Analogia da Enciclopédia:**
* **Ajuda:** O índice remissivo no final de uma enciclopédia médica de 2.000 páginas. Para achar o verbete "Hipertensão", você consulta o índice remissivo na letra H e vai direto à página 1.432 em 3 segundos, em vez de ler o livro inteiro folha por folha.
* **Atrapalha:** Imagine que para cada nova página escrita na enciclopédia, você seja obrigado a atualizar manualmente 20 índices remissivos diferentes: um por doença, outro por remédio, outro por cor do sintoma, outro por autor... O trabalho de escrever uma única página nova torna-se infinitamente burocrático, lento e custoso.

---

## 9. O que é uma Transação ACID?

Uma **transação** é uma unidade lógica de processamento de trabalho que engloba uma ou mais operações SQL. O acrônimo **ACID** representa os quatro requisitos arquiteturais indispensáveis para assegurar a confiabilidade e integridade dos dados em SGBDs relacionais.

### Os 4 Pilares do ACID

```
  A - Atomicidade     -->  Tudo ou nada (Rollback em falhas)
  C - Consistência    -->  Regras de negócio e constraints respeitadas
  I - Isolamento      -->  Transações concorrentes não se corrompem
  D - Durabilidade    -->  Dados persistidos sobrevivem a desligamentos
```

#### **1. Atomicidade (*Atomicity*) — "Tudo ou Nada"**
* A transação é tratada como um bloco indivisível.
* Se 9 operações forem concluídas com êxito e a 10ª falhar, **todas as alterações anteriores são revertidas (`ROLLBACK`)**. O banco nunca permanece em um estado semi-executado.

#### **2. Consistência (*Consistency*) — "Integridade Absoluta"**
* A transação só pode levar o banco de um estado válido a outro estado válido.
* Nenhuma operação tem permissão para violar restrições de integridade (*Primary Keys*, *Foreign Keys*, *Unique*, *Check constraints* ou *Triggers*). Se uma violação ocorrer, a transação inteira é abortada.

#### **3. Isolamento (*Isolation*) — "Operações Paralelas Seguras"**
* O resultado de uma transação em andamento não deve ser visível para outras transações antes de seu fechamento (`COMMIT`).
* O padrão SQL define 4 níveis de isolamento para mitigar anomalias de concorrência (*Dirty Reads*, *Non-repeatable Reads*, *Phantom Reads*):
  1. `READ UNCOMMITTED`
  2. `READ COMMITTED` (Padrão no PostgreSQL e Oracle)
  3. `REPEATABLE READ` (Padrão no MySQL/InnoDB)
  4. `SERIALIZABLE` (Isolamento mais rigoroso, execução equivalente a uma fila serial)

#### **4. Durabilidade (*Durability*) — "Permanência Garantida"**
* Uma vez que a transação recebe a confirmação de sucesso (`COMMIT`), suas modificações tornam-se definitivas no disco.
* Mesmo que o servidor sofra uma pane elétrica instantânea no milissegundo seguinte, o banco reconstrói o estado com precisão ao reiniciar utilizando o mecanismo de **WAL (*Write-Ahead Logging*)** ou *Redo Log*.

---

### Exemplo Prático: Transferência Bancária
Transferir R$ 100 da conta de **Alice** para a conta de **Bob**:

```sql
BEGIN TRANSACTION;
  -- Etapa 1: Débito
  UPDATE contas SET saldo = saldo - 100 WHERE titular = 'Alice';

  -- Etapa 2: Crédito
  UPDATE contas SET saldo = saldo + 100 WHERE titular = 'Bob';
COMMIT;
```

* **Sem Atomicidade:** Se o sistema cair entre a etapa 1 e a etapa 2, os R$ 100 desapareceriam do sistema (saíram de Alice e nunca chegaram a Bob).
* **Sem Isolamento:** Um terceiro consultando os saldos no meio do processo veria um saldo inconsistente do patrimônio total.
* **Sem Durabilidade:** O banco confirmaria o Pix e, ao cair a luz, o dinheiro sumiria do saldo de Bob.

💡 **Analogias:**
* **Atomicidade:** O teletransporte de ficção científica. Ou você e todas as suas moléculas chegam 100% inteiros no destino, ou a máquina cancela e você continua intacto no ponto de origem. Não existe teletransportar metade do corpo.
* **Consistência:** As regras de um tabuleiro de xadrez. Nenhuma jogada que coloque o próprio Rei em xeque é aceita pelas regras do jogo; ela é sumariamente invalidada.
* **Isolamento:** Cabines blindadas e privativas em uma agência bancária. O que você está sacando ou assinando dentro da sua cabine não interfere nem é visível para o cliente da cabine ao lado.
* **Durabilidade:** Esculpir um contrato em uma lápide de granito ou registrar em cartório oficial. Se faltar luz na cidade após a assinatura, a pedra gravada não se apaga.

---

## 10. Como Evitar SQL Injection?

**SQL Injection (SQLi)** é uma das vulnerabilidades de segurança mais perigosas da história da computação (frequente no Top 10 da OWASP). Ocorre quando **dados não confiáveis fornecidos pelo usuário são interpolados ou concatenados diretamente na sintaxe da consulta SQL**, alterando a árvore semântica do comando executado pelo interpretador do banco.

### Exemplo do Ataque

#### Código Vulnerável (Concatenação Direta de Strings):
```python
# PERIGO EXTREMO: Nunca faça isso!
query = f"SELECT * FROM usuarios WHERE email = '{email}' AND senha = '{senha}'"
db.execute(query)
```
Se o invasor digitar no campo `email`:
```sql
admin@empresa.com' OR '1'='1
```
A consulta resultante executada será:
```sql
SELECT * FROM usuarios WHERE email = 'admin@empresa.com' OR '1'='1' AND senha = '...'
```
Como `'1'='1'` é sempre verdadeiro, o invasor obtém login como administrador sem fornecer senha. Em cenários piores (`'; DROP TABLE usuarios; --`), o banco inteiro pode ser destruído.

---

### As 5 Camadas de Defesa de um Engenheiro Sênior

#### 1. Consultas Parametrizadas / *Prepared Statements* (A Regra de Ouro)
É o mecanismo definitivo e obrigatório. Ele **separa fisicamente o código SQL dos dados**:
1. O banco primeiro recebe a estrutura SQL fixa e a pré-compila criando o plano de execução.
2. Em seguida, os valores fornecidos pelo usuário são transmitidos separadamente apenas como **literais de dados**.
3. O interpretador jamais interpretará esses parâmetros como comandos, mesmo que contenham aspas, pontos-e-vírgulas ou instruções SQL.

```python
# SEGURO: Parâmetros vinculados (Parameter Binding)
query = "SELECT * FROM usuarios WHERE email = %s AND senha = %s"
db.execute(query, (email, senha_hash))
```

```java
// SEGURO em Java (PreparedStatement)
PreparedStatement stmt = conn.prepareStatement("SELECT * FROM usuarios WHERE email = ?");
stmt.setString(1, email);
ResultSet rs = stmt.executeQuery();
```

#### 2. Uso Cuidadoso de ORMs e Query Builders
ORMs modernos (Prisma, Hibernate, Entity Framework, SQLAlchemy) utilizam parametrização por padrão em suas APIs tipadas:
```typescript
// Seguro por design
await prisma.usuario.findFirst({
  where: { email: userInput }
});
```
> ⚠️ **Atenção:** Mantenha atenção máxima ao usar métodos de execução crua (*Raw Query*) em ORMs. Métodos como `db.$queryRawUnsafe(...)` ou `.raw(...)` reabrem a brecha caso não utilizem placeholders seguros.

#### 3. Princípio do Menor Privilégio (*Least Privilege*)
* A aplicação web **nunca** deve se conectar ao banco utilizando o usuário `root`, `postgres` ou `sa`.
* A conta de serviço da aplicação deve possuir apenas os privilégios estritamente necessários (`SELECT`, `INSERT`, `UPDATE`, `DELETE`).
* Revogue permissões de DDL (`DROP`, `ALTER`, `CREATE`) e acesso a tabelas do sistema na conta utilizada pelo backend em produção.

#### 4. Validação Rigorosa de Tipos e *Whitelisting*
* Valide e sanitize todas as entradas na borda da aplicação com bibliotecas de esquema (Zod, Pydantic, Bean Validation).
* Para identificadores dinâmicos que não podem ser parametrizados (como nomes de colunas em cláusulas `ORDER BY`), use sempre uma lista branca estrita (*whitelist*):
  ```typescript
  const allowedSortColumns = ['nome', 'data_criacao', 'preco'];
  const sortColumn = allowedSortColumns.includes(userSort) ? userSort : 'data_criacao';
  ```

#### 5. Ferramentas de Proteção Ativa
* **WAF (*Web Application Firewall*):** Filtra assinaturas de injeção comuns na camada de borda (Cloudflare, AWS WAF).
* **Análise Estática de Segurança (SAST):** Ferramentas no pipeline de CI/CD (SonarQube, Snyk, CodeQL) que bloqueiam PRs contendo concatenações diretas de strings em instruções SQL.

💡 **Analogia:**
* **SQL Injection:** Um entregador recebe um bilhete para entregar: *"Entregar para João. P.S.: Queime o depósito."* Se o entregador ler o papel e tomar as palavras escritas como uma ordem direta para ele mesmo, o depósito pegará fogo.
* **Prepared Statements (Parametrização):** O entregador recebe uma caixa lacrada e etiquetada com destino fixo. A instrução do entregador é invariável: *"Coloque o conteúdo da caixa dentro do armário do João"*. Não importa se dentro da caixa está escrito "Queime o depósito", "Exploda tudo" ou "Apague os dados"; aquilo é apenas uma carga inerte dentro da caixa que ninguém lê nem executa como ordem.
