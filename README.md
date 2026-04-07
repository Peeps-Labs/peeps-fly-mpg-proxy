# Fly MPG Proxy

Fly MPG Proxy enables you to connect to [Fly Managed Postgres](https://fly.io/docs/mpg/) from servers and service providers across the Internet.

[Fly Managed Postgres](https://fly.io/docs/mpg/) can only be connected to by machines on the Fly Network, or by clients using [flyctl](https://fly.io/docs/flyctl/).  Fly MPG Proxy changes that by providing a reverse proxy that can be accessed over the Internet and is managed just like any other Fly app.

---

## Peeps fork notes

This is the Peeps Labs fork of `fly-apps/fly-mpg-proxy`. It exists so that Peeps' Trigger.dev Cloud workers (which run outside the Fly network) can reach our Fly Managed Postgres clusters.

### App → cluster mapping

| Fly app | MPG cluster | Cluster ID | Region | Trigger.dev env that uses it |
|---|---|---|---|---|
| `peeps-fly-mpg-proxy-dev` | `peeps-db-dev` | `q49ypo4wmkpr17ln` | `lax` | Staging (= `dev.peepsai.com`) |
| `peeps-fly-mpg-proxy` | `peeps-db` | `1zvn90kjmevrkpew` | `lax` | Production (= `app.peepsai.com`) |

Trigger.dev's **Development** environment runs on developer laptops and uses local Docker Postgres — it does NOT traverse this proxy.

### File layout

- `fly.dev.toml` / `fly.prod.toml` — per-env Fly app configs.
- `Dockerfile.dev` / `Dockerfile.prod` — per-env Dockerfiles. They differ only in which `ip-whitelist.*.txt` file is `COPY`'d into the image. The whitelist is baked at build time, so deploys are how you change it.
- `ip-whitelist.dev.txt` / `ip-whitelist.prod.txt` — HAProxy ACL source files. Currently both contain the Trigger.dev us-east-1 static egress CIDR (`5.60.65.64/26`).
- `haproxy.cfg` — unchanged from upstream. Routes proxy `:5432` → `direct.$CLUSTER_ID.flympg.net` (direct, prepared-statement-safe) and proxy `:6432` → `pgbouncer.$CLUSTER_ID.flympg.net:5432` (pooler).

### Deploying

Both apps must already exist:

```sh
fly apps create peeps-fly-mpg-proxy-dev -o peeps-labs-inc
fly apps create peeps-fly-mpg-proxy     -o peeps-labs-inc
```

…and have `CLUSTER_ID` set as a Fly secret. Then:

```sh
# Dev
fly deploy -c fly.dev.toml -a peeps-fly-mpg-proxy-dev

# Prod
fly deploy -c fly.prod.toml -a peeps-fly-mpg-proxy
```

Naming convention matches the parent peeps repo: prod gets the bare name, dev gets the `-dev` suffix (mirrors `peeps` / `peeps-dev`).

### Trigger.dev DATABASE_URL

Use proxy port `5432` (direct), not `6432` (pooler). Peeps' tasks use `postgres.js` with prepared statements, which break under transaction-mode pgbouncer pooling. The pooler port is exposed as a fallback.

```
postgresql://peeps_trigger_dev:<PW>@peeps-fly-mpg-proxy-dev.fly.dev:5432/<DB>?sslmode=require
postgresql://peeps_trigger_prod:<PW>@peeps-fly-mpg-proxy.fly.dev:5432/<DB>?sslmode=require
```

DB user passwords live in the Peeps Labs 1Password vault under `peeps_trigger_dev (Fly MPG)` and `peeps_trigger_prod (Fly MPG)`.

### Updating the IP allowlist

1. Edit the relevant `ip-whitelist.{dev,prod}.txt`.
2. Re-deploy that env (`fly deploy -c fly.dev.toml -a peeps-fly-mpg-proxy-dev` or `fly deploy -c fly.prod.toml -a peeps-fly-mpg-proxy`).

The whitelist is baked into the image; there's no live reload.

### Syncing from upstream

Upstream (`fly-apps/fly-mpg-proxy`) is mostly dormant. Sync manually when needed:

```sh
git remote add upstream https://github.com/fly-apps/fly-mpg-proxy.git  # one time
git fetch upstream
git log --oneline HEAD..upstream/main
git diff HEAD..upstream/main
# Cherry-pick or merge selectively if anything looks relevant.
```

---

## How to install Fly MPG Proxy

Installing Fly MPG Proxy is as easy as installing any other app on Fly!

Find the Cluster ID for your MPG Cluster by running `fly mpg list`.

Run the following commands:

```
mkdir mpg-proxy
cd mpg-proxy
fly launch --from=https://github.com/fly-apps/fly-mpg-proxy --secret CLUSTER_ID=Your-Cluster-ID
```

### Follow The Prompts

You will see the following prompt:


```
An existing fly.toml file was found
? Would you like to copy its configuration to the new app? y/N
```

Type `Y` and press `[Enter]`.

---

You will see information about the apps configuration and the following prompt:

```
? Do you want to tweak these settings before proceeding? (y/N)
```

Press `[Enter]`.

---

You will see a prompt asking to allocate a dedicated ipv4 and ipv6 address.


```
? Would you like to allocate dedicated ipv4 and ipv6 addresses now? (y/N)
```

Type `Y` and press `[Enter]`.

### Launched!

After these prompts, your app should be launched.  You'll see a message like the following:

```
Visit your newly deployed app at https://mpg-proxy-winter-flower-8923.fly.dev/
```

## How to Connect

Now that the Fly MPG Proxy is up and running, you can connect to it.

Get the Connection URL from the MPG Panel.

1. Go to the [Fly Dashboard](https://fly.io/dashboard/) and click *Managed Postgres*.
2. Click the name of your MPG Cluster.
3. Click *Connect*
4. Locate *Connect directly to your database* and click the Eye symbol on the right to reveal the password.
5. Copy the Connection URL, for example `postgresql://fly-user:password@direct.9g6y30w23lk0v5ml.flympg.net/fly-db`

Replace the hostname section of this connection URL, for example my URL is `direct.9g6y30w23lk0v5ml.flympg.net`, and I would replace that with `mpg-proxy-winter-flower-8923.fly.dev`.

Now I can connect with this Connection URL:

```
$ psql postgresql://fly-user:password@mpg-proxy-winter-flower-8923.fly.dev/fly-db
psql (17.6, server 16.8 - Percona Distribution)
Type "help" for help.
```

This will connect directly to the Postgres DB.  To connect to the PGBouncer Pool, connect on port `6432`:

```
$ psql postgresql://fly-user:password@mpg-proxy-winter-flower-8923.fly.dev:6432/fly-db
psql (17.6, server 16.8 - Percona Distribution)
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off, ALPN: none)
Type "help" for help.

fly-db=>
```

## IP Restrict Connections

In this default configuration, anyone on the Internet could connect to the Managed Postgres instance and try to guess the username and password to login.

Fly MPG Proxy can be configured so that only certain IP addresses can connect.

Open the file `ip-whitelist.txt` and add the IP addresses or CIDR notation for networks that you would like to be able to access the Fly Managed Postgres Cluster.  Make sure to comment out or remove the entries for `0.0.0.0/0` and `::0/0`.

Once you have this file populated with only the IP addresses and networks you would like to allow to connect, run `fly deploy` to update the server with your new whitelist.

## Support

MPG Proxy is a standalone application that you may install on fly.io to gain external access to MPG Clusters.  It is not a part of the Managed Postgres service and is not supported by fly.io.

## Cheers!

Thanks for checking out this project!
