# IMDTravel — Parte 2: injeção de falhas

Na [Parte 1](README.md#parte-1--a-versão-base) o sistema foi construído e observado com as
três dependências sempre sadias. A [Parte 2](docs/parte2-especificacao.md) começa aqui: as
dependências passam a **falhar de propósito**. Cada uma traz um modelo de falhas que se liga
na subida dos containers.

É o ponto de partida da Parte 2. Primeiro, com o IMDTravel da tag `BASELINE`, ainda **sem
nenhuma proteção**, mede-se o estrago que cada falha causa, com números e com o log. Depois,
a equipe torna o IMDTravel resiliente a estas mesmas falhas, que continuam acontecendo do
mesmo jeito.

> A injeção de falhas é **instrumento de observação**, não tolerância a falhas.
> Ela só piora os serviços de dependência. Nada nela protege o IMDTravel.

Para testar e observar cada falha, veja [Testes com falhas](TESTES-FALHAS.md). As decisões
de projeto e os detalhes de implementação estão no
[plano da injeção de falhas](docs/injecao-falhas-plano.md).

---

## 1. O modelo

Quatro tipos de falha, um em cada endpoint alvo. O par (endpoint, tipo) e o comportamento de
cada tipo são **fixos**:

| Serviço | Endpoint | Tipo de falha | O que o IMDTravel vê | Efeito colateral |
|---|---|---|---|---|
| AirlinesHub | `GET /flight` | **Tempo** | A resposta correta, mas atrasada até o fim da janela | não tem |
| AirlinesHub | `POST /sell` | **Omissão** | Conexão aceita, nenhum byte de volta; no fim da janela, a conexão é fechada | **acontece**: o assento é vendido e a resposta se perde |
| Exchange | `GET /convert` | **Resposta** | Status 200 com um `rate` errado | não tem |
| Fidelity | `POST /bonus` | **Crash** | Conexão recusada: o **Fidelity inteiro** fica fora do ar até o fim da janela | **não acontece**: o bônus não é creditado |

Os demais endpoints (`/health`, `/points`, `/sales`, `/buyTicket`) não são alvo. O Crash é a
exceção: ao derrubar o Fidelity, ele leva junto o `/points` e o `/health`.

Cada endpoint tem só dois parâmetros:

- **`p`**: probabilidade de a falha disparar em cada requisição, entre 0 e 1;
- **`ttlSec`**: depois que a falha dispara, por quantos segundos o endpoint fica em falha.

```
           sorteio < p                      agora >= fim da janela
 NORMAL ───────────────────► EM FALHA ──────────────────────────► NORMAL
   ▲  │ sorteio >= p           │ toda requisição dentro da janela
   └──┘ (atende normal)        │ sofre a falha, sem novo sorteio
```

- `p = 0` desliga a falha. É o padrão: sem nenhuma variável, as dependências se comportam
  como na Parte 1.
- `p = 1` com `ttlSec > 0` é falha contínua: a janela reabre na requisição seguinte.
- Tempo, Omissão e Crash só existem dentro de uma janela e exigem `ttlSec > 0` quando
  `p > 0`. A Resposta aceita `ttlSec = 0`: só a requisição sorteada é afetada.

## 2. O que cada falha faz

**Tempo (`/flight`).** A resposta só sai no fim da janela, correta. Todas as requisições que
chegam durante a janela esperam juntas e são respondidas juntas.

**Omissão (`/sell`).** O AirlinesHub **vende o assento**, segura a resposta até o fim da
janela e fecha a conexão sem mandar nada. Para o IMDTravel, a venda falhou; para o
AirlinesHub, ela aconteceu. O assento some do estoque e a venda não aparece em `/sales`.
Cada requisição omitida ocupa uma thread do AirlinesHub durante a janela inteira: com muitas
ao mesmo tempo, o AirlinesHub para de atender também o `/flight` e o `/health`.

**Resposta (`/convert`).** O Exchange responde 200, mas com o `rate` trocado por um valor
errado, sorteado a cada requisição:

| Variante | `rate` enviado | O que um IMDTravel sem validação faz |
|---|---|---|
| zero | `0` | 201, venda gravada com `valueBRL = 0` |
| negativo | a taxa real com o sinal trocado | 201, venda gravada com `valueBRL` negativo |
| absurdo | a taxa real × 10^k, com k entre 3 e 9 | 201, venda gravada na casa dos bilhões |
| não numérico | `"5,50"`, `"NaN"`, `"abc"` ou `""` | 500 antes de vender: o `/convert` vem antes do `/sell` |

**Crash (`/bonus`).** O bônus não é creditado e o Fidelity para de aceitar conexões até o fim
da janela; depois volta sozinho, no mesmo container. Como o `/bonus` é a última chamada, o
assento já foi vendido quando ele cai. Com uma janela de mais de uns 25 s, o healthcheck do
Docker marca o Fidelity como `unhealthy`.

**O `/health` não denuncia Tempo nem Omissão:** ele continua respondendo 200 enquanto o
endpoint falha. Um health check que só olha a si mesmo não vê a falha.

## 3. Como ligar

Oito variáveis de ambiente, duas por endpoint, todas com padrão `0` no `docker-compose.yml`:

| Serviço | Endpoint | Variáveis |
|---|---|---|
| AirlinesHub | `GET /flight` | `FAULT_FLIGHT_P`, `FAULT_FLIGHT_TTL_SEC` |
| AirlinesHub | `POST /sell` | `FAULT_SELL_P`, `FAULT_SELL_TTL_SEC` |
| Exchange | `GET /convert` | `FAULT_CONVERT_P`, `FAULT_CONVERT_TTL_SEC` |
| Fidelity | `POST /bonus` | `FAULT_BONUS_P`, `FAULT_BONUS_TTL_SEC` |

As variáveis só valem na subida. Não existe API para ligar ou desligar uma falha com o
serviço no ar: para mudar o cenário, recria-se o container.

```
# Omissão no /sell: 20% das requisições abrem uma janela de 5 s
FAULT_SELL_P=0.2 FAULT_SELL_TTL_SEC=5 docker compose up -d --wait airlineshub
docker compose restart imdtravel

# Desliga a falha: recria com os padrões
docker compose up -d --wait airlineshub
docker compose restart imdtravel
```

**Reinicie o IMDTravel depois de recriar uma dependência.** Recriar um container pode mudar
o IP dele, e a JVM do IMDTravel guarda o IP antigo por cerca de 30 s. Nesse intervalo, uma
chamada ao AirlinesHub pode cair noutro serviço e voltar com um 404 sem sentido.

**Configuração inválida impede o serviço de subir**, e a mensagem diz qual variável está
errada:

```
FAULT_SELL: a falha omissao só existe dentro de uma janela; com p > 0, ttlSec precisa ser maior que 0
```

**Imagens do Docker Hub.** A injeção existe nas imagens com a tag `parte2`, que é o padrão
dos dois arquivos de compose: `docker compose -f docker-compose.hub.yml up -d --wait` já sobe
as dependências com a injeção, desligada. As imagens `baseline` são as da Parte 1 e ignoram
as variáveis `FAULT_*`; para usá-las, `IMAGE_TAG=baseline`.

## 4. Como ver no log

Cada falha deixa eventos no log JSON do serviço que falhou, com o `cid` da compra quando há
um:

| Evento | Quando |
|---|---|
| `falha.configurada` | Na subida, um por endpoint com `p > 0` |
| `falha.disparada` | O sorteio venceu e abriu uma janela (`fimJanela`) |
| `falha.aplicada` | Cada requisição afetada |
| `falha.janela.fechada` | A janela fechou (`fimJanela` traz o instante real) |
| `falha.corrupcao` | Resposta: `variante`, `rateReal`, `rateEnviado` |
| `resposta.omitida` | Omissão e Crash, no lugar de `resposta.enviada`; `statusOmitido` é o status que nunca saiu |
| `falha.crash.inicio` / `falha.crash.fim` | Crash: Fidelity fora do ar e de volta; o `ms` do fim é o tempo fora do ar |

```
docker compose logs airlineshub exchange fidelity | grep '"event":"falha\.'
docker compose logs | grep <cid>          # o caminho de uma compra nos quatro serviços
```

## 5. Onde fica o código

| Caminho | Conteúdo |
|---|---|
| `common/.../common/falhas/` | O modelo (`ModeloFalha`, `TipoFalha`, `ConfiguracaoFalha`), o filtro que aplica a falha (`InjecaoFalhas`), o fechamento da conexão sem resposta (`AbortoConexao`) e a queda do serviço (`Crash`) |
| `airlineshub/.../FalhasAirlinesHub.java` | `/flight` = Tempo, `/sell` = Omissão |
| `exchange/.../FalhasExchange.java`, `CorrupcaoTaxa.java` | `/convert` = Resposta e as variantes do `rate` errado |
| `fidelity/.../FalhasFidelity.java` | `/bonus` = Crash |

Os controllers dos serviços não mudaram em relação à Parte 1.
