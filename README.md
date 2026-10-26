# Manabrew on Unraid

Three templates, from upstream images (all `linux/amd64`, published to GHCR):

| Template | Image | Needed? | Public hostname |
|---|---|---|---|
| `my-manabrew-relay.xml` | `manabrew-server` | **Required**: the lobby every client connects to | `relay.example.com` → `:9443` |
| `my-manabrew-web.xml` | `manabrew-web` | Recommended: your own browser URL, already pointed at your relay | `play.example.com` → `:8095` |
| `my-manabrew-node.xml` | `manabrew-node` | Needed if browser players want the Forge engine or Forge AI ([Bots](#bots)) | none (only connects out) |

```
friends' browser ──https──▶ play.example.com ─┐
friends' browser ──wss───▶ relay.example.com ─┤  reverse proxy (TLS)
desktop app ───────wss───▶ relay.example.com ─┘        │
                                                       ▼
                         manabrew-web:80     manabrew-relay:9443 ◀──ws── manabrew-node
```

## Option A: one compose stack (recommended)

[compose/docker-compose.yml](compose/docker-compose.yml) runs the same three
containers from a single file, with one `.env`. Use Unraid's **Docker Compose
Manager** plugin (from Community Applications):

1. Docker tab → Compose → Add New Stack, named `manabrew`.
2. Paste in `docker-compose.yml`. Under "Edit Stack → ENV file", paste
   `.env.example` and fill it in.
3. Set `PROXY_NETWORK` to the network your NPM container uses. The stack joins it
   as an external network, so NPM reaches `manabrew-web` and `manabrew-relay` by
   name. Nothing is published on host ports.
4. Compose Up. To update, click "Update Stack", which pulls all three and restarts
   them together.

What this gets you over the templates: the key is set once, the network is
already wired up, the node waits until the relay is healthy before starting, and
all three containers always run the same version (`MANABREW_TAG`). After that,
the [reverse proxy](#reverse-proxy-required-for-browser-play) steps are the same.

## Option B: three Unraid templates

1. Create the shared network (Unraid terminal):
   ```bash
   docker network create manabrew
   ```
   If your reverse proxy container already uses a custom network (e.g. `proxynet`),
   set each template's Network to that one instead and skip this step.
2. Copy the XMLs to `/boot/config/plugins/dockerMan/templates-user/`.
3. Docker → Add Container → pick the template: relay first, then web, then node (optional).
4. Generate a long random key (`openssl rand -hex 32`). Set it as relay
   `MANABREW_SERVER_KEY` and node `SELF_HOSTED_NODE_SERVER_KEY`. Leave web's
   `RELAY_PASSWORD` empty unless the web hostname is behind a login. See
   [Keeping it private](#keeping-it-private).

## Reverse proxy (required for browser play)

The browser client needs **HTTPS**. Its WASM engine uses SharedArrayBuffer, and
browsers only allow that on a secure, cross-origin-isolated page. Over plain
`http://192.168.x.x` the page loads but games never start.

The relay has to be on its own hostname. The client connects to `host:port` with
no path, so you can't put it under a path like `/relay`.

### Nginx Proxy Manager

**1. DNS and router.** Create two records, `play` and `relay`, pointing at your
public IP. A wildcard `*.example.com` also works. On the router, forward
external 80 and 443 to NPM's HTTP and HTTPS ports. If you use Cloudflare's orange
cloud, set SSL mode to *Full (strict)*. Cloudflare passes WebSockets through.

**2. Address containers by name, not by IP.** Container IPs on a Docker network
change when containers are recreated, for example on every Unraid update. Put NPM
on the `manabrew` network as well:

```bash
docker network connect manabrew Nginx-Proxy-Manager-Official
```

(Use your NPM container's real name, from `docker ps`.) Unraid drops networks
added this way whenever NPM is recreated. The lasting fix is one shared network
that NPM and all three Manabrew containers use as their main network. If NPM
already has a custom network (often `proxynet`), set the Manabrew templates'
Network to that instead of `manabrew`. Container names still resolve, so the
node's `ws://manabrew-relay:9443` doesn't change. If you'd rather not share a
network, forward to `UNRAID-IP` plus the mapped host ports (`8095`, `9443`)
instead. That works too.

**3. Proxy host: `play.example.com`**

| Tab | Setting |
|---|---|
| Details | Scheme `http`, Forward Hostname `manabrew-web`, Port `80` |
| Details | Cache Assets **off**: it rewrites `Cache-Control`, and `config.js` must not be cached |
| Details | Block Common Exploits on, Websockets Support off (not needed) |
| SSL | Request a new Let's Encrypt cert, Force SSL, HTTP/2, HSTS |
| Access List | Optional. See [Keeping it private](#keeping-it-private). Only use one if you want `RELAY_PASSWORD` prefilled. |

**4. Proxy host: `relay.example.com`**

| Tab | Setting |
|---|---|
| Details | Scheme `http`, Forward Hostname `manabrew-relay`, Port `9443` |
| Details | **Websockets Support on** (required) |
| Details | Block Common Exploits on, Cache Assets off |
| SSL | Let's Encrypt cert, Force SSL, HTTP/2 |
| Access List | **None.** The desktop app can't get past basic auth, and the server key already protects this host. |

You don't need anything in the Advanced tab. The relay pings every 25 s, which is
under nginx's 60 s idle timeout, so game sockets stay open.

**5. Lock down the direct ports (optional).** Once NPM reaches the containers by
name, you can delete the port mappings from the relay (9443, 9444) and web (8095)
templates. Then the only way in is through NPM with TLS.

**6. Check it:**

```bash
curl -sI https://play.example.com | grep -i cross-origin
```

You should see all three `Cross-Origin-*` headers. Then open the site, go to
DevTools → Console, and confirm `crossOriginIsolated` is `true`. For the relay,
`curl -sI https://relay.example.com` should return a response from nginx, not a
502. A 502 means NPM can't reach `manabrew-relay`, which usually means the
network from step 2 isn't set up.

## How friends connect

- **Browser:** open `https://play.example.com`. The host is already filled in.
  On the first visit, go to Settings → Server and paste the key. It's saved in
  that browser. (They could also use the official `play.manabrew.app` and add your
  relay there, so the web container is a convenience, not a requirement.)
- **Windows/macOS desktop app:** Settings → Server. Host `relay.example.com`,
  port `443`, password = your server key.

## Bots

There are two AIs, and they run in different places:

| | Basic AI (Manabrew engine) | Forge AI (Forge engine) |
|---|---|---|
| **Desktop app** | Runs locally | Runs locally: the app hosts its own Forge room |
| **Browser** | Runs locally (WASM) | **Runs in a `manabrew-node` room** |

- **"Add bot" in a multiplayer room:** the bot runs inside the room creator's own
  client and joins the relay as an extra player. You don't need the node, but the
  bot disappears if that player closes their tab or app.
- **Browser + Forge** (Forge "Play vs AI", or creating a table with the Forge
  engine): the browser looks for an **empty** node room with format `Any`, claims
  it, and the node starts the game and any bots, using whatever deck the player
  picked.

So for a mixed group of browser and desktop players who want Forge bots, yes, run
all three containers. The settings that matter:

| Where | Setting | Why |
|---|---|---|
| web | `HOSTED_AI_ENABLED=true` | Shows the Forge "Play vs AI" option in the browser |
| node | `SELF_HOSTED_NODE_FORMAT=any` | Browsers only claim `Any` rooms. Players still choose the format per game. |
| node | `SELF_HOSTED_NODE_BOT_ENABLED=false` | `true` puts a preset-deck bot in every room as it opens, so rooms are never empty and browsers skip them |
| node | `SELF_HOSTED_NODE_ROOM_PASSWORD` empty | The browser claims a room without sending a password |
| node | `SELF_HOSTED_NODE_MAX_GAMES=3` (or more) | How many browser Forge games can run at the same time. Each one uses a room. |

Optional: `FORGE_WASM=true` on web shows a Settings toggle that lets a browser run
Forge itself. It's a 72 MB download, and the browser falls back to the node if it
fails the compatibility test. Upstream ships it switched off, so treat it as
experimental and keep the node anyway.

## Keeping it private

The relay is the only thing that matters, and the server key gates it. A
connection's very first message has to be `Authenticate` with the right key. Any
other first message gets dropped, and so does no message within 10 seconds.
Without the key, nobody can list rooms, create one, or join one. So the whole job
is keeping that key out of public view:

1. **Don't put the key in the web container.** `RELAY_PASSWORD` is written in
   plain text into `/config.js`, which anyone who loads your site can read. Leave
   it empty, and friends paste the key into Settings once. A stranger who finds
   `play.example.com` gets a static page that fails with "Invalid server key".
   - *Optional:* put `play.example.com` behind a login, such as an NPM Access List
     (basic auth with one user per friend), Authelia, or Cloudflare Access. Then
     you can set `RELAY_PASSWORD` so friends don't have to paste anything.
   - **Don't** put `relay.example.com` behind a login page. The desktop app can't
     get through a browser redirect, so the key has to protect that hostname
     on its own.
2. **Make the key long and random** (`openssl rand -hex 32`). The relay doesn't
   rate-limit failed attempts, so a short key could be brute-forced. A 64-hex-char
   key can't be.
3. **Only proxy port 9443.** Never expose 9444, which serves `/health` and
   `/metrics`.
4. **Room passwords** that a player sets when hosting a table add a second lock
   on that table. Don't set one on the node, though: browsers would no longer be
   able to claim its rooms (see [Bots](#bots)). The relay key already keeps
   strangers away from them.

Limits to be aware of:
- Everyone shares one key. If someone leaks it, change it on the relay and node,
  restart both, and send the new key to everyone.
- There are no accounts. Usernames are whatever each player types in, so friends
  could impersonate each other.
- Once someone has the key, they can create rooms. Nothing restricts that further.

## Things to know
- **Who runs the game:** for Manabrew-engine tables, the hosting player's client
  runs the engine and the relay just passes messages. For browser Forge tables,
  the node runs it.
- **Node RAM:** about 1 GB for 4 rooms with the default shared isolate. The template
  caps the container at 2 GB (`--memory=2g` in Extra Parameters). Raise the cap if
  you set `MAX_GAMES` above 4.
- **Stopping the node** ends games in progress. To drain it gracefully, run
  `docker kill --signal=USR1 manabrew-node`. It exits once the current games finish.
- **Updates:** the templates track `:latest` and the project ships often. Update
  relay, web and node together so they speak the same protocol.
- **No persistent data:** none of the three containers needs an appdata path.
- **LAN-only option:** the relay can advertise over mDNS so desktop apps find it
  without any setup (`MANABREW_LAN_ADVERTISE=true`). This needs Network `host` or
  `br0`, and it only helps players on your home network.
