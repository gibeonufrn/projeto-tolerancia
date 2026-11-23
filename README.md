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
repositório. A Parte 2 só começa depois que a Parte 1 está entregue, e parte do código e das
medições dela.

```
Parte 1 — a versão base, sem proteção           Parte 2 — tolerância a falhas
  etapa 1: o sistema funcionando                  o IMDTravel passa a tolerar
  etapa 2: as dependências passam a falhar  ───▶  as falhas da etapa 2
  tag BASELINE                                    tag FAULT-TOLERANT
```

| | **Parte 1** | **Parte 2** |
|---|---|---|
| **Objetivo** | Construir o IMDTravel e medir o estrago que as falhas das dependências causam nele. | Tornar o IMDTravel resiliente a essas mesmas falhas. |
| **Tolerância a falhas** | **Proibida.** A ausência de proteção é o requisito. | **Obrigatória**, toda no IMDTravel. |
| **O que a equipe implementa** | O IMDTravel. | O IMDTravel, a partir da versão da Parte 1. |
| **Dependências** | AirlinesHub, Exchange e Fidelity prontos, como caixas-pretas. | As mesmas, sem nenhuma alteração. |
| **Entrega** | Tag `BASELINE` | Tag `FAULT-TOLERANT` (a `BASELINE` continua existindo) |
| **Especificação** | [Parte 1](docs/parte1-especificacao.md) | [Parte 2](docs/parte2-especificacao.md) |

---

## Parte 1 — a versão base, sem tolerância a falhas

A equipe implementa por conta própria apenas o serviço IMDTravel. Os demais serviços devem
ser considerados como serviços externos ("caixas-pretas"). A equipe pode usar a linguagem e
o framework que desejar. O IMDTravel faz as requisições aos demais serviços através dos
containers Docker e **NÃO** deve implementar nenhuma estratégia de tolerância a falhas.

Entre as imagens Docker publicadas já existe uma com um IMDTravel de referência. Use-a
apenas para testar a sua infraestrutura e como referência para a sua implementação.

A Parte 1 tem duas etapas:

**Etapa 1 — o sistema funcionando.** As quatro chamadas, o registro de vendas e a
instrumentação (correlação, log estruturado, `/health`), com as dependências sempre sadias.

**Etapa 2 — injeção de falhas.** Os três serviços de dependência passam a falhar de
propósito. O modelo de falhas é desligado por padrão e ligado na subida dos containers, por
variáveis de ambiente. Cada endpoint alvo tem um tipo de falha fixo:

| Endpoint | Serviço | Falha | O que acontece |
|---|---|---|---|
| `GET /flight` | AirlinesHub | Tempo | A resposta certa, atrasada |
| `POST /sell` | AirlinesHub | Omissão | O assento é vendido e a resposta nunca chega |
| `GET /convert` | Exchange | Resposta | 200 com uma taxa de câmbio errada |
| `POST /bonus` | Fidelity | Crash | O Fidelity sai do ar e o bônus não é creditado |

O IMDTravel continua sem nenhuma proteção. O que se faz nesta etapa é ligar cada falha e
medir o que ela causa: compras lentas, assentos vendidos sem registro, vendas com valor
absurdo, bônus perdidos. Esses números são a base de comparação da Parte 2.

A versão entregue fica marcada com a tag **`BASELINE`**.

**Documentos da Parte 1:**

| Documento | Conteúdo |
|---|---|
| [Especificação da Parte 1](docs/parte1-especificacao.md) | Os requisitos: os serviços, a instrumentação e a entrega. **É a fonte da verdade.** |
| [Como executar](EXECUCAO.md) | Etapa 1. O que instalar, como subir e parar os serviços, o catálogo e a configuração. |
| [Como testar](TESTES.md) | Etapa 1. Testes manuais com `curl`, o script `testes.sh` e os testes unitários. |
| [Injeção de falhas](FALHAS.md) | Etapa 2. O modelo de falhas: os tipos, o que cada um faz, como ligar e como ver no log. |
| [Testes da etapa 2](TESTES-FALHAS.md) | Etapa 2. O script `cenario.sh`, testes manuais de cada falha e os testes unitários da injeção. |
| [Plano da injeção de falhas](docs/injecao-falhas-plano.md) | Etapa 2. As decisões de projeto e os detalhes de implementação da injeção. |

---

## Parte 2 — tolerância a falhas no IMDTravel

Começa a partir da tag `BASELINE`. As dependências continuam as mesmas e continuam falhando
do mesmo jeito da etapa 2 da Parte 1. O que muda é o IMDTravel: a equipe implementa nele as
estratégias de tolerância a falhas que o tornam resiliente a essas falhas, e demonstra com
testes que os números medidos na Parte 1 melhoraram.

A versão entregue fica marcada com a tag **`FAULT-TOLERANT`**. A tag `BASELINE` precisa
continuar existindo e funcionando, porque a comparação depende dela.

**Documentos da Parte 2:**

| Documento | Conteúdo |
|---|---|
| [Especificação da Parte 2](docs/parte2-especificacao.md) | O que o IMDTravel precisa garantir sob falha, as restrições e a entrega. **É a fonte da verdade.** |
| [Injeção de falhas](FALHAS.md) | Herdado da Parte 1: as falhas que o IMDTravel passa a tolerar. |
