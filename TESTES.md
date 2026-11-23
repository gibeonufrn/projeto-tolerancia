# IMDTravel — como testar

Três formas de conferir que o sistema faz o que a
[especificação da Parte 1](docs/parte1-especificacao.md) pede:

| Forma | Para quê | Tempo |
|---|---|---|
| [Testes manuais com `curl`](#1-testes-manuais-com-curl) | Entender cada serviço, uma requisição por vez | — |
| [`testes.sh`](#2-o-script-testessh) | Verificar o sistema inteiro, de fora, com um comando | ~10 s |
| [Testes unitários](#3-testes-unitários) | Verificar a lógica de cada serviço, sem subir containers | ~1 min |

Todas partem do sistema no ar. Para instalar o ambiente e subir os serviços, veja
[Como executar](EXECUCAO.md).

Estes são os testes da **etapa 1**, com as dependências sadias. Os testes com falhas
injetadas são da etapa 2 e estão em [Testes da etapa 2](TESTES-FALHAS.md).

---

## 1. Testes manuais com `curl`

Só o **IMDTravel** responde no seu computador, em `http://localhost:8080`. Os outros três
ficam na rede interna do Docker: para chamá-los, rode o `curl` **de dentro** do container
com `docker compose exec <serviço> curl ...`.

### Os quatro estão vivos?

```
curl -s localhost:8080/health
docker compose exec airlineshub curl -s localhost:8080/health
docker compose exec exchange    curl -s localhost:8080/health
docker compose exec fidelity    curl -s localhost:8080/health
```
Esperado: `{"status":"ok","uptimeS":...}` em cada um. Um `"status":"down"` vem com o motivo
em `reason`.

### Uma compra, do começo ao fim

1. **Estoque antes** (AirlinesHub):
   ```
   docker compose exec airlineshub curl -s "localhost:8080/flight?flight=IMD100&day=2026-12-01"
   ```
   `{"flight":"IMD100","day":"2026-12-01","priceUSD":350.50,"seatsAvailable":20}`

2. **A compra** (IMDTravel). O `-i` mostra também os cabeçalhos da resposta:
   ```
   curl -i -H 'Content-Type: application/json' \
        -H 'X-Correlation-Id: meu-teste-1' \
        -d '{"flight":"IMD100","day":"2026-12-01","user":"ana"}' \
        localhost:8080/buyTicket
   ```
   Esperado: `201`, o cabeçalho `X-Correlation-Id: meu-teste-1` e um corpo com
   `transactionId`, `flight`, `day`, `user`, `valueUSD`, `valueBRL`, `rate`, `rateAsOf` e `bonus`.
   Confira que `valueBRL` ≈ `valueUSD × rate` e que `bonus` é a parte inteira de `valueUSD`.

3. **Estoque depois**: repita o passo 1; `seatsAvailable` deve ter baixado **exatamente 1**.

4. **Registro de vendas** (IMDTravel): a compra aparece com o seu `correlationId`:
   ```
   curl -s localhost:8080/sales | jq '.[] | select(.correlationId=="meu-teste-1")'
   ```

5. **Pontos** (Fidelity): o bônus foi creditado, e a entrada traz o mesmo
   `X-Correlation-Id`, prova de que ele foi propagado até a quarta chamada:
   ```
   docker compose exec fidelity curl -s "localhost:8080/points?user=ana"
   ```

6. **Os logs** dos quatro serviços: cada linha é um objeto JSON com `ts`, `svc`, `cid`,
   `event` e `ms`. Filtrando pelo cid, aparecem as cinco requisições da compra, cada uma
   com `requisicao.recebida` e `resposta.enviada` (esta com o `status` e a duração em `ms`):
   ```
   docker compose logs --no-log-prefix | grep meu-teste-1 | jq -c '{svc,event,ms,path,status}'
   ```
   O `/health` só aparece no log quando a resposta não é 200, para o healthcheck do
   compose não afogar o resto.

### A taxa de câmbio

```
docker compose exec exchange curl -s localhost:8080/convert
```
Duas chamadas seguidas devolvem o mesmo `rate` e o mesmo `asOf`. Espere 10 s e chame de
novo: a taxa mudou um pouco (no máximo ±0,3%) e o `asOf` avançou.

### Os erros que o IMDTravel devolve

| Pedido | Esperado |
|---|---|
| `{"flight":"XXX999","day":"2026-12-01","user":"ana"}` (voo inexistente) | `404` `{"error":"not-found"}` |
| `{"flight":"IMD100","user":"ana"}` (sem `day`) | `400` `{"error":"invalid-request"}` |
| `{malformado` | `400` |
| Um par (voo, dia) já esgotado | `409` `{"error":"sold-out"}` |

Nenhum deles consome assento nem cria linha em `/sales`.

Para ver o **409**, esgote um par com 25 compras simultâneas (o par tem 20 assentos):
```
seq 25 | xargs -P 25 -I{} curl -s -o /dev/null -w '%{http_code}\n' \
    -H 'Content-Type: application/json' \
    -d '{"flight":"IMD300","day":"2026-12-07","user":"u{}"}' \
    localhost:8080/buyTicket | sort | uniq -c
```
Esperado: `20 201` e `5 409` — nunca 21 vendas, nunca estoque negativo.

### Uma dependência fora do ar

```
docker compose stop fidelity
curl -s -w ' [%{http_code}] %{time_total}s\n' -H 'Content-Type: application/json' \
     -d '{"flight":"IMD200","day":"2026-12-01","user":"bia"}' localhost:8080/buyTicket
docker compose start fidelity
```
Esperado: **500** — e o assento de IMD200/2026-12-01 foi consumido mesmo assim, sem linha
nova em `/sales`. A resposta **pode levar de segundos a mais de dois minutos**: na Parte 1
o IMDTravel não tem timeout nas chamadas de saída. Isso não é defeito; é o que o roteiro de
observação mede.

---

## 2. O script `testes.sh`

Faz, automaticamente e com conferência, tudo o que a seção anterior faz à mão.

### Como rodar

Na raiz do repositório, com o sistema no ar:

```
./testes.sh                  # testa o sistema que já está no ar
./testes.sh --com-falhas     # inclui o teste com o Fidelity parado (pode levar minutos)
./testes.sh --reiniciar      # derruba tudo, APAGA o volume e sobe do zero antes de testar
./testes.sh --help           # mostra as opções
```

As opções podem ser combinadas: `./testes.sh --reiniciar --com-falhas`.

> **`--reiniciar` apaga o registro de vendas** (`docker compose down -v`) e reconstrói as
> imagens a partir do código. Use quando quiser um ponto de partida limpo.

**Requisitos:** `docker compose`, `curl` e `jq` (veja [o que instalar](EXECUCAO.md#1-o-que-instalar)).
Roda no bash do macOS, do Linux e do WSL no Windows.

### O que ele verifica

| Seção | Verificações |
|---|---|
| **1. `/health`** | Os quatro serviços respondem `status: ok` com `uptimeS`. |
| **2. Uma compra** | 201; os 9 campos presentes; `valueBRL = valueUSD × rate`; bônus = parte inteira do preço; taxa entre 5,00 e 6,00; `X-Correlation-Id` devolvido; estoque −1; exatamente uma linha nova em `/sales`, com cid, `transactionId`, par, usuário, valores e instante; bônus creditado no Fidelity com o mesmo cid. |
| **3. cid gerado** | Sem `X-Correlation-Id` no pedido, o IMDTravel gera um, devolve no cabeçalho e o grava em `/sales`. |
| **4. Determinismo** | O mesmo par tem sempre o mesmo preço, e a compra usou esse preço; duas leituras com o mesmo `asOf` têm a mesma taxa. |
| **5. Pedidos inválidos** | Voo inexistente e dia fora do catálogo → 404; campo faltando, usuário vazio e JSON malformado → 400; nenhum deles mexe no estoque nem em `/sales`. |
| **6. Esgotamento** | 25 compras simultâneas para 20 assentos → exatamente 20 × 201 e 5 × 409, estoque 0, 20 linhas novas; mais uma compra → 409 `sold-out`, sem bônus. |
| **7. Falha** (só com `--com-falhas`) | Com o Fidelity parado, a compra devolve 500, **o assento é consumido** e não entra linha em `/sales`. O script mostra quanto tempo a compra levou e religa o Fidelity no fim. |

### Como ler o resultado

```
== 2. Uma compra bem-sucedida
  par: IMD100 2026-12-01   usuário: r1794521808-ana   cid: r1794521808-cid-1
  ok status 201
  ok resposta tem os 9 campos, nenhum nulo
  ...
== Resumo: 40 passaram, 0 falharam
```

- Cada linha `ok` é uma verificação que passou.
- Uma linha `FALHOU` mostra o **valor obtido** e o **esperado**; quando a verificação é sobre
  um JSON, mostra também o JSON recebido:
  ```
  FALHOU corpo do 404 é { error: not-found } — obtido: false, esperado: true
         recebido: {"timestamp":"...","status":400,"error":"Bad Request",...}
  ```
- O código de saída é **0** se tudo passou e **1** se alguma verificação falhou (útil em
  scripts e em CI: `./testes.sh && echo tudo certo`). Os códigos **2** indicam erro de uso:
  opção desconhecida ou falta `docker`, `curl` ou `jq`.

### Rodar de novo, sem limpar

O script **não precisa de um sistema limpo**. Ele escolhe pares (voo, dia) com assentos
livres, confere diferenças (antes × depois) em vez de valores absolutos, e usa usuários com
um prefixo próprio a cada execução. Dá para rodar várias vezes seguidas.

O limite: **cada execução esgota um par inteiro** (seção 6). O catálogo padrão tem 21 pares,
então depois de umas vinte execuções não sobra par intacto e a seção 6 é pulada com o aviso
`nenhum par com os 20 assentos intactos — pulado`. Rode `./testes.sh --reiniciar`.

### Variáveis de ambiente

| Variável | Para quê | Exemplo |
|---|---|---|
| `IMDTRAVEL_HOST_PORT` | O IMDTravel está noutra porta | `IMDTRAVEL_HOST_PORT=9090 ./testes.sh` |
| `IMDTRAVEL` | O IMDTravel está noutro endereço | `IMDTRAVEL=http://192.168.0.10:8080 ./testes.sh` |
| `COMPOSE_FILE` | O sistema foi subido com outro arquivo de compose | `COMPOSE_FILE=docker-compose.hub.yml ./testes.sh` |

**Com as imagens do Docker Hub:** numa pasta que tem só o `docker-compose.hub.yml` (sem
`docker-compose.yml`), o script usa esse arquivo sozinho: basta `./testes.sh`. Num clone do
repositório, onde os dois arquivos existem, indique o do Hub com
`COMPOSE_FILE=docker-compose.hub.yml ./testes.sh`. Assim o `--reiniciar` também usa as
imagens do Hub em vez de reconstruir a partir do código.

**Rode o script na pasta em que o sistema foi subido.** O IMDTravel é acessado pela porta,
mas os outros três são acessados por `docker compose exec`, que só enxerga os containers
do projeto compose da pasta atual. Se o sistema no ar for de outra pasta, o script para
antes de testar e diz qual projeto está no ar e onde ele foi subido.

### Problemas comuns

| Sintoma | Solução |
|---|---|
| `Serviços do projeto compose '...' que não estão no ar` | O sistema no ar foi subido noutra pasta ou com outro arquivo de compose; a mensagem diz qual. Rode o script naquela pasta, ou derrube aquele sistema e suba este. |
| `IMDTravel não responde em http://localhost:8080/health` | O sistema não está no ar: `docker compose up -d --wait`, ou rode com `--reiniciar`. |
| `Permission denied` | `chmod +x testes.sh` |
| `falta o comando: jq` | Instale o `jq` (veja [o que instalar](EXECUCAO.md#1-o-que-instalar)). |
| No Windows: `bad interpreter` ou `$'\r': command not found` | O arquivo ficou com fim de linha CRLF: `sed -i 's/\r$//' testes.sh`. |
| `nenhum par com assento livre — rode com --reiniciar` | O catálogo inteiro foi vendido: `./testes.sh --reiniciar`. |
| A seção 7 parece travada | É esperado: com o Fidelity parado, a compra pode levar mais de dois minutos para dar 500 (não há timeout na Parte 1). O script espera até 10 minutos. |

---

## 3. Testes unitários

Rodam sem subir containers. O projeto usa Java 21; se a sua máquina tiver outra versão,
rode dentro de um container Maven:

```
docker run --rm -v "$PWD":/src -v imdtravel-m2:/root/.m2 -w /src \
       maven:3.9-eclipse-temurin-21 mvn -B test
```

Com Java 21 instalado, basta `mvn test`. A mesma execução roda também os testes da
injeção de falhas, descritos em [Testes da etapa 2](TESTES-FALHAS.md#3-testes-unitários).

---
