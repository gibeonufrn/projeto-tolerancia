# IMDTravel — Tolerância a Falhas (DIMAp/UFRN)

Projeto da disciplina de Tolerância a Falhas do DIMAp/UFRN: um sistema didático de venda
de passagens aéreas, construído para ser derrubado de propósito e medido.

## O sistema

Um cliente pede uma compra ao **IMDTravel**, que orquestra quatro chamadas síncronas, em
ordem obrigatória, a três serviços de dependência:

```
cliente ──POST /buyTicket──▶ IMDTravel ──1. GET  /flight ──▶ AirlinesHub   preço e assentos
                                │      ──2. GET  /convert──▶ Exchange      taxa de câmbio
                                │      ──3. POST /sell   ──▶ AirlinesHub   consome o assento
                                │      ──4. POST /bonus  ──▶ Fidelity      credita os pontos
                                ▼
                     registro local de vendas (GET /sales)
```

| Serviço | Papel |
|---|---|
| **IMDTravel** | Orquestrador: recebe a compra, faz as quatro chamadas e registra a venda. |
| **AirlinesHub** | Mantém o estoque de assentos por voo e dia. É o único estado que importa. |
| **Exchange** | Fornece a taxa de câmbio, que varia devagar ao longo do tempo. |
| **Fidelity** | Acumula pontos de fidelidade por usuário. |

Cada serviço roda no seu próprio container, e todos sobem juntos com Docker Compose.
Os serviços são escritos em Java com Spring Boot.

## As duas partes do projeto

O projeto é desenvolvido em duas partes **subsequentes**, pela mesma equipe e sobre o mesmo
repositório. A Parte 2 só começa depois que a Parte 1 está entregue, e parte do IMDTravel
que ela produziu.

```
Parte 1 — a versão base                 Parte 2 — falhas e tolerância
  o sistema funcionando, sem              ponto de partida: as dependências passam a
  nenhuma proteção                ───▶      falhar e se mede o estrago na BASELINE
  tag BASELINE                            depois: o IMDTravel passa a tolerar essas falhas
                                          tag FAULT-TOLERANT
```

| | **Parte 1** | **Parte 2** |
|---|---|---|
| **Objetivo** | Construir e instrumentar o IMDTravel, com as dependências sadias. | Injetar falhas nas dependências, medir o estrago e tornar o IMDTravel resiliente a elas. |
| **Tolerância a falhas** | **Proibida.** A ausência de proteção é o requisito. | **Obrigatória**, toda no IMDTravel. |
| **O que a equipe implementa** | O IMDTravel. | O IMDTravel, a partir da versão da Parte 1. |
| **Dependências** | AirlinesHub, Exchange e Fidelity prontos, como caixas-pretas. | As mesmas, agora com a injeção de falhas ligada. Sem nenhuma alteração pela equipe. |
| **Entrega** | Tag `BASELINE` | Tag `FAULT-TOLERANT` (a `BASELINE` continua existindo) |
| **Especificação** | [Parte 1](docs/parte1-especificacao.md) | [Parte 2](docs/parte2-especificacao.md) |

---

## Parte 1 — a versão base

A equipe implementa por conta própria apenas o serviço IMDTravel. Os demais serviços devem
ser considerados como serviços externos ("caixas-pretas"). A equipe pode usar a linguagem e
o framework que desejar. O IMDTravel faz as requisições aos demais serviços através dos
containers Docker e **NÃO** deve implementar nenhuma estratégia de tolerância a falhas.

Entre as imagens Docker publicadas já existe uma com um IMDTravel de referência. Use-a
apenas para testar a sua infraestrutura e como referência para a sua implementação.

O que se entrega: as quatro chamadas na ordem, o registro de vendas e a instrumentação
(correlação, log estruturado, `/health`). A versão entregue fica marcada com a tag
**`BASELINE`**.

**Documentos da Parte 1:**

| Documento | Conteúdo |
|---|---|
| [Especificação da Parte 1](docs/parte1-especificacao.md) | Os requisitos: os serviços, a instrumentação e a entrega. **É a fonte da verdade.** |
| [Como executar](EXECUCAO.md) | O que instalar, como subir e parar os serviços, o catálogo e a configuração. |
| [Como testar](TESTES.md) | Testes manuais com `curl`, o script `testes.sh` e os testes unitários. |

---

## Parte 2 — falhas e tolerância a falhas

Começa a partir da tag `BASELINE`, em dois passos.

**Ponto de partida: as dependências passam a falhar.** Os três serviços de dependência
trazem um modelo de falhas, desligado por padrão e ligado na subida dos containers por
variáveis de ambiente. Cada endpoint alvo tem um tipo de falha fixo:

| Endpoint | Serviço | Falha | O que acontece |
|---|---|---|---|
| `GET /flight` | AirlinesHub | Tempo | A resposta certa, atrasada |
| `POST /sell` | AirlinesHub | Omissão | O assento é vendido e a resposta nunca chega |
| `GET /convert` | Exchange | Resposta | 200 com uma taxa de câmbio errada |
| `POST /bonus` | Fidelity | Crash | O Fidelity sai do ar e o bônus não é creditado |

Com o IMDTravel da `BASELINE`, ainda sem nenhuma proteção, a equipe liga cada falha e mede o
que ela causa: compras lentas, assentos vendidos sem registro, vendas com valor absurdo,
bônus perdidos. Esses números são a linha de comparação.

**Depois: a tolerância.** As falhas continuam acontecendo do mesmo jeito. A equipe
implementa no IMDTravel as estratégias de tolerância a falhas que o tornam resiliente a
elas, e demonstra com testes que os números do ponto de partida melhoraram.

A versão entregue fica marcada com a tag **`FAULT-TOLERANT`**. A tag `BASELINE` precisa
continuar existindo e funcionando, porque a comparação depende dela.

**Documentos da Parte 2:**

| Documento | Conteúdo |
|---|---|
| [Especificação da Parte 2](docs/parte2-especificacao.md) | O ponto de partida, o que o IMDTravel precisa garantir sob falha, as restrições e a entrega. **É a fonte da verdade.** |
| [Injeção de falhas](FALHAS.md) | O modelo de falhas: os tipos, o que cada um faz, como ligar e como ver no log. |
| [Testes com falhas](TESTES-FALHAS.md) | O script `cenario.sh`: liga cada falha, faz compras e mostra o estrago. |
| [Plano da injeção de falhas](docs/injecao-falhas-plano.md) | As decisões de projeto e os detalhes de implementação da injeção. |
