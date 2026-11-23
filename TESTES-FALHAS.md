# IMDTravel — Etapa 2: testes com injeção de falhas

Como observar cada falha do [modelo de falhas](FALHAS.md) e conferir que ela faz o que
promete. Os testes da etapa 1 (`curl`, `testes.sh`, testes unitários) estão em
[Como testar](TESTES.md) e continuam valendo: sem nenhuma variável `FAULT_*`, o sistema é o
da etapa 1 e o `./testes.sh` passa inteiro.

| Forma | Para quê | Tempo |
|---|---|---|
| [`cenario.sh`](#1-o-script-cenariosh) | Ver o estrago de cada falha, comparado com o cenário normal | ~30 s por cenário |
| [Testes manuais](#2-testes-manuais) | Ligar uma falha à mão e olhar uma requisição por vez | — |

**Requisitos:** os mesmos da etapa 1: `docker compose`, `curl` e `jq` (veja
[o que instalar](EXECUCAO.md#1-o-que-instalar)).

---

## 1. O script `cenario.sh`

Liga uma falha, faz algumas compras e mostra o que aconteceu com elas e com o estado dos
serviços. Rode primeiro o cenário normal, que serve de comparação.

```
./cenario.sh normal                  # sem falha: a linha de comparação
./cenario.sh omissao                 # ou tempo | resposta | crash, com p e ttlSec padrão
./cenario.sh crash 0.5 10 8          # p, ttlSec e número de compras
INTERVALO=2 ./cenario.sh tempo       # segundos entre uma compra e a seguinte (padrão 1)
MANTER=1 ./cenario.sh tempo          # deixa a falha ligada no fim
./cenario.sh --help
```

Para cada cenário, o script:

1. recria os três serviços de dependência (estoque cheio) com só a falha pedida ligada e
   reinicia o IMDTravel;
2. dispara as compras, todas para o mesmo (voo, dia) e um usuário novo, sem esperar a
   anterior terminar, para que várias caiam na mesma janela;
3. mostra o status, o tempo e o valor ou o erro de cada compra;
4. faz o balanço (assentos consumidos × vendas em `/sales` × bônus creditados) e aponta
   cada divergência;
5. reconstrói pelo log, via `X-Correlation-Id`, o caminho da primeira compra e lista os
   eventos da falha;
6. desliga a falha e deixa o sistema como na etapa 1.

### O que esperar

Com os padrões (5 compras, 1 s de intervalo):

| Cenário | Padrão (p, ttlSec) | Status | Assentos / vendas / bônus | O que se vê |
|---|---|---|---|---|
| `normal` | — | 5 × 201, em menos de 1 s | 5 / 5 / 5 | Tudo bate |
| `tempo` | 1, 3 | 5 × 201, as primeiras em ~3,8 s | 5 / 5 / 5 | Correto, mas lento; as compras dividem a mesma janela |
| `omissao` | 1, 5 | 5 × 500, em ~5 s | **5 / 0 / 0** | Assentos vendidos sem venda registrada |
| `resposta` | 1, 0 | 201 ou 500 | 5 / 5 / 5 nos 201 | **Vendas com valor errado aceitas** (R$ 0, negativo, bilhões); taxa não numérica dá 500 sem vender |
| `crash` | 1, 10 | 5 × 500, em menos de 1 s | **5 / 0 / 0** | Assentos vendidos sem bônus; o script espera o Fidelity voltar para contar os pontos |

Os números da `resposta` variam de uma execução para outra: a variante do `rate` errado é
sorteada a cada requisição.

O curl das compras tem `--max-time 300` só para o script não ficar preso para sempre. O
IMDTravel continua sem timeout nenhum.

### Problemas comuns

| Sintoma | Solução |
|---|---|
| O script termina com a mensagem de uma variável `FAULT_*` | Configuração inválida, por exemplo `omissao 1 0`: Tempo, Omissão e Crash exigem `ttlSec > 0`. O script restaura os padrões antes de sair. |
| Uma compra volta 404 logo depois de uma recriação feita à mão | Cache de DNS da JVM: reinicie o IMDTravel (`docker compose restart imdtravel`). O script já faz isso. |
| Rodar junto com o `testes.sh` dá resultados estranhos | O `cenario.sh` recria containers e zera o estoque. Rode um de cada vez. |