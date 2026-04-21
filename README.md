<div align="center">

# ⛏️ Bitcoin Corders Bootcamp

**Ambiente de desenvolvimento Bitcoin Core em `regtest` via Docker**

Um setup mínimo e reprodutível pra aprender, testar e prototipar em cima do protocolo Bitcoin sem depender da rede real, sem sincronizar blocos, sem taxa e sem espera.

![Bitcoin Core](https://img.shields.io/badge/Bitcoin_Core-28.0-F7931A?logo=bitcoin&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![Network](https://img.shields.io/badge/network-regtest-8A2BE2)

</div>

---

## 📚 Sumário

- [Sobre](#-sobre)
- [Pré-requisitos](#-pré-requisitos)
- [Estrutura do projeto](#-estrutura-do-projeto)
- [Início rápido](#-início-rápido)
- [Acessando o `bitcoin-cli`](#-acessando-o-bitcoin-cli)
- [Fluxo básico no regtest](#-fluxo-básico-no-regtest)
- [Experimentos](#-experimentos)
- [Comandos do dia a dia](#-comandos-do-dia-a-dia)
- [Portas e credenciais](#-portas-e-credenciais)
- [Persistência e reset](#-persistência-e-reset)
- [Troubleshooting](#-troubleshooting)
- [Referências](#-referências)

---

## 🧭 Sobre

Este repositório provisiona um **node Bitcoin Core isolado em modo regtest**, ideal pra:

- Estudar o protocolo Bitcoin sem risco (nada sai pra mainnet).
- Minerar blocos sob demanda (sem PoW real, um bloco por comando).
- Testar wallets, scripts, transações e aplicações que usam RPC.
- Rodar testes determinísticos em CI.

O node fica dentro de um container, os dados persistem num volume nomeado, e o `bitcoin-cli` é usado direto pelo host via `docker compose exec`.

---

## 🧰 Pré-requisitos

| Ferramenta | Versão mínima |
|---|---|
| Docker Engine | 20.10+ |
| Docker Compose | v2+ |

> Verifique com `docker --version` e `docker compose version`.

---

## 🗂️ Estrutura do projeto

```text
.
├── docker-compose.yml        # serviço bitcoind em regtest
├── bitcoind/
│   └── bitcoin.conf          # config do node (RPC, txindex, portas)
└── README.md
```

- **`bitcoin.conf`** — fonte da verdade do node: porta RPC, credenciais, `txindex`, `fallbackfee`.
- **Volume `bitcoind-data`** — persiste blockchain, wallets e chainstate entre restarts.

---

## 🚀 Início rápido

```bash
# 1. subir o node em background
docker compose up -d

# 2. verificar que está rodando
docker compose ps

# 3. acompanhar logs
docker compose logs -f bitcoind

# 4. parar (preserva dados)
docker compose down

# 5. parar e zerar tudo
docker compose down -v
```

Na primeira vez, o Docker baixa a imagem `bitcoin/bitcoin:28.0` (~80 MB) e inicializa o `datadir`.

### Smoke test

Valide que o node está respondendo:

```bash
docker compose exec --user bitcoin bitcoind bitcoin-cli -regtest getblockchaininfo
```

Saída esperada num node recém-criado:

```json
{
  "chain": "regtest",
  "blocks": 0,
  "headers": 0,
  "bestblockhash": "0f9188f13cb7b2c71f2a335e3a4fc328bf5beb436012afca590b1a11466e2206",
  "initialblockdownload": true,
  "size_on_disk": 293,
  "warnings": []
}
```

O `bestblockhash` mostrado é o **genesis block do regtest** — é sempre o mesmo, então serve como sanity check.

---

## 💻 Acessando o `bitcoin-cli`

O CLI roda **dentro do container**, então você não precisa instalar nada no host.

### Forma direta

```bash
docker compose exec --user bitcoin bitcoind bitcoin-cli -regtest getblockchaininfo
```

> A flag `--user bitcoin` é obrigatória: sem ela o exec entra como `root` e o `bitcoin-cli` procura a config em `/root/.bitcoin/` em vez de `/home/bitcoin/.bitcoin/`, resultando em `Could not locate RPC credentials`.

### Alias recomendado ✨

Adicione ao seu `~/.bashrc` ou `~/.zshrc`:

```bash
alias bitcoin-cli='docker compose -f ~/Work/bitcoin-corders-bootcamp/docker-compose.yml exec -T --user bitcoin bitcoind bitcoin-cli'
```

Recarregue:

```bash
source ~/.bashrc
```

Agora o uso é transparente, como se fosse local:

```bash
bitcoin-cli -regtest getblockchaininfo
```

> **Dica:** a flag `-T` desliga o TTY — importante pra pipes (`| jq`, `| grep`) e scripts.

---

## 🧪 Fluxo básico no regtest

Do zero até uma transação confirmada:

```bash
# 1. criar uma wallet
bitcoin-cli -regtest createwallet dev

# 2. gerar um endereço pra receber
ADDR=$(bitcoin-cli -regtest getnewaddress)
echo "Meu endereço: $ADDR"

# 3. minerar 101 blocos pra ter saldo gastável
#    (a recompensa coinbase só matura após 100 confirmações)
bitcoin-cli -regtest generatetoaddress 101 "$ADDR"

# 4. checar saldo
bitcoin-cli -regtest getbalance

# 5. enviar pra outro endereço
DEST=$(bitcoin-cli -regtest getnewaddress)
TXID=$(bitcoin-cli -regtest sendtoaddress "$DEST" 1.5)
echo "TXID: $TXID"

# 6. confirmar minerando 6 blocos
bitcoin-cli -regtest generatetoaddress 6 "$ADDR"

# 7. inspecionar a tx
bitcoin-cli -regtest gettransaction "$TXID"
```

---

## 🧬 Experimentos

Pequenos laboratórios pra ganhar intuição sobre como o protocolo funciona. Cada um é independente e pode rodar no mesmo node.

### 1. Transferência entre duas wallets

Simula dois "usuários" no mesmo node — útil pra entender como wallets isoladas vêem a mesma blockchain.

```bash
# criar uma segunda wallet
bitcoin-cli -regtest createwallet alice

# gerar endereço da alice (precisa especificar a wallet)
ALICE=$(bitcoin-cli -regtest -rpcwallet=alice getnewaddress)

# enviar da wallet 'dev' pra alice
bitcoin-cli -regtest -rpcwallet=dev sendtoaddress "$ALICE" 1.5

# tx está na mempool (ainda não confirmada)
bitcoin-cli -regtest getrawmempool

# minerar 1 bloco pra confirmar
bitcoin-cli -regtest -generate 1

# saldos das duas wallets
bitcoin-cli -regtest -rpcwallet=dev getbalance
bitcoin-cli -regtest -rpcwallet=alice getbalance
```

### 2. Inspecionar uma transação raw

Mostra os bytes reais da tx — inputs, outputs, scripts.

```bash
TXID=<txid_da_tx_anterior>

# versão decodificada (JSON)
bitcoin-cli -regtest getrawtransaction "$TXID" true

# versão crua (hex) — é isso que vai na wire
bitcoin-cli -regtest getrawtransaction "$TXID"
```

Repare no campo `vin[].scriptSig` (desbloqueio) e `vout[].scriptPubKey` (trava) — o coração do modelo UTXO.

### 3. Mempool e confirmações

```bash
# criar várias txs sem minerar
for i in 1 2 3; do
  bitcoin-cli -regtest -rpcwallet=dev sendtoaddress "$ALICE" 0.1
done

# ver a fila na mempool
bitcoin-cli -regtest getmempoolinfo
bitcoin-cli -regtest getrawmempool

# confirmar todas em um bloco só
bitcoin-cli -regtest -generate 1
bitcoin-cli -regtest getmempoolinfo   # mempool vazia
```

### 4. Fee e taxa customizada

```bash
# estimar fee (em regtest retorna valores mínimos)
bitcoin-cli -regtest estimatesmartfee 6

# enviar com fee rate explícito (sat/vB)
bitcoin-cli -regtest -rpcwallet=dev -named sendtoaddress \
  address="$ALICE" amount=0.5 fee_rate=10
```

### 5. Reorg — reescrever a história

Exclusivo do regtest: força uma reorganização e veja a chain "mudar de opinião".

```bash
# snapshot inicial
bitcoin-cli -regtest getblockcount
bitcoin-cli -regtest getbestblockhash

# invalida o último bloco (ele volta pra mempool)
TIP=$(bitcoin-cli -regtest getbestblockhash)
bitcoin-cli -regtest invalidateblock "$TIP"

# minera outro no lugar
bitcoin-cli -regtest -generate 1
bitcoin-cli -regtest getbestblockhash   # hash diferente

# reabilita o bloco anterior (se ficar mais longo, vence)
bitcoin-cli -regtest reconsiderblock "$TIP"
```

### 6. PSBT — Partially Signed Bitcoin Transaction

Fluxo usado em wallets multisig e hardware wallets.

```bash
ADDR=$(bitcoin-cli -regtest -rpcwallet=dev getnewaddress)

# cria um PSBT enviando 0.1 BTC pra ADDR
PSBT=$(bitcoin-cli -regtest -rpcwallet=dev walletcreatefundedpsbt \
  "[]" "[{\"$ADDR\":0.1}]" | jq -r .psbt)

# inspeciona
bitcoin-cli -regtest decodepsbt "$PSBT"

# assina
SIGNED=$(bitcoin-cli -regtest -rpcwallet=dev walletprocesspsbt "$PSBT" | jq -r .psbt)

# finaliza e extrai a tx pronta
HEX=$(bitcoin-cli -regtest finalizepsbt "$SIGNED" | jq -r .hex)

# broadcast
bitcoin-cli -regtest sendrawtransaction "$HEX"
```

> Requer `jq` instalado no host (`sudo pacman -S jq`).

---

## 📋 Comandos do dia a dia

### Blockchain

```bash
bitcoin-cli -regtest getblockchaininfo      # altura, chain, estado
bitcoin-cli -regtest getblockcount          # só altura
bitcoin-cli -regtest getbestblockhash       # hash do tip
bitcoin-cli -regtest getblock <hash>        # detalhes de um bloco
```

### Wallet

```bash
bitcoin-cli -regtest listwallets
bitcoin-cli -regtest createwallet <name>
bitcoin-cli -regtest loadwallet <name>
bitcoin-cli -regtest getwalletinfo
bitcoin-cli -regtest getnewaddress
bitcoin-cli -regtest getbalance
bitcoin-cli -regtest listunspent            # UTXOs
```

### Transações

```bash
bitcoin-cli -regtest sendtoaddress <addr> <btc>
bitcoin-cli -regtest gettransaction <txid>
bitcoin-cli -regtest getrawtransaction <txid> true
bitcoin-cli -regtest decoderawtransaction <hex>
bitcoin-cli -regtest listtransactions
```

### Mineração (regtest)

```bash
bitcoin-cli -regtest generatetoaddress 1 <addr>   # 1 bloco pro endereço
bitcoin-cli -regtest -generate 10                 # 10 blocos (usa wallet atual)
```

### Ajuda

```bash
bitcoin-cli -regtest help                   # lista todos os comandos
bitcoin-cli -regtest help sendtoaddress     # detalhes de um comando
```

---

## 🔌 Portas e credenciais

| Porta | Protocolo | Uso |
|-------|-----------|-----|
| `18443` | HTTP | JSON-RPC (regtest) |
| `18444` | TCP | P2P (regtest) |

Credenciais RPC definidas em `bitcoind/bitcoin.conf`:

```ini
rpcuser=bitcoin
rpcpassword=bitcoin
```

> ⚠️ Essas credenciais são **apenas pra dev local**. Nunca exponha na internet — em regtest já é ruim, em mainnet seria catastrófico.

### Exemplo: chamada RPC direta via `curl`

```bash
curl --user bitcoin:bitcoin \
  --data-binary '{"jsonrpc":"1.0","method":"getblockchaininfo","params":[]}' \
  -H 'content-type: text/plain;' \
  http://127.0.0.1:18443/
```

---

## 💾 Persistência e reset

Os dados (blockchain + wallets) vivem no volume nomeado `bitcoind-data`, então sobrevivem a `docker compose down` e `up` normais.

**Resetar tudo** (apaga volume, começa do bloco 0 e sem wallets):

```bash
docker compose down -v
docker compose up -d
```

**Listar volumes** do projeto:

```bash
docker volume ls | grep bitcoind
```

---

## 🛠️ Troubleshooting

<details>
<summary><b>O container reinicia em loop</b></summary>

Confira os logs:

```bash
docker compose logs bitcoind
```

Causas comuns: erro de sintaxe no `bitcoin.conf`, flag inválida no `command:` do compose, ou conflito de porta (18443/18444 já em uso no host).

</details>

<details>
<summary><b><code>chown: ... bitcoin.conf: Read-only file system</code></b></summary>

O entrypoint da imagem faz `chown` em `/home/bitcoin/.bitcoin/*` antes de iniciar o `bitcoind`. Se o bind mount do `bitcoin.conf` estiver como `:ro`, o chown falha e o container reinicia em loop.

**Fix:** no `docker-compose.yml`, remova o `:ro` do mount:

```yaml
volumes:
  - ./bitcoind/bitcoin.conf:/home/bitcoin/.bitcoin/bitcoin.conf   # sem :ro
```

Depois `docker compose down && docker compose up -d`.

</details>

<details>
<summary><b><code>Could not locate RPC credentials ... /root/.bitcoin/bitcoin.conf</code></b></summary>

Por padrão, `docker compose exec` entra no container como `root` — mas o `bitcoin.conf` está em `/home/bitcoin/.bitcoin/`, não em `/root/.bitcoin/`.

**Fix:** rode o exec como user `bitcoin`:

```bash
docker compose exec --user bitcoin bitcoind bitcoin-cli -regtest getblockchaininfo
```

Ou use o alias da seção [Acessando o `bitcoin-cli`](#-acessando-o-bitcoin-cli), que já inclui `--user bitcoin`.

</details>

<details>
<summary><b>`Loading block index...` demora muito</b></summary>

No regtest isso é instantâneo. Se travar, provavelmente o volume ficou corrompido — resete com `docker compose down -v`.

</details>

<details>
<summary><b>`error code: -18 Requested wallet does not exist`</b></summary>

Nenhuma wallet está carregada. Rode:

```bash
bitcoin-cli -regtest createwallet dev
# ou, se já existe:
bitcoin-cli -regtest loadwallet dev
```

</details>

<details>
<summary><b>`Insufficient funds` logo depois de minerar</b></summary>

A recompensa coinbase só matura após **100 confirmações**. Minere `101` blocos antes de gastar:

```bash
bitcoin-cli -regtest -generate 101
```

</details>

<details>
<summary><b>Porta 18443 ou 18444 já em uso</b></summary>

Outro processo está ocupando a porta. Descubra e encerre:

```bash
ss -lntp | grep -E '18443|18444'
```

Ou remapeie no `docker-compose.yml` (ex: `"28443:18443"`).

</details>

---

## 🔗 Referências

- [Bitcoin Coders — Curso Bitcoin CLI](https://bitcoincoders.org/curso/bitcoin-cli/) — curso-base deste bootcamp
- [Bitcoin Core — Documentação oficial](https://bitcoincore.org/en/doc/)
- [RPC API reference (28.0)](https://developer.bitcoin.org/reference/rpc/)
- [Learning Bitcoin from the Command Line](https://github.com/ChristopherA/Learning-Bitcoin-from-the-Command-Line)
- [Imagem Docker oficial — `bitcoin/bitcoin`](https://hub.docker.com/r/bitcoin/bitcoin)
