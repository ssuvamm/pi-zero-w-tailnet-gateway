# Access Home Cameras from Home Assistant on a Remote Coolify Server

This is a follow-up to the Raspberry Pi Zero W + Tailscale remote home-network setup.

The goal was to let a **Home Assistant instance running remotely in Coolify** access cameras that are physically inside the home network, without installing Tailscale directly on the cameras.

## The problem

Home Assistant was running as a Docker container on a remote server managed by Coolify.

The cameras were at home:

```text
Camera: 192.168.0.51
MJPEG port: 8080
```

The home network was already exposed through the Raspberry Pi Zero W using Tailscale and NETMAP:

```text
Remote network
      |
   Tailscale
      |
192.168.10.0/24
      |
Pi Zero W
      |
    NETMAP
      |
192.168.0.0/24
      |
Home devices
```

Therefore the camera was reachable remotely as:

```text
192.168.10.51:8080
```

The problem was that the remote Home Assistant Docker container did not automatically have access to the Tailscale subnet route.

## Solution

Instead of changing the existing Pi gateway, a **Tailscale sidecar container** was added to the Home Assistant Compose stack.

Home Assistant shares the sidecar's network namespace:

```yaml
homeassistant:
  network_mode: 'service:tailscale'
```

The Tailscale sidecar uses kernel networking and accepts subnet routes:

```yaml
tailscale:
  image: 'tailscale/tailscale:latest'
  hostname: 'homeassistant-ts'
  environment:
    - 'TS_AUTHKEY=${TS_AUTHKEY:?}'
    - 'TS_STATE_DIR=/var/lib/tailscale'
    - 'TS_USERSPACE=false'
    - 'TS_EXTRA_ARGS=--accept-routes'
  volumes:
    - 'homeassistant-tailscale:/var/lib/tailscale'
  devices:
    - '/dev/net/tun:/dev/net/tun'
  cap_add:
    - NET_ADMIN
    - NET_RAW
```

The Home Assistant container then shares that network namespace:

```yaml
homeassistant:
  network_mode: 'service:tailscale'
  depends_on:
    - tailscale
```

This gives Home Assistant access to the same Tailscale interface and routes as the sidecar.

## Complete Compose configuration

```yaml
services:
  tailscale:
    image: 'tailscale/tailscale:latest'
    hostname: 'homeassistant-ts'
    environment:
      - 'TS_AUTHKEY=${TS_AUTHKEY:?}'
      - 'TS_STATE_DIR=/var/lib/tailscale'
      - 'TS_USERSPACE=false'
      - 'TS_EXTRA_ARGS=--accept-routes'
    volumes:
      - 'homeassistant-tailscale:/var/lib/tailscale'
    devices:
      - '/dev/net/tun:/dev/net/tun'
    cap_add:
      - NET_ADMIN
      - NET_RAW
    healthcheck:
      test:
        - CMD-SHELL
        - 'tailscale status --json | grep -q "BackendState"'
      interval: 10s
      timeout: 5s
      retries: 10

  homeassistant:
    image: 'ghcr.io/home-assistant/home-assistant:2025.10.2'
    network_mode: 'service:tailscale'
    depends_on:
      - tailscale
    environment:
      - SERVICE_URL_HOMEASSISTANT_8123
      - 'TZ=${TZ:-UTC}'
      - 'DISABLE_JEMALLOC=${DISABLE_JEMALLOC:-false}'
    volumes:
      - 'homeassistant-config:/config'
      - '/run/dbus:/run/dbus:ro'
      -
        type: bind
        source: ./configuration.yaml
        target: /config/configuration.yaml
    privileged: true
    healthcheck:
      test:
        - CMD
        - curl
        - '-f'
        - 'http://localhost:8123'
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

volumes:
  homeassistant-config:
  homeassistant-tailscale:
```

## Tailscale authentication

The sidecar needs a **Tailscale auth key**, not a Tailscale API access token.

In Coolify, add:

```text
TS_AUTHKEY=<Tailscale auth key>
```

The key is kept as an environment variable rather than being written into the Compose file.

## ESPHome camera configuration

The cameras in this setup are **ESPHome-based devices**.

They were initially flashed/configured with ESPHome from a laptop. The camera was first tested using its direct MJPEG endpoint, but the final Home Assistant setup uses the **ESPHome integration/connection** instead of treating the device as a generic MJPEG camera.

Once the Tailscale sidecar was running, it installed the advertised route:

```text
192.168.10.0/24
```

inside the shared network namespace.

The ESPHome device remains physically on the home LAN:

```text
ESPHome camera
192.168.0.51
```

but Home Assistant reaches it remotely through the translated address:

```text
192.168.10.51
```

The important part is that Home Assistant's ESPHome connection now travels through:

```text
Home Assistant
    ↓
Tailscale sidecar
    ↓
192.168.10.0/24
    ↓
Pi Zero W
    ↓ NETMAP
192.168.0.0/24
    ↓
ESPHome camera
```

So the camera does **not** need Tailscale installed on the ESP device. The ESPHome device continues operating normally on the home network, while the remote Home Assistant instance reaches it through the Tailscale subnet route.

The original MJPEG endpoint was useful for testing the camera, but it is not the final integration method used in Home Assistant.

## Coolify domain gotcha

There was one important side effect.

Because Home Assistant uses:

```yaml
network_mode: 'service:tailscale'
```

it no longer has its own Docker network namespace.

That means Coolify's reverse proxy cannot target the `homeassistant` container in the normal way.

The working solution was to point the Coolify domain at the **Tailscale service** instead, using Home Assistant's internal port:

```text
Domain:
ha.example.com

Target service:
tailscale

Internal port:
8123
```

The traffic path becomes:

```text
Internet
   |
Cloudflare
   |
Coolify reverse proxy
   |
Tailscale container :8123
   |
shared network namespace
   |
Home Assistant :8123
```

The Home Assistant domain continued to work through Cloudflare/HTTPS after this change.

## Final architecture

```text
                         Tailscale
                            │
                    ┌───────▼────────┐
                    │ Remote Server  │
                    │    Coolify     │
                    │                │
                    │  Tailscale     │
                    │   sidecar      │
                    │       │        │
                    │       ├────────┤
                    │       │ shared │
                    │       │ network│
                    │       ▼        │
                    │ Home Assistant │
                    └───────┬────────┘
                            │
                       Tailscale route
                            │
                     192.168.10.0/24
                            │
                            ▼
                     Raspberry Pi Zero W
                            │
                          NETMAP
                            │
                     192.168.0.0/24
                            │
                            ▼
                    Camera 192.168.0.51
```

## What this solves

- Home Assistant stays on the remote Coolify server.
- Cameras remain on the private home LAN.
- Cameras do not need Tailscale installed.
- The existing Pi Zero W subnet router remains unchanged.
- The `192.168.10.0/24` translation avoids conflicts with a remote `192.168.0.x` network.
- Home Assistant receives the Tailscale subnet route through its sidecar.
- Coolify can still expose Home Assistant through its domain and Cloudflare.

## Key idea

The important part is not installing Tailscale *inside Home Assistant* itself.

Instead:

> **Run Tailscale beside Home Assistant and make Home Assistant share Tailscale's network namespace.**

That gives the application the routing capabilities of the Tailscale container while keeping the existing home-network gateway untouched.
