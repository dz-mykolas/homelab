## Prep folders & env

In Komodo add:

```bash
HEADSCALE_SERVER_URL=https://vpn.example.com
HEADSCALE_DNS_BASE_DOMAIN=tail.example.com
TS_AUTHKEY=generated_in_next_step
```

(Caddy will terminate TLS; Headscale runs HTTP on :8080 internally.)

## Start Headscale, create user + preauth key

```bash
podman compose -f compose-headscale.yml up -d headscale

podman exec -it headscale headscale users create main
podman exec -it headscale headscale preauthkeys create --user main --reusable --expiration 720h
```

Copy the tskey-... into Komodo as TS_AUTHKEY=

## Start the Tailscale subnet router (needs TUN & NET_ADMIN)

With Podman, run this service rootful:

```bash
sudo podman compose -f compose-headscale.yml up -d tailnet-router
sudo podman logs -f tailnet-router
```

You should see it logged in and advertising routes: 172.30.0.0/24.

## Approve the route in Headscale

```bash
podman exec -it headscale headscale routes list
podman exec -it headscale headscale routes enable -i tailgw --routes 172.30.0.0/24
```

## Join a client to your tailnet

On your laptop/box/phone:

```bash
tailscale up --login-server https://vpn.example.com --authkey tskey-<your-key> --accept-routes=true
```

Verify: tailscale status should show the subnet route via tailgw.

## Attach apps to the tailnet network (in your Jellyfin/arr compose)

Any service you add to the external tailnet network (e.g., 172.30.0.11 for Sonarr) is now reachable only to your tailnet:

Sonarr → http://172.30.0.11:8989

Radarr → http://172.30.0.12:7878

Prowlarr → http://172.30.0.13:9696

qBittorrent → http://172.30.0.14:8080

Shoko → http://172.30.0.15:8111

(Do not publish ports: on those services; that keeps them off LAN/Internet.)
