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
desktop app ───────wss───▶ relay.example.com ─┘        │  (NPM)
                                                       ▼
                         UNRAID-IP:8095      UNRAID-IP:9443          published host ports
                               │                   │
                         manabrew-web:80     manabrew-relay:9443 ◀──ws── manabrew-node
                         └──────────── "manabrew" bridge network ─────────────┘
```

## Option A: one compose stack (recommended)

[compose/docker-compose.yml](compose/docker-compose.yml) runs the same three
containers from a single file, with one `.env`. Use Unraid's **Docker Compose
Manager** plugin (from Community Applications):

1. Docker tab → Compose → Add New Stack, named `manabrew`.
2. Paste in `docker-compose.yml`. Under "Edit Stack → ENV file", paste
   `.env.example` and fill it in.
3. Networking needs nothing from you. The stack creates its own `manabrew` bridge
   network (the node reaches the relay by name on it) and publishes only two
   ports on the Unraid host: web on `WEB_HOST_PORT` (8095) and relay on
   `RELAY_HOST_PORT` (9443). NPM forwards to those. See
   [Nginx Proxy Manager](#nginx-proxy-manager).
4. Compose Up. To update, click "Update Stack", which pulls all three and restarts
   them together.

What this gets you over the templates: the key is set once, the network is
already wired up, the node waits until the relay is healthy before starting, and
all three containers always run the same version (`MANABREW_TAG`). After that,
the [reverse proxy](#reverse-proxy-required-for-browser-play) steps are the same.

## Option B: three Unraid templates

1. Create the shared network (Unraid terminal), so the node can reach the relay
   by name:
   ```bash
   docker network create manabrew
   ```
   The templates publish web on 8095 and relay on 9443, which is what NPM forwards
   to.
2. Copy the XMLs to `/boot/config/plugins/dockerMan/templates-user/`.
3. Docker → Add Container → pick the template: relay first, then web, then node (optional).
4. Generate a long random key (`openssl rand -hex 32`). Set it as relay
   `MANABREW_SERVER_KEY` and node `SELF_HOSTED_NODE_SERVER_KEY`. Set it as web's
   `RELAY_PASSWORD` too, but only once the web hostname is behind an NPM Access
   List. See [Keeping it private](#keeping-it-private).

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

**2. Forward to the Unraid host's ports.** NPM never joins the Manabrew network.
It forwards to the two ports the stack publishes on the Unraid host, using the
server's LAN IP (`UNRAID-IP` below, e.g. `192.168.2.147`). That stays the same no
matter how often the containers get recreated or re-addressed.

If NPM runs on `br0` (its own LAN IP), it **can't reach the Unraid host's IP by
default**, because macvlan/ipvlan isolates containers from their host. Turn on
Settings → Docker → **Host access to custom networks** (Advanced View; you have
to stop the Docker service to change it). If NPM runs on a normal bridge network
with published ports, you don't need this.

Fallback if you can't enable host access: attach NPM to the stack's network and
forward to `manabrew-web:80` / `manabrew-relay:9443` by name instead. Add this to
the end of NPM's template under Post Arguments, so it reattaches every time
Unraid recreates NPM. This is a common community workaround, not an official
Unraid feature. Use your NPM container's name, from `docker ps`:

```bash
&& docker network connect manabrew Nginx-Proxy-Manager-Official
```

**3. Proxy host: `play.example.com`**

| Tab | Setting |
|---|---|
| Details | Scheme `http`, Forward Hostname `UNRAID-IP`, Port `8095` (`WEB_HOST_PORT`) |
| Details | Cache Assets **off**: it rewrites `Cache-Control`, and `config.js` must not be cached |
| Details | Block Common Exploits on, Websockets Support off (not needed) |
| SSL | Request a new Let's Encrypt cert, Force SSL, HTTP/2, HSTS |
| Access List | Recommended: your friends list. See [Keeping it private](#keeping-it-private). It lets you set `RELAY_PASSWORD`, which browser Forge "Play vs AI" needs. |

**4. Proxy host: `relay.example.com`**

| Tab | Setting |
|---|---|
| Details | Scheme `http`, Forward Hostname `UNRAID-IP`, Port `9443` (`RELAY_HOST_PORT`) |
| Details | **Websockets Support on** (required) |
| Details | Block Common Exploits on, Cache Assets off |
| SSL | Let's Encrypt cert, Force SSL, HTTP/2 |
| Access List | **None.** The desktop app can't get past basic auth, and the server key already protects this host. |

You don't need anything in the Advanced tab. The relay pings every 25 s, which is
under nginx's 60 s idle timeout, so game sockets stay open.

**5. Direct ports.** `UNRAID-IP:8095` and `UNRAID-IP:9443` are also reachable
directly on your home network, without TLS. That's LAN only: don't forward them
on the router. The relay still requires the key. Health/metrics (9444) isn't
published at all.

**6. Check it:**

```bash
curl -sI https://play.example.com | grep -i cross-origin
```

You should see all three `Cross-Origin-*` headers. Then open the site, go to
DevTools → Console, and confirm `crossOriginIsolated` is `true`. For the relay,
`curl -sI https://relay.example.com` should return a response from nginx, not a
502. A 502 on both hosts means NPM can't reach `UNRAID-IP`. With NPM on `br0`,
that's almost always *Host access to custom networks* being off (step 2).

## Which hostnames are public

| Hostname | Must be public? | Why |
|---|---|---|
| `relay.example.com` | **Yes** | Every client, browser or desktop, connects straight to it. The server key protects it. Never put a login page in front of it. |
| `play.example.com` | Only if friends play in a browser on *your* web client | It only serves the app to browsers. Put it behind an NPM Access List (see [Keeping it private](#keeping-it-private)). |

Can `play` be internal-only, with friends using the official `play.manabrew.app`
and adding your relay in Settings → Server? Basic play works that way, but there
are two catches:
- **Forge "Play vs AI" won't use your server.** That button always reconnects to
  the site's built-in relay, so on the official site it runs on Manabrew's own
  servers, not your node. Forge *multiplayer tables* created while connected to
  your relay do use your node.
- **Version drift:** the official site always runs the newest client. If you pin
  `MANABREW_TAG` to an older relay version, the two can stop understanding each
  other. Stay on `latest` if you go this way.

Desktop-only friends never need `play` at all.

Even internal-only, `play` still needs HTTPS for browser games. An NPM Access List
that only allows your LAN subnet does that: public DNS and a Let's Encrypt cert,
but outside visitors get a 403.

## How friends connect

- **Browser:** open `https://play.example.com` and sign in with the Access List
  username and password you gave them. Everything else is preconfigured.
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
| web | `RELAY_PASSWORD` = server key | "Play vs AI" reconnects using the site's built-in key and ignores the one a player typed into Settings. If it's empty, Forge "Play vs AI" fails with "Invalid server key". |
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

1. **Put `play.example.com` behind an NPM Access List, then set `RELAY_PASSWORD`.**
   `RELAY_PASSWORD` is written in plain text into `/config.js`, so only logged-in
   friends should be able to load that file. Browser Forge "Play vs AI" needs the
   key there (see [Bots](#bots)).
   - NPM → Access Lists → Add. On the **Authorization** tab, add one
     username/password per friend. On the **Access** tab, `allow` your LAN
     subnet (e.g. `192.168.1.0/24`), and set **Satisfy Any** on the Details tab.
     With that, you skip the login prompt at home and friends log in once.
     Then attach the list to the `play` proxy host.
   - Without an Access List, leave `RELAY_PASSWORD` empty. Friends paste the key
     into Settings → Server instead. Everything works except browser Forge
     "Play vs AI" (basic AI and Forge multiplayer tables are fine).
   - **Don't** put `relay.example.com` behind a login page. The desktop app can't
     get through basic auth or a redirect, so the key has to protect that
     hostname on its own.
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
