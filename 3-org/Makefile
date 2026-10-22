# Resolve this Makefile's own directory so paths work when called via make -C devnet.
FABRICX_DIR := $(patsubst %/,%,$(dir $(abspath $(lastword $(MAKEFILE_LIST)))))

# Container images
TOOLS_IMAGE          ?= ghcr.io/hyperledger/fabric-x-tools:1.0.1
ORDERER_IMAGE        ?= ghcr.io/hyperledger/fabric-x-orderer:1.0.6

UID := $(shell id -u)
GID := $(shell id -g)
export UID
export GID

DOCKER          ?= docker
COMPOSE         ?= docker compose
RUN_AS          ?= --user "$(UID):$(GID)"
COMPOSE_OVERRIDE ?=

# Namespace init defaults — override on the command line or per-target below
NS      ?= mynamespace
POLICY  ?= AND('Org1MSP.member','Org2MSP.member')
NETWORK ?= fabric-x

# Port probe that needs only bash (netcat is not installed everywhere).
port_open = bash -c 'exec 3<>/dev/tcp/127.0.0.1/$(1)' 2>/dev/null

# Containers started from this folder (Compose labels them with their project directory).
OWN_CONTAINERS = $(DOCKER) ps --filter label=com.docker.compose.project.working_dir=$(FABRICX_DIR)

.DEFAULT_GOAL := help

# ─── Init ───────────────────────────────────────────────────────────────────

.PHONY: init
init:
	@rm -rf $(FABRICX_DIR)/crypto
	@$(DOCKER) run --rm \
		$(RUN_AS) \
		-v "$(FABRICX_DIR):/config" \
		$(TOOLS_IMAGE) \
		cryptogen generate --config=/config/crypto-config.yaml --output=/config/crypto
	@$(DOCKER) run --rm \
		$(RUN_AS) \
		-v "$(FABRICX_DIR):/config" \
		-v "$(FABRICX_DIR)/crypto:/crypto" \
		--entrypoint /usr/local/bin/armageddon \
		$(ORDERER_IMAGE) \
		createSharedConfigProto \
		--sharedConfigYaml=/config/shared_config.yaml \
		--output=/config/crypto/
	@$(DOCKER) run --rm \
		$(RUN_AS) \
		-v "$(FABRICX_DIR):/config" \
		$(TOOLS_IMAGE) \
		configtxgen --channelID mychannel --profile OrgsChannel \
		--outputBlock /config/crypto/config-block.pb.bin \
		--configPath /config

.PHONY: clean
clean:
	@rm -rf $(FABRICX_DIR)/crypto

# ─── Full stack (4-party orderer + Org1 committer) ──────────────────────────

.PHONY: start
start:
	@if [ ! -d "$(FABRICX_DIR)/crypto" ]; then echo "Error: crypto directory not found — run 'make init' first."; exit 1; fi
	@if $(call port_open,7050); then echo "Error: port 7050 is already in use — stop any running Fabric orderer before starting."; exit 1; fi
	@for p in 1 2 3 4; do mkdir -p \
		$(FABRICX_DIR)/data/orderers/party$$p-router \
		$(FABRICX_DIR)/data/orderers/party$$p-batcher \
		$(FABRICX_DIR)/data/orderers/party$$p-consenter \
		$(FABRICX_DIR)/data/orderers/party$$p-assembler; done
	@mkdir -p $(FABRICX_DIR)/data/committer-org1/sidecar-ledger \
		$(FABRICX_DIR)/data/committer-org1/db
	@$(COMPOSE) -f $(FABRICX_DIR)/compose.yaml $(COMPOSE_OVERRIDE) up -d
	@echo "Waiting for orderer to be ready..."
	@while ! $(call port_open,7050); do sleep 1; done
	@echo "Waiting for committer to be ready..."
	@while ! $(call port_open,7001); do sleep 1; done

.PHONY: stop
stop: stop-explorer
	@$(COMPOSE) -f $(FABRICX_DIR)/compose.yaml $(COMPOSE_OVERRIDE) -f $(FABRICX_DIR)/compose.org2.yaml -f $(FABRICX_DIR)/compose.centralbank.yaml down

.PHONY: purge
purge: stop
	@# Postgres (UID 70 on Alpine) creates files the host user can't delete on Linux; use a root container.
	@$(DOCKER) run --rm -v "$(FABRICX_DIR)/data:/data" alpine rm -rf \
		/data/committer-org1/db /data/committer-org2/db /data/committer-cb/db /data/explorer/db 2>/dev/null || true
	@rm -rf $(FABRICX_DIR)/data

# ─── Both orgs (Org1 + Org2 committer stacks) ───────────────────────────────

.PHONY: start-both
start-both:
	@if [ ! -d "$(FABRICX_DIR)/crypto" ]; then echo "Error: crypto directory not found — run 'make init' first."; exit 1; fi
	@if $(call port_open,7050); then echo "Error: port 7050 is already in use — stop any running Fabric orderer before starting."; exit 1; fi
	@for p in 1 2 3 4; do mkdir -p \
		$(FABRICX_DIR)/data/orderers/party$$p-router \
		$(FABRICX_DIR)/data/orderers/party$$p-batcher \
		$(FABRICX_DIR)/data/orderers/party$$p-consenter \
		$(FABRICX_DIR)/data/orderers/party$$p-assembler; done
	@mkdir -p $(FABRICX_DIR)/data/committer-org1/sidecar-ledger \
		$(FABRICX_DIR)/data/committer-org1/db \
		$(FABRICX_DIR)/data/committer-org2/sidecar-ledger \
		$(FABRICX_DIR)/data/committer-org2/db
	@$(COMPOSE) -f $(FABRICX_DIR)/compose.yaml $(COMPOSE_OVERRIDE) -f $(FABRICX_DIR)/compose.org2.yaml up -d
	@echo "Waiting for orderer to be ready..."
	@while ! $(call port_open,7050); do sleep 1; done
	@echo "Waiting for committers to be ready..."
	@while ! $(call port_open,7001); do sleep 1; done
	@while ! $(call port_open,7002); do sleep 1; done

# ─── All three orgs (Org1 + Org2 + CentralBank committer stacks) ────────────

.PHONY: start-3org
start-3org:
	@if [ ! -d "$(FABRICX_DIR)/crypto" ]; then echo "Error: crypto directory not found — run 'make init' first."; exit 1; fi
	@if $(call port_open,7050); then echo "Error: port 7050 is already in use — stop any running Fabric orderer before starting."; exit 1; fi
	@for p in 1 2 3 4; do mkdir -p \
		$(FABRICX_DIR)/data/orderers/party$$p-router \
		$(FABRICX_DIR)/data/orderers/party$$p-batcher \
		$(FABRICX_DIR)/data/orderers/party$$p-consenter \
		$(FABRICX_DIR)/data/orderers/party$$p-assembler; done
	@for o in org1 org2 cb; do mkdir -p \
		$(FABRICX_DIR)/data/committer-$$o/sidecar-ledger \
		$(FABRICX_DIR)/data/committer-$$o/db; done
	@$(COMPOSE) -f $(FABRICX_DIR)/compose.yaml $(COMPOSE_OVERRIDE) -f $(FABRICX_DIR)/compose.org2.yaml -f $(FABRICX_DIR)/compose.centralbank.yaml up -d
	@echo "Waiting for orderer to be ready..."
	@while ! $(call port_open,7050); do sleep 1; done
	@echo "Waiting for committers to be ready..."
	@for p in 7001 7002 7003; do while ! $(call port_open,$$p); do sleep 1; done; done

# ─── Dev mode (all-in-one test committer, includes built-in orderer) ─────────

.PHONY: start-dev
start-dev:
	@if [ ! -d "$(FABRICX_DIR)/crypto" ]; then echo "Error: crypto directory not found — run 'make init' first."; exit 1; fi
	@if $(call port_open,7050); then echo "Error: port 7050 is already in use."; exit 1; fi
	@$(COMPOSE) -f $(FABRICX_DIR)/compose.test-committer.yaml up -d
	@echo "Waiting for test committer to be ready..."
	@while ! $(call port_open,7001); do sleep 1; done

.PHONY: stop-dev
stop-dev: stop-explorer
	@$(COMPOSE) -f $(FABRICX_DIR)/compose.test-committer.yaml down

.PHONY: purge-dev
purge-dev: stop-explorer
	@$(COMPOSE) -f $(FABRICX_DIR)/compose.test-committer.yaml down -v
	@$(DOCKER) run --rm -v "$(FABRICX_DIR)/data:/data" alpine rm -rf \
		/data/explorer/db 2>/dev/null || true

# ─── Block explorer ─────────────────────────────────────────────────────────
# Web UI for browsing blocks and transactions. Joins the external fabric-x
# network, so start a stack first. Open the UI at http://localhost:3000.

.PHONY: start-explorer
start-explorer:
	@if ! $(DOCKER) network inspect $(NETWORK) >/dev/null 2>&1; then echo "Error: network $(NETWORK) not found — run 'make start' first."; exit 1; fi
	@mkdir -p $(FABRICX_DIR)/data/explorer/db
	@$(COMPOSE) -f $(FABRICX_DIR)/compose.block-explorer.yaml up -d
	@echo "Waiting for block explorer UI to be ready..."
	@while ! $(call port_open,$${UI_PORT:-3000}); do sleep 1; done
	@echo "Block explorer available at http://localhost:$${UI_PORT:-3000}"

.PHONY: stop-explorer
stop-explorer:
	@$(COMPOSE) -f $(FABRICX_DIR)/compose.block-explorer.yaml down

# ─── Namespace init ─────────────────────────────────────────────────────────
# Run one of these after start or start-dev to create the application namespace.
# Works with both backends: the test-committer exposes the same service aliases
# as the full stack, so applications and fxconfig.yaml need no changes.

define run-fxconfig
	@$(DOCKER) run --rm --network $(NETWORK) \
		$(RUN_AS) \
		--env "FX_NS=$(NS)" \
		--env "FX_POLICY=$(POLICY)" \
		-v "$(FABRICX_DIR)/fxconfig.yaml:/config/fxconfig.yaml:ro,Z" \
		-v "$(FABRICX_DIR)/crypto/peerOrganizations/org1.example.com/peers/fxconfig.org1.example.com/tls:/tls:ro,Z" \
		-v "$(FABRICX_DIR)/crypto/peerOrganizations/org1.example.com/users/User1@org1.example.com/msp:/msp:ro,Z" \
		-v "$(FABRICX_DIR)/crypto/peerOrganizations/org1.example.com/msp/tlscacerts/tlsca.org1.example.com-cert.pem:/org-tls-ca.pem:ro,Z" \
		-v "$(FABRICX_DIR)/crypto/ordererOrganizations/orderer-org-1/msp/tlscacerts/tlsca.orderer-org-1-cert.pem:/orderer-tls-ca.pem:ro,Z" \
		$(TOOLS_IMAGE) \
		sh -c 'fxconfig namespace list --config=/config/fxconfig.yaml 2>/dev/null | grep -q ") $$FX_NS:" || \
		fxconfig namespace create "$$FX_NS" --policy="$$FX_POLICY" --endorse --submit --wait --config=/config/fxconfig.yaml'
endef

.PHONY: list-namespaces
list-namespaces:
	@$(DOCKER) run --rm --network $(NETWORK) \
		$(RUN_AS) \
		-v "$(FABRICX_DIR)/fxconfig.yaml:/config/fxconfig.yaml:ro,Z" \
		-v "$(FABRICX_DIR)/crypto/peerOrganizations/org1.example.com/peers/fxconfig.org1.example.com/tls:/tls:ro,Z" \
		-v "$(FABRICX_DIR)/crypto/peerOrganizations/org1.example.com/users/User1@org1.example.com/msp:/msp:ro,Z" \
		-v "$(FABRICX_DIR)/crypto/peerOrganizations/org1.example.com/msp/tlscacerts/tlsca.org1.example.com-cert.pem:/org-tls-ca.pem:ro,Z" \
		-v "$(FABRICX_DIR)/crypto/ordererOrganizations/orderer-org-1/msp/tlscacerts/tlsca.orderer-org-1-cert.pem:/orderer-tls-ca.pem:ro,Z" \
		$(TOOLS_IMAGE) \
		fxconfig namespace list --config=/config/fxconfig.yaml

.PHONY: init-namespace-org1
init-namespace-org1: POLICY = AND('Org1MSP.member')
init-namespace-org1:
	$(run-fxconfig)

.PHONY: init-namespace-org2
init-namespace-org2: POLICY = AND('Org2MSP.member')
init-namespace-org2:
	$(run-fxconfig)

.PHONY: init-namespace
init-namespace:
	$(run-fxconfig)

# Wholesale CBDC namespaces used by X-Client (branch `main`). Existing namespaces are skipped.
# Changing a policy here also needs the same rule in every X-Client config (endorsement_policies).
define register
	out=$$($(MAKE) --no-print-directory init-namespace NS=$(1) POLICY="$(2)" 2>&1) || { echo "$$out" | tail -20; echo "Error: could not register $(1)"; exit 1; }; \
	echo "  ok  $(1)  $(2)"
endef

.PHONY: init-namespace-cbdc
init-namespace-cbdc:
	@echo "Registering CBDC namespaces..."
	@$(call register,cbdc_public,AND('CentralBankMSP.member'))
	@$(call register,cbdc_cb_pdc,AND('CentralBankMSP.member'))
	@$(call register,cbdc_credits,AND('CentralBankMSP.member'))
	@$(call register,cbdc_org1_pdc,AND('Org1MSP.member','CentralBankMSP.member'))
	@$(call register,cbdc_org2_pdc,AND('Org2MSP.member','CentralBankMSP.member'))

# ─── Beginner targets ───────────────────────────────────────────────────────

.PHONY: help
help:
	@echo "Fabric-X 3-org (CentralBank + Org1 + Org2) network"
	@echo ""
	@echo "  make up       generate crypto (first run), start the network, register the CBDC namespaces"
	@echo "  make status   show containers and registered namespaces"
	@echo "  make down     stop the network (keeps the ledger)"
	@echo "  make reset    stop and delete ledger and crypto (next 'make up' starts from scratch)"
	@echo ""
	@echo "Then run X-Client (branch main): see README.md"

# Refuse to start if a Fabric-X network from another folder is running: they share container names and ports.
define guard_other_network
	id=$$($(DOCKER) ps -q --filter name=^orderer-party1-router$$); \
	if [ -n "$$id" ]; then \
		dir=$$($(DOCKER) inspect -f '{{ index .Config.Labels "com.docker.compose.project.working_dir" }}' $$id); \
		if [ "$$dir" != "$(FABRICX_DIR)" ]; then \
			echo "Error: another Fabric-X network is already running (started from $$dir)."; \
			echo "Stop it first:   cd $$dir && make down"; exit 1; \
		fi; \
	fi
endef

.PHONY: up
up:
	@$(guard_other_network)
	@if [ ! -d "$(FABRICX_DIR)/crypto" ]; then echo "Generating crypto material and genesis block..."; $(MAKE) --no-print-directory init >/dev/null 2>&1 || { echo "Error: make init failed (run 'make init' to see why)"; exit 1; }; fi
	@if [ -z "$$($(OWN_CONTAINERS) -q)" ]; then \
		echo "Starting the network (first start pulls images and takes a few minutes)..."; \
		$(MAKE) --no-print-directory start-3org >/dev/null || exit 1; \
	else \
		$(COMPOSE) $(foreach f,compose.yaml compose.org2.yaml compose.centralbank.yaml,-f $(FABRICX_DIR)/$(f)) $(COMPOSE_OVERRIDE) up -d >/dev/null 2>&1; \
	fi
	@$(MAKE) --no-print-directory wait-healthy
	@$(MAKE) --no-print-directory init-namespace-cbdc
	@echo ""
	@echo "Network is up. Next: start X-Client (branch main):"
	@echo "  cd <X-Client> && ./scripts/copy-crypto.sh $(FABRICX_DIR) && docker compose -f docker-compose.3org.yaml up -d --build"

.PHONY: wait-healthy
wait-healthy:
	@echo "Waiting for all containers to become healthy..."
	@for i in $$(seq 1 120); do \
		st=$$($(OWN_CONTAINERS) --format '{{.Status}}'); \
		if [ -n "$$st" ] && ! echo "$$st" | grep -q "starting\|unhealthy\|Restarting"; then \
			echo "  $$(echo "$$st" | wc -l) containers up and healthy"; exit 0; \
		fi; sleep 5; \
	done; echo "Error: containers not healthy after 10 minutes; check 'docker ps -a' and 'docker logs <name>'"; exit 1

.PHONY: down
down: stop

.PHONY: reset
reset: purge
	@rm -rf $(FABRICX_DIR)/crypto
	@echo "Ledger and crypto deleted. 'make up' starts from scratch; afterwards re-run X-Client's copy-crypto.sh."

.PHONY: status
status:
	@n=$$($(OWN_CONTAINERS) -q | wc -l); echo "$$n containers running from $(FABRICX_DIR)"
	@$(OWN_CONTAINERS) --format 'table {{.Names}}\t{{.Status}}' | sort
	@if [ -n "$$($(OWN_CONTAINERS) -q)" ]; then $(MAKE) --no-print-directory list-namespaces 2>/dev/null | grep -E '^(Installed|[0-9]+\))'; fi
