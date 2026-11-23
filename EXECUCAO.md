# IMDTravel — como executar

Como instalar o ambiente, subir e parar os quatro serviços, e rodar os testes.
Para a visão geral do projeto, veja o [README](README.md); para os requisitos, a
[especificação da Parte 1](docs/parte1-especificacao.md).

```
cliente ──POST /buyTicket──▶ IMDTravel ──1. GET  /flight ──▶ AirlinesHub
                                │      ──2. GET  /convert──▶ Exchange
                                │      ──3. POST /sell   ──▶ AirlinesHub
                                │      ──4. POST /bonus  ──▶ Fidelity
                                ▼
                     registro local de vendas (GET /sales)
```

---

## 1. O que instalar

| Ferramenta | Para quê | Como conferir |
|---|---|---|
| **Docker** com **Compose v2** | subir os quatro serviços | `docker compose version` |
| **git** | clonar o repositório | `git --version` |
| **bash**, **curl** e **jq** | rodar o `testes.sh` | `bash --version`, `curl --version`, `jq --version` |

Reserve pelo menos **4 GB de memória** para o Docker. Os quatro serviços juntos usam cerca
de 1,2 GB em repouso, e o build precisa de folga.

### macOS
- Instale o [Docker Desktop](https://www.docker.com/products/docker-desktop/) e abra-o uma vez.
- `bash`, `curl` e `jq` já vêm com o macOS (o `jq` desde o macOS 15). Em versões
  anteriores: `brew install jq`.

### Linux (Ubuntu/Debian)
```
sudo apt install docker.io docker-compose-v2 git curl jq
sudo usermod -aG docker $USER      # depois saia e entre de novo na sessão
```
Ou siga a [instalação oficial do Docker Engine](https://docs.docker.com/engine/install/).

### Windows
1. Instale o **WSL 2** com Ubuntu: no PowerShell como administrador, `wsl --install`, e reinicie.
2. Instale o [Docker Desktop](https://www.docker.com/products/docker-desktop/) e deixe marcada
   a opção **"Use WSL 2 based engine"** (é o padrão). Em *Settings → Resources → WSL
   integration*, habilite a integração com o Ubuntu.
3. **Use o terminal do Ubuntu (WSL) para tudo** — clonar, subir e rodar os scripts:
   ```
   sudo apt update && sudo apt install git curl jq
   ```
4. Clone o repositório **dentro do sistema de arquivos do WSL** (por exemplo em `~/`), e não
   em `/mnt/c/...`: fica bem mais rápido e evita problemas de permissão e de fim de linha.

As imagens são Linux e funcionam no Windows através do WSL 2. Não é preciso trocar o Docker
Desktop para o modo "Windows containers".

---

## 2. Subir o sistema

```
git clone <url-do-repositório> imdtravel
cd imdtravel
docker compose up -d --build --wait
```

- `--build` compila os quatro serviços. **A primeira vez leva alguns minutos** (baixa as
  imagens base e as dependências do Maven); as seguintes usam o cache e levam segundos.
- `--wait` só devolve o terminal quando os quatro estão saudáveis.
- Sem `-d`, os logs ficam presos no terminal; `Ctrl+C` para tudo.

### 2.b Com as imagens prontas do Docker Hub (quem só vai usar)

```
mkdir imdtravel && cd imdtravel
```
Dentro de um clone do repositório, o equivalente é:
```
docker compose -f docker-compose.hub.yml up -d --wait
```

As imagens usadas por padrão têm a tag `parte2`: são as da [Parte 2](README.md#parte-2--falhas-e-tolerância-a-falhas),
com a injeção de falhas desligada, e se comportam como as da Parte 1. Para usar exatamente
as imagens da Parte 1: `IMAGE_TAG=baseline docker compose -f docker-compose.hub.yml up -d --wait`.

### Conferir que subiu

```
docker compose ps                     # os quatro devem aparecer como (healthy)
curl http://localhost:8080/health     # {"status":"ok","uptimeS":...}
```

Só o **IMDTravel** tem porta exposta (`8080`). Os outros três ficam na rede interna; para
falar com eles, use `docker compose exec`:
```
docker compose exec airlineshub curl -s localhost:8080/health
docker compose exec airlineshub curl -s "localhost:8080/flight?flight=IMD100&day=2026-12-01"
```

### Fazer uma compra

```
curl -i -H 'Content-Type: application/json' \
     -d '{"flight":"IMD100","day":"2026-12-01","user":"ana"}' \
     http://localhost:8080/buyTicket

curl http://localhost:8080/sales      # o registro local de vendas
```

---

## 3. Parar o sistema

| Comando | O que faz |
|---|---|
| `docker compose -f docker-compose.hub.yml stop` | Para os containers sem removê-los. `docker compose -f docker-compose.hub.yml start` volta de onde parou. |
| `docker compose -f docker-compose.hub.yml down` | Para e remove os containers e a rede. **O registro de vendas é mantido** (volume `sales-data`). |
| `docker compose -f docker-compose.hub.yml down -v` | Remove também o volume: **apaga o registro de vendas**. É o "começar do zero". |


Lembre que o **estoque de assentos, a taxa de câmbio e os pontos de fidelidade vivem em
memória**: reiniciar um serviço zera o estado dele. Só o registro de vendas do IMDTravel
sobrevive, porque está no volume.

---

## 4. Testar

Com o sistema no ar, na raiz do repositório:

```
./testes.sh                  # ~10 s; verifica o sistema inteiro, de fora
./testes.sh --com-falhas     # inclui o teste com o Fidelity parado (pode levar minutos)
./testes.sh --reiniciar      # APAGA o volume e sobe do zero antes de testar
```

Termina com `Resumo: N passaram, 0 falharam`. O guia completo — testes manuais com `curl`,
o que cada seção do `testes.sh` verifica, como ler uma falha, variáveis de ambiente e testes
unitários — está em **[Como testar](TESTES.md)**.

Para a Parte 2, com falhas injetadas: [Testes com falhas](TESTES-FALHAS.md).

---

## 5. Catálogo e configuração

Toda a configuração vem de variáveis de ambiente, com o valor padrão visível no
[`docker-compose.yml`](docker-compose.yml). Para mudar numa execução:
`CATALOG_SEATS=5 docker compose up -d --wait`.

**Catálogo padrão** — 3 voos × 7 dias, 20 assentos em cada par (voo, dia):

| Voo | Preço em 2026-12-01 | Acréscimo por dia | Preço em 2026-12-07 |
|---|---|---|---|
| IMD100 | US$ 350,50 | US$ 12,25 | US$ 424,00 |
| IMD200 | US$ 420,50 | US$ 12,25 | US$ 494,00 |
| IMD300 | US$ 510,50 | US$ 12,25 | US$ 584,00 |

Dias: `2026-12-01` a `2026-12-07`. O preço é determinístico: a mesma consulta dá sempre o
mesmo valor.

| Variável | Padrão | Serviço |
|---|---|---|
| `CATALOG_FLIGHTS` | `IMD100,IMD200,IMD300` | AirlinesHub |
| `CATALOG_BASE_PRICES` | `350.50,420.50,510.50` | AirlinesHub |
| `CATALOG_PRICE_STEP_PER_DAY` | `12.25` | AirlinesHub |
| `CATALOG_START_DATE` | `2026-12-01` | AirlinesHub |
| `CATALOG_DAYS` / `CATALOG_SEATS` | `7` / `20` | AirlinesHub |
| `SELL_MIN_MS` / `SELL_MAX_MS` | `300` / `900` | AirlinesHub (latência do `POST /sell`) |
| `EXCHANGE_INITIAL_RATE` | `5.50` | Exchange |
| `EXCHANGE_TICK_S` / `EXCHANGE_MAX_STEP_PCT` | `10` / `0.3` | Exchange (a cada 10 s, até ±0,3%) |
| `EXCHANGE_MIN` / `EXCHANGE_MAX` | `5.00` / `6.00` | Exchange |
| `IMDTRAVEL_HOST_PORT` | `8080` | porta do IMDTravel no seu computador |

O `docker-compose.yml` também traz as variáveis `FAULT_*`, todas com padrão `0`: são a
injeção de falhas da Parte 2, descrita em [Injeção de falhas](FALHAS.md). Com os padrões,
não há falha nenhuma, como pede a Parte 1.

---

## 6. Onde fica cada coisa

| Caminho | Conteúdo |
|---|---|
| [`docker-compose.hub.yml`](docker-compose.hub.yml) | Sobe o sistema com as imagens do Docker Hub |
| [`testes.sh`](testes.sh) | Verificação funcional |
| [`cenario.sh`](cenario.sh) | Parte 2: observação de um cenário normal ou de uma das quatro falhas |
| [`docs/`](docs/) | Especificações e figuras |

---

## 7. Problemas comuns

| Sintoma | Causa provável e solução |
|---|---|
| `port is already allocated` na 8080 | Outro programa usa a porta. `IMDTRAVEL_HOST_PORT=9090 docker compose up -d --wait` e use `http://localhost:9090`. Rode o `testes.sh` com a mesma variável. |
| Um serviço fica em `(health: starting)` muito tempo | A primeira subida da JVM é lenta em máquinas com pouca memória. Aumente a memória do Docker Desktop. `docker compose logs <serviço>` mostra o erro, se houver. |
| `Cannot connect to the Docker daemon` | O Docker Desktop não está aberto (macOS/Windows) ou o serviço não está rodando (Linux: `sudo systemctl start docker`). |
| No Windows: `bad interpreter` ou `$'\r': command not found` | O script ficou com fim de linha do Windows (CRLF). O `.gitattributes` do repositório força LF nos clones novos; se o problema aparecer, o arquivo foi editado ou copiado por um editor do Windows. Conserte com `sed -i 's/\r$//' testes.sh` e configure o editor para salvar com LF. |
| O `testes.sh` diz que nenhum par tem assento livre | Muitas execuções esgotaram o catálogo. `./testes.sh --reiniciar`. |
| Uma compra demora minutos para dar erro | **Comportamento esperado na Parte 1.** O IMDTravel não tem timeout nas chamadas de saída; quando uma dependência some, a compra fica presa até o sistema operacional desistir. É exatamente o que o roteiro de observação mede. |

---
