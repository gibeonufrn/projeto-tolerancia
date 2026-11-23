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

## Esta parte: a linha de base

Este repositório corresponde à **Parte 1** do projeto: a versão básica do sistema,
instrumentada para ser observada e **sem nenhum mecanismo de tolerância a falhas**.

A ausência de proteção não é descuido, é o requisito!


## O que deve ser feito?

Nesta parte, a equipe deve implementar por conta própria apenas o serviço IMDTravel. Os demais 
serviços devem ser considerados como serviços externos ("caixas-pretas"). A equipe pode usar a linguagem 
e framework que desejar. A implementação do IMDTravel deve realizar as requisições aos demais serviços através dos containers docker
e **NÃO** deve implementar nenhuma estratégia de Tolerância a Falhas.

Entre as imagens docker publicadas já existe um dele com o serviço IMDTravel de referência. Use este apenas como testes da sua infraestrutura e também para servir como referência para sua implementação.

A versão entregue fica marcada com a tag **`BASELINE`**.

## Documentação

| Documento | Conteúdo |
|---|---|
| [Especificação da Parte 1](docs/parte1-especificacao.md) | Requisitos: os serviços, a instrumentação, o roteiro de observação e a entrega. **É a fonte da verdade.** |
| [Especificação da Parte 2](docs/parte2-especificacao.md) | Requisitos: Em construção .** |
| [Como executar](EXECUCAO.md) | O que instalar, como subir e parar os serviços, o catálogo e a configuração. |
| [Como testar](TESTES.md) | Testes manuais com `curl`, o script `testes.sh` (o que verifica, como ler o resultado) e os testes unitários. |
