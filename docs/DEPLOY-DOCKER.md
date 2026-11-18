# Deploy com Docker

Imagem: `ghcr.io/xnetinho/rustdesk-server-s6:latest` (amd64, arm64, armv7, i386). Roda `hbbs` e `hbbr` no mesmo container. A imagem `ghcr.io/xnetinho/rustdesk-server` (classic) tem só os binários, sem as variáveis abaixo.

Os workflows `build.yaml` publicam no GHCR com o `GITHUB_TOKEN`, sem outros secrets. O disparo é manual (`workflow_dispatch`) ou por tag `vX.Y.Z`.

## Variáveis (imagem s6)

| Variável | Função |
|---|---|
| `RELAY` | `host:21117`, o relay que o `hbbs` anuncia |
| `ENCRYPTED_ONLY` | `1` faz `hbbs` e `hbbr` exigirem a chave. Sem isso o `hbbr` aceita qualquer cliente |
| `KEY_PUB` | chave pública, gravada em `/data/id_ed25519.pub` |
| `KEY_PRIV` | chave privada, gravada em `/data/id_ed25519`. **Segredo**: não versionar |

Gerar o par: `rustdesk-utils genkeypair`. A chave pública é a mesma do secret `RD_KEY` do cliente.

## Portas

TCP 21115, 21116, 21117 e UDP 21116. As portas 21118 (WebSocket do `hbbs`) e 21119 (WebSocket do `hbbr`) **não** são publicadas: só o Traefik as alcança dentro da rede, porque o servidor confia no header `X-Real-IP`.

## Stack (Portainer, com Traefik)

```yaml
services:
  rustdesk:
    image: ghcr.io/xnetinho/rustdesk-server-s6:latest
    container_name: rustdesk
    restart: unless-stopped
    environment:
      RELAY: rustdesk.example.com:21117
      ENCRYPTED_ONLY: "1"
      KEY_PUB: ${KEY_PUB}
      KEY_PRIV: ${KEY_PRIV}
    ports:
      - "21115:21115/tcp"
      - "21116:21116/tcp"
      - "21116:21116/udp"
      - "21117:21117/tcp"
    volumes:
      - /opt/rustdesk:/data
    networks:
      - network_public
    labels:
      traefik.enable: "true"
      traefik.docker.network: network_public
      traefik.http.routers.rd-id.rule: Host(`rustdesk.example.com`) && PathPrefix(`/ws/id`)
      traefik.http.routers.rd-id.entrypoints: websecure
      traefik.http.routers.rd-id.tls.certresolver: letsencryptresolver
      traefik.http.routers.rd-id.service: rd-id
      traefik.http.services.rd-id.loadbalancer.server.port: "21118"
      traefik.http.routers.rd-relay.rule: Host(`rustdesk.example.com`) && PathPrefix(`/ws/relay`)
      traefik.http.routers.rd-relay.entrypoints: websecure
      traefik.http.routers.rd-relay.tls.certresolver: letsencryptresolver
      traefik.http.routers.rd-relay.service: rd-relay
      traefik.http.services.rd-relay.loadbalancer.server.port: "21119"

networks:
  network_public:
    external: true
```

`/opt/rustdesk` guarda o banco `db_v2.sqlite3` e as chaves. Entradas e saídas do Traefik (`websecure`, `letsencryptresolver`) são os nomes do nosso ambiente.

## Verificar

- `rustdesk-utils doctor <host>` testa as portas TCP (não o UDP).
- O handshake `wss://<host>/ws/id` e `/ws/relay` deve responder `101 Switching Protocols`.
- Relay: sem `ENCRYPTED_ONLY=1`, qualquer cliente usa a sua banda.
