# Especificação do Projeto (Tolerância a Falhas)

**Unidade 2 – Parte 2 · Tolerância a falhas no IMDTravel, sem replicação**

|                         |                                                                                                                                              |
|-------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| **Objetivo**            | Medir o estrago das falhas injetadas nas dependências e implementar no IMDTravel as estratégias de tolerância a falhas que o tornam resiliente a elas. |
| **O que se implementa** | **Só o IMDTravel.** AirlinesHub, Exchange e Fidelity são os mesmos da Parte 1: caixas-pretas, com o mesmo contrato, agora com a injeção de falhas ligada (seção 3). |
| **Equipe**              | A mesma da Parte 1. Mínimo 2, máximo 3 pessoas.                                                                                              |
| **Pré-requisito**       | A tag BASELINE da Parte 1.                                                                                                                   |
| **Entrega**             | Evidências do ponto de partida (a BASELINE sob falha), tag FAULT-TOLERANT, testes que demonstram cada propriedade e as novas evidências. |
| **Restrição central**   | Nenhum serviço é replicado. Cada serviço continua sendo exatamente um container, e nenhum serviço de dependência é alterado.                 |

> **Segunda de duas partes subsequentes.** Esta parte começa onde a [Parte 1](parte1-especificacao.md) terminou, com o IMDTravel da tag BASELINE. Primeiro, as dependências passam a falhar e se mede o estrago; depois, o IMDTravel é tornado resiliente a essas falhas.

**Documentos desta parte**

| Documento | Conteúdo |
|---|---|
| Esta especificação | O que o IMDTravel precisa garantir sob falha, as restrições e a entrega. |
| [Injeção de falhas](../FALHAS.md) | O modelo de falhas das dependências: os tipos, o que cada um faz, como ligar e como ver no log. |
| [Testes com falhas](../TESTES-FALHAS.md) | O script `cenario.sh`, que liga cada falha e mede o estrago. |

> **Leitura de apoio.** *Patterns for Fault Tolerant Software* (HANMER, Wiley, 2007), capítulos 5, 6 e 7, vistos nas aulas 07, 08 e 09.

# 1. O que esta parte pede

Na Parte 1, o IMDTravel foi construído sob uma hipótese que nunca foi dita em voz alta: a de que toda dependência responde, responde a tempo e responde certo. Esta parte derruba essa hipótese e depois conserta o que ela deixou frágil, em dois passos.

**Passo 1 · Ponto de partida: as dependências passam a falhar.** Os três serviços de dependência trazem um modelo de falhas ([Injeção de falhas](../FALHAS.md)), ligado por variáveis de ambiente na subida dos containers (seção 3). Com o IMDTravel da tag BASELINE, ainda sem nenhuma proteção, a equipe liga cada falha e mede o estrago — com o `cenario.sh` ([Testes com falhas](../TESTES-FALHAS.md)) ou com os próprios testes. Essas medições são a linha de comparação do passo 2 e fazem parte da entrega.

**Passo 2 · A tolerância.** **A equipe implementa no IMDTravel as estratégias de tolerância a falhas que tornam o sistema resiliente a essas falhas.** As falhas continuam acontecendo nos outros serviços, do mesmo jeito e com a mesma configuração. O que muda é o IMDTravel.

Esta especificação diz **o que** o IMDTravel precisa garantir (seção 4), e não como. Escolher as estratégias, combiná-las e justificar os parâmetros é o trabalho da equipe. Cada estratégia adotada é nomeada no README, com o padrão do catálogo de onde ela vem, o lugar no código e o teste que a demonstra (seção 6).

> O entregável desta parte não é “o sistema funciona”. É “o sistema continua útil enquanto as dependências falham, de forma verificável”. Um IMDTravel que passa no caminho feliz e trava na primeira falha injetada vale zero aqui, mesmo que o código esteja limpo.

# 2. Restrições de escopo

## 2.1 O que é obrigatório

- **Só o IMDTravel muda.** Os três serviços de dependência são usados como estão, pelas imagens publicadas. A equipe não altera o código, o contrato nem a configuração de falha deles.

- Um container por serviço. IMDTravel, AirlinesHub, Exchange e Fidelity: quatro containers, um de cada.

- Docker Compose como única forma de subir o sistema. Um `docker compose up` sobe tudo, sem passo manual anterior.

- Toda a configuração das estratégias por variável de ambiente, com valor padrão declarado no compose.

- É permitido um único container adicional de infraestrutura (banco, broker ou cache) se alguma estratégia exigir. Um volume com SQLite ou arquivo também resolve e é aceito. Isso não conta como replicação.

- Pode-se usar frameworks de resiliência prontos (Resilience4j, Polly, opossum, Hystrix e similares), desde que se consiga descrever precisamente qual a estratégia de tolerância a falhas usada, o comportamento e os parâmetros escolhidos. 

- O `GET /health`, a correlação por `X-Correlation-Id`, o log estruturado e o registro local de vendas (`GET /sales`) vêm da Parte 1 e continuam valendo.

## 2.2 O que fica fora desta versão

Os padrões abaixo dependem de haver mais de uma cópia de alguma coisa. Nenhum deles entra agora.

| **Nº** | **Padrão**            | **Por que fica para a próxima parte**                                         |
|--------|-----------------------|-------------------------------------------------------------------------------|
| **36** | Failover              | Exige um elemento redundante preparado para assumir. Não há segundo elemento. |
| **51** | Share the Load        | Exige nós entre os quais redistribuir trabalho.                               |
| **21** | Voting                | Exige três ou mais execuções da mesma função para decidir por maioria.        |
| **57** | Error Correcting Code | Exige redundância de informação suficiente para reconstruir o valor.          |

- Sem Kubernetes, Swarm, service mesh, balanceador de carga ou proxy reverso com failover.

- Sem múltiplas instâncias do mesmo serviço, nem com `deploy: replicas`, nem com dois containers de mesma imagem.

- Sem serviço externo pago ou SaaS. O sistema inteiro precisa subir sem rede externa além do download das imagens.

# 3. O sistema e as falhas que ele vai sofrer

O sistema é o mesmo da Parte 1, com o mesmo contrato entre os serviços e as quatro chamadas na mesma ordem.

```
cliente ──POST /buyTicket──▶ IMDTravel ──1. GET  /flight ──▶ AirlinesHub   falha: Tempo
                                │      ──2. GET  /convert──▶ Exchange      falha: Resposta
                                │      ──3. POST /sell   ──▶ AirlinesHub   falha: Omissão
                                │      ──4. POST /bonus  ──▶ Fidelity      falha: Crash
                                ▼
                     registro local de vendas (GET /sales) · GET /status (novo)
```

## 3.1 O modelo de falhas

As falhas são descritas em [Injeção de falhas](../FALHAS.md). Cada endpoint alvo tem um tipo fixo; só a probabilidade `p` e a duração `ttlSec` da janela de falha mudam, pelas variáveis `FAULT_<ENDPOINT>_P` e `FAULT_<ENDPOINT>_TTL_SEC`.

| **Chamada**     | **Tipo**     | **O que o IMDTravel vê**                                                   |
|-----------------|--------------|----------------------------------------------------------------------------|
| `GET /flight`   | **Tempo**    | A resposta certa, só no fim da janela de falha.                            |
| `GET /convert`  | **Resposta** | 200 com `rate` zero, negativo, absurdo ou não numérico.                    |
| `POST /sell`    | **Omissão**  | A venda acontece, mas nenhum byte volta; a conexão fecha no fim da janela. |
| `POST /bonus`   | **Crash**    | Conexão recusada: o Fidelity inteiro fica fora do ar até o fim da janela.  |

**Fatos do contrato que valem lembrar:**

- O preço de um par (voo, dia) é determinístico. A disponibilidade é decidida pelo `POST /sell` (201 ou 409).
- A taxa do Exchange caminha devagar e fica sempre entre 5,00 e 6,00 (`EXCHANGE_MIN`, `EXCHANGE_MAX`).
- O `POST /sell` leva de 300 a 900 ms (`SELL_MIN_MS`, `SELL_MAX_MS`).
- O AirlinesHub e o Fidelity **não** oferecem chave de idempotência: cada `POST` aceito produz o seu efeito.


# 4. O que o IMDTravel precisa garantir

As propriedades abaixo são obrigatórias. Cada uma precisa ser demonstrada por pelo menos um teste, indicado na tabela de rastreabilidade (seção 6).

| **Nº**  | **Propriedade**                         | **O que precisa valer, com qualquer combinação das falhas da seção 3**                                                                                                     |
|---------|-----------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **P1**  | Prazo total                             | Nenhuma compra responde depois de `5000ms`, contado da chegada da requisição 0. Vale também com uma dependência pausada.                                         |
| **P2**  | Nenhum valor inválido                   | Nenhuma venda é gravada em `/sales` nem devolvida ao cliente com taxa fora de [5.0 , 6.0], não numérica, ou com valor derivado dela.                           |
| **P3**  | No máximo um assento por compra         | Uma requisição de compra consome no máximo um assento no AirlinesHub, aconteça o que acontecer com as respostas.                                                         |
| **P4**  | Resposta honesta                        | 201 só quando a venda foi confirmada pelo AirlinesHub. Quando o IMDTravel não puder saber se a venda aconteceu, a resposta diz isso (seção 5). Compra concluída em condição degradada declara a degradação. |
| **P5**  | O Fidelity não é essencial              | Falha do Fidelity nunca faz a compra falhar. Todo bônus devido é creditado, exatamente uma vez, depois que o Fidelity volta.                                               |
| **P6**  | Compra possível sob Tempo e Resposta    | Com o `/flight` lento ou o `/convert` errado, a compra continua possível. Dado usado numa compra degradada não pode ser mais velho que a idade máxima declarada pela equipe. |
| **P7**  | Falha rápida e volta sozinha            | Com uma dependência falhando de forma persistente, as compras afetadas respondem bem antes do prazo total e o IMDTravel deixa de sobrecarregar a dependência. Quando ela volta, o sistema volta ao normal sem intervenção. |

# 5. Resposta da compra

A resposta da requisição 0 deixa de ser apenas sucesso ou fracasso. Os campos da Parte 1 continuam, e dois são acrescentados:

- `degraded`: `true` quando a compra foi concluída em alguma condição degradada;
- `degradations`: a lista do que foi degradado. Os valores possíveis são definidos pela equipe e documentados no README.



# 6. Forma de entrega

- Todo o código, os Dockerfiles e o docker-compose.yml no mesmo repositório da parte 1 no GitHub.

- Tag FAULT-TOLERANT marcando esta versão. A tag BASELINE precisa continuar existindo e funcionando, porque a comparação futura depende dela.

- README com: como subir, como rodar os testes que demonstram as propriedades, os valores adotados para todos os parâmetros e a justificativa de cada um, e a tabela de rastreabilidade descrita abaixo.

A tabela de rastreabilidade do README liga cada propriedade (falha tolerada) às estratégias adotadas, ao código e à evidência. 

| **Falha tolerada** | **Estratégia adotada (padrão do catálogo)** | **Onde está no código** | **Teste que demonstra** |
|-----------------|---------------------------------------------|-------------------------|-------------------------------|
| **P1**          | …                                           | …                       | …                             |
| **…**           | …                                           | …                       | …                             |

