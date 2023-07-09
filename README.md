# XDSwap

XDSwap is a Uniswap fork deployed to XDC mainnet. Due to lack of time and a late start, I wasn't able to finalize the deployment of the Uniswap interface.

## Infrastructure

Some infrastructure had to be set up for this project.

### The Graph

Hosted on the following server: 37.187.27.137. Everything was kept open for this proof of concept, but ideally ports 8000, 8001, 8020 and 8030 should be behind SSL, and 8020 and 8030 should be restricted or not accessible to the public.

Docker Compose file used:

### IPFS

Hosted on the following URL: http://37.187.27.137:5001. Again, this should be protected by SSL and put on a specific domain rather than direct IP. Should add authentication as well.

### Docker Compose

```yaml
version: "3.9"

services:
  graph-node:
    image: gzliudan/graph-node
    ports:
      - "8000:8000"
      - "8001:8001"
      - "8020:8020"
      - "8030:8030"
      - "8040:8040"
    depends_on:
      - ipfs
      - postgres
    environment:
      postgres_host: postgres
      postgres_user: graph-node
      postgres_pass: let-me-in
      postgres_db: graph-node
      ipfs: "ipfs:5001"
      ethereum: "xinfin:traces,archive,no_eip1898:https://erpc.xinfin.network"
      ETHEREUM_REORG_THRESHOLD: 1
      GRAPH_LOG: info
      GRAPH_LOG_TIME_FORMAT: "%Y-%m-%d %H:%M:%S"
    labels:
      traefik.enable: "true"
      traefik.docker.network: "public"
      traefik.http.routers.xdc-graph.rule: "Host(`xdc-graph.apyos.com`)"
      traefik.http.routers.xdc-graph.entrypoints: "http,https"
      traefik.http.routers.xdc-graph.service: "xdc-graph"
      traefik.http.routers.xdc-graph.middlewares: "xdc-graph-redirect"
      traefik.http.routers.xdc-graph.tls: "true"
      traefik.http.routers.xdc-graph.tls.certresolver: "letsencrypt"
      traefik.http.middlewares.xdc-graph-redirect.redirectscheme.scheme: "https"
      traefik.http.middlewares.xdc-graph-redirect.redirectscheme.permanent: "true"
      traefik.http.services.xdc-graph.loadBalancer.server.port: 8000
      traefik.http.services.xdc-graph.loadBalancer.server.scheme: "http"

  ipfs:
    image: ipfs/go-ipfs:v0.10.0
    ports:
      - "5001:5001"
    volumes:
      - ./data/ipfs:/data/ipfs

  postgres:
    image: postgres
    command: ["postgres", "-cshared_preload_libraries=pg_stat_statements"]
    environment:
      POSTGRES_USER: graph-node
      POSTGRES_PASSWORD: let-me-in
      POSTGRES_DB: graph-node
      # FIXME: remove this env. var. which we shouldn't need. Introduced by
      # <https://github.com/graphprotocol/graph-node/pull/3511>, maybe as a
      # workaround for https://github.com/docker/for-mac/issues/6270?
      PGDATA: "/var/lib/postgresql/data"
      POSTGRES_INITDB_ARGS: "-E UTF8 --locale=C"
```

## Deployment

### Deploy the smart contracts

```bash
# Change folder and install dependencies
cd deploy-v3
yarn
yarn build

# Run deploy command
node dist/index.js \
    --private-key $PRIVATE_KEY \
    --json-rpc https://erpc.xinfin.network \
    --weth9-address 0x951857744785E80e2De051c32EE7b25f9c458C42 \
    --native-currency-label XDC \
    --owner-address 0xBC4787Cb82060da29f85fe6C0dA03436019329e7
```

Resulting state (deployed by `0xBC4787Cb82060da29f85fe6C0dA03436019329e7`):

```json
{
  "v3CoreFactoryAddress": "0x3A586887C7cE970157A56f89C180246a3258C6dF",
  "multicall2Address": "0xccca48b4c78Ab89398BB54BB3b0AB67fDF6AcD0E",
  "proxyAdminAddress": "0x897834930A7f3692a364a5077C69b1B44995Fe73",
  "tickLensAddress": "0xBA7dE62De38c37Ee2B229C732C238C119685b765",
  "nftDescriptorLibraryAddressV1_3_0": "0x37F687BDccD588Df1fC7D3F1D1e573F315ecC782",
  "nonfungibleTokenPositionDescriptorAddressV1_3_0": "0xd0cDA7F02d0719B9819098Db05ec3901a00A5083",
  "descriptorProxyAddress": "0x317e6B3e7AF4c78dcEe0E03ac9a0a7f409d7BC06",
  "nonfungibleTokenPositionManagerAddress": "0x4eF22247EaD0C966C232e7f8F45057A9540ad7F4",
  "v3MigratorAddress": "0x99d5D8dd09c7eEa021BD4Ae743f376AAB3D655fa",
  "v3StakerAddress": "0x8f4C0b48db153b72CD0097758701C3B22d36947D",
  "quoterV2Address": "0x2450fFA63b3ea23026A0fA77bBC5f11C72D11CBa",
  "swapRouter02": "0x835dD71196d0F76CfE7F9B004698c5CB1C551eff"
}
```

### Deploy the subgraphs

```bash
# Change folder and install dependencies
cd v3-subgraph
yarn

# Copy config file
cp ../subgraph.yaml .

# Build
npm run codegen
npm run build

# Deploy
npx graph create ianlapham/uniswap-v3 --node http://37.187.27.137:8020
npx graph deploy ianlapham/uniswap-v3 --debug --ipfs http://37.187.27.137:5001 --node http://37.187.27.137:8020
```
