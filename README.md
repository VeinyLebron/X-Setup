# X-Setup

Ready-to-run local [Hyperledger Fabric-X](https://github.com/hyperledger/fabric-x-samples) networks for the [X-Client](https://github.com/VeinyLebron/X-Client) wholesale CBDC sample. Pick a setup, run `make up`, then start the matching X-Client.

| Setup | Organizations | Folder | X-Client branch |
|---|---|---|---|
| **3 organizations** | Central Bank, Bank A (Org1), Bank B (Org2) | [`3-org/`](3-org/) | `main` |
| **2 organizations** | Org1 (issuer + bank), Org2 (bank) | [`2-org/`](2-org/) | `two-org` |

Run one setup at a time.

## Prerequisites

| Need | Check | Notes |
|---|---|---|
| Linux or macOS | | Windows: use WSL2 |
| Docker with Compose v2 | `docker compose version` | |
| Docker without `sudo` | `docker ps` works | Linux: `sudo usermod -aG docker $USER`, then **log out and back in** |
| `git`, `make` | `git --version`, `make --version` | |
| ~4 GB free RAM, ~5 GB disk | | the 3-org setup runs 35 containers |
| Free ports | `7050 7150 7250 7350`, `7001`–`7003`, `4001`–`4003`, `8080`–`8082` | |
| Go 1.27.3+ | `go version` | only to run X-Client outside Docker |

## Quick start: 3 organizations

```bash
mkdir fabric-x && cd fabric-x                      # both repos must sit side by side
git clone https://github.com/VeinyLebron/X-Setup.git
git clone https://github.com/VeinyLebron/X-Client.git

cd X-Setup/3-org
make up                                            # crypto, network, namespaces (a few minutes the first time)

cd ../../X-Client
./scripts/copy-crypto.sh ../X-Setup/3-org          # give X-Client its keys and certificates
docker compose -f docker-compose.3org.yaml up -d --build
```

Check it works:
```bash
curl localhost:8080/healthz      # Central Bank: "ready":true
curl -X POST localhost:8080/api/v1/wallets -H 'Content-Type: application/json' \
  -d '{"walletId":"WA","ownerMsp":"Org1MSP","participantType":"bank"}'
# → "status":"COMMITTED"
```
X-Clients: Central Bank `:8080`, Bank A `:8081`, Bank B `:8082`. Next steps (minting, transfers, the API): X-Client's `SETUP.md` and `docs/API.md`.

## Quick start: 2 organizations

```bash
mkdir fabric-x && cd fabric-x
git clone https://github.com/VeinyLebron/X-Setup.git
git clone -b two-org https://github.com/VeinyLebron/X-Client.git

cd X-Setup/2-org
make up

cd ../../X-Client
./scripts/copy-crypto.sh ../X-Setup/2-org
docker compose -f docker-compose.2org.yaml up -d --build
```

Check it works:
```bash
curl localhost:8080/healthz      # Org1: "ready":true
curl -X POST localhost:8080/api/v1/wallets -H 'Content-Type: application/json' \
  -d '{"walletId":"WA","ownerMsp":"Org1MSP","participantType":"bank"}'
```
X-Clients: Org1 `:8080`, Org2 `:8081`.

## What `make up` does

1. Generates crypto material and the genesis block into `crypto/` (first run only).
2. Starts the 4-party Arma orderer and one committer stack per organization, and waits until every container is healthy.
3. Registers the five CBDC namespaces with the endorsement policies that the matching X-Client branch expects (listed in each folder's README).

It is safe to run again: it resumes a stopped network and skips namespaces that already exist.

## Day-to-day commands (inside `2-org/` or `3-org/`)

| Command | What it does |
|---|---|
| `make up` | start (or resume) the network |
| `make status` | running containers and registered namespaces |
| `make down` | stop, keep the ledger |
| `make reset` | stop and delete ledger and crypto. After the next `make up`, re-run X-Client's `copy-crypto.sh` |

Stop X-Client before the network: `docker compose -f docker-compose.3org.yaml down` (or `.2org.yaml`) in the X-Client folder.

## Switching between setups

```bash
cd X-Client && docker compose -f docker-compose.3org.yaml down
cd ../X-Setup/3-org && make down
cd ../2-org && make up
cd ../../X-Client && git checkout two-org && ./scripts/copy-crypto.sh ../X-Setup/2-org
docker compose -f docker-compose.2org.yaml up -d --build
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| `permission denied … /var/run/docker.sock` | add yourself to the `docker` group and log out and back in |
| `Error: another Fabric-X network is already running` | run the `make down` command it prints |
| `port … is already in use` / `address already in use` | something else uses that port: `ss -ltnp \| grep <port>` |
| `make up` times out waiting for health | `docker ps -a` to find the failing container, `docker logs <name>`. Then `make reset && make up` |
| X-Client: `failed to read CA cert` / `no such file` | run `./scripts/copy-crypto.sh ../X-Setup/<setup>` in X-Client |
| X-Client: `private key does not match certificate` or `ABORTED_SIGNATURE_INVALID` after `make reset` | crypto was regenerated: re-run `copy-crypto.sh`, then restart X-Client with `--build` |
| X-Client `"ready": false` | it is still reading the ledger, or the network is down: `make status` |
| `namespace … is not registered` | `make up` (registers missing namespaces) |

## Licence

MIT for this repository (`LICENSE`). The network folders are derived from hyperledger/fabric-x-samples and stay under Apache-2.0. See `NOTICE`.
