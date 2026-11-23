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

## Esta parte: duas etapas, nenhuma proteção

Este repositório corresponde à **Parte 1** do projeto: o sistema **sem nenhum mecanismo de
tolerância a falhas**, instrumentado para ser observado. A ausência de proteção não é
descuido, é o requisito! A Parte 1 é feita em duas etapas:

| Etapa | O que muda | Documentos |
|---|---|---|
| **1. A versão base** | O sistema funcionando, com as dependências sempre sadias. | [Especificação](docs/parte1-especificacao.md) · [Como executar](EXECUCAO.md) · [Como testar](TESTES.md) |
| **2. Injeção de falhas** | As dependências passam a falhar de propósito, e se observa o estrago. | [Injeção de falhas](FALHAS.md) · [Testes da etapa 2](TESTES-FALHAS.md) |

Na **Parte 2**, a equipe implementa no IMDTravel as estratégias de tolerância a falhas que o
tornam resiliente às falhas da etapa 2, que continuam acontecendo nos outros serviços
([especificação](docs/parte2-especificacao.md)).

### Etapa 1 — a versão base

Nesta etapa, a equipe deve implementar por conta própria apenas o serviço IMDTravel. Os demais
serviços devem ser considerados como serviços externos ("caixas-pretas"). A equipe pode usar a linguagem
e framework que desejar. A implementação do IMDTravel deve realizar as requisições aos demais serviços através dos containers docker
e **NÃO** deve implementar nenhuma estratégia de Tolerância a Falhas.

Entre as imagens docker publicadas já existe um dele com o serviço IMDTravel de referência. Use este apenas como testes da sua infraestrutura e também para servir como referência para sua implementação.

A versão entregue fica marcada com a tag **`BASELINE`**.

### Etapa 2 — injeção de falhas

Os três serviços de dependência ganham um modelo de falhas, desligado por padrão e ligado
na subida dos containers por variáveis de ambiente. Cada endpoint alvo tem um tipo de falha
fixo:

| Endpoint | Serviço | Falha | O que acontece |
|---|---|---|---|
| `GET /flight` | AirlinesHub | Tempo | A resposta certa, atrasada |
| `POST /sell` | AirlinesHub | Omissão | O assento é vendido e a resposta nunca chega |
| `GET /convert` | Exchange | Resposta | 200 com uma taxa de câmbio errada |
| `POST /bonus` | Fidelity | Crash | O Fidelity sai do ar e o bônus não é creditado |

O IMDTravel da etapa 1 continua sem nenhuma proteção. O que se faz nesta etapa é ligar cada
falha e medir o que ela causa: compras lentas, assentos vendidos sem registro, vendas com
valor absurdo, bônus perdidos. Esses números são a base de comparação da Parte 2.

## Documentação

| Documento | Conteúdo |
|---|---|
| [Especificação da Parte 1](docs/parte1-especificacao.md) | Etapa 1. Requisitos: os serviços, a instrumentação, o roteiro de observação e a entrega. **É a fonte da verdade.** |
| [Como executar](EXECUCAO.md) | Etapa 1. O que instalar, como subir e parar os serviços, o catálogo e a configuração. |
| [Como testar](TESTES.md) | Etapa 1. Testes manuais com `curl`, o script `testes.sh` (o que verifica, como ler o resultado) e os testes unitários. |
| [Injeção de falhas](FALHAS.md) | Etapa 2. O modelo de falhas: os tipos, o que cada um faz, como ligar e como ver no log. |
| [Testes da etapa 2](TESTES-FALHAS.md) | Etapa 2. O script `cenario.sh`, testes manuais de cada falha e os testes unitários da injeção. |
| [Plano da injeção de falhas](docs/injecao-falhas-plano.md) | Etapa 2. As decisões de projeto e os detalhes de implementação. |
| [Especificação da Parte 2](docs/parte2-especificacao.md) | Parte 2. As estratégias de tolerância a falhas que a equipe implementa no IMDTravel, os experimentos e a entrega. |
