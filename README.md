# self host nextcloud: What It Really Costs, How Big a Server You Need, and a Docker Compose Setup That Covers HTTPS, Cron, and Backups

To self host Nextcloud, you need one thing the big cloud providers will never hand you: a machine where your files, calendar, and contacts live under your own control. Everything else breaks down into three practical decisions — how much server to rent, what it costs per month, and which deployment method you won't hate maintaining six months from now. The short version: a single user runs fine on a 1 GB VPS with swap enabled (currently $11.00/mo on Vultr's entry plan), a household wants 2 GB, and the setup most current walkthroughs have converged on is five Docker containers with Caddy handling HTTPS.

> **In a hurry?** Single user: the 1 GB Cloud Compute plan at $11.00/mo plus a 2 GB swapfile. Household or growing photo library: 2 GB. 👉 [Check Vultr's current Cloud Compute pricing](https://www.vultr.com/pricing/?ref=7268283) — every plan bills hourly with a monthly cap, so you can deploy the full stack on a Saturday morning and destroy it by lunch if Nextcloud turns out not to be your thing.

## What you're actually signing up for

Nextcloud is an open-source cloud platform (AGPL-3.0 licensed) that covers Dropbox-style file sync, calendar, contacts, notes, and — with the right apps — collaborative document editing. The software itself costs nothing. What you pay for is the machine it runs on, and what you take on is the operational side: updates, monitoring, backups, and the odd 3 a.m. breakage that no support ticket will fix for you.

For most people that trade is worth it. File versioning, granular sharing controls, and optional end-to-end encryption come standard instead of sitting behind a subscription. But go in clear-eyed about one thing: a self-hosted cloud is the service where data loss hurts most, so the backup section below is not optional reading.

## How much server Nextcloud actually needs

RAM decides everything here; CPU mostly idles unless you enable office document editing. Size by file count and user count together:

| What you're storing | RAM floor | What that looks like |
| --- | --- | --- |
| Up to ~100 files | 512 MB | Testing, light note sync |
| 100–10,000 files | 2 GB | Where most households land |
| 10,000–100,000 files | 4 GB+ | Redis caching stops it crawling |
| 100,000+ files | 8 GB+ | Dedicated PostgreSQL and Redis, plus a reverse proxy |

User count pushes in the same direction. One to five users are comfortable on 2 vCPU and 4 GB even with a photo library in the mix; Nextcloud Office or Collabora is what justifies 8 GB. On the small end, a single user with a modest library can live on 1 GB — but only once swap exists, because a burst of uploads on a swapless box can get PostgreSQL killed mid-write by the out-of-memory handler.

Adding 2 GB of swap takes a minute:

shell
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab


Swap absorbs spikes; it doesn't rescue an undersized server. If the machine swaps constantly during normal use, uploads and preview generation will crawl. That's a sign to move up a tier.

## Where to rent the server: where Vultr fits, and where it doesn't

Vultr's Cloud Compute line suits this workload in three ways. Billing is hourly with a monthly cap, which makes a "will this even work for me?" experiment cheap — the entry plan works out to roughly 37 cents a day. Instances deploy in 30+ datacenter locations, and for a sync-heavy service like Nextcloud, picking one near your household genuinely cuts latency. There's also a one-click Nextcloud app in Vultr's Marketplace if you want to see the interface before committing to a Compose stack.

Current Cloud Compute pricing:

| Plan | vCPU / RAM / Storage | Bandwidth | Monthly price | Deploy |
| --- | --- | --- | --- | --- |
| Cloud Compute 1 GB | 1 / 1 GB / 25 GB SSD | 1 TB | $11.00/mo | [ View 1 GB pricing](https://www.vultr.com/pricing/?ref=7268283) |
| Cloud Compute 2 GB | 1 / 2 GB / 55 GB SSD | 2 TB | $22.00/mo | [ View 2 GB pricing](https://www.vultr.com/pricing/?ref=7268283) |
| Cloud Compute 4 GB | 2 / 4 GB / 80 GB SSD | 3 TB | $44.00/mo | [ View 4 GB pricing](https://www.vultr.com/pricing/?ref=7268283) |
| Cloud Compute 8 GB | 2 / 8 GB / 160 GB SSD | 4 TB | $88.00/mo | [ View 8 GB pricing](https://www.vultr.com/pricing/?ref=7268283) |
| Cloud Compute 16 GB | 4 / 16 GB / 320 GB SSD | 5 TB | $176.00/mo | [ View 16 GB pricing](https://www.vultr.com/pricing/?ref=7268283) |

All sizes bill hourly, capped at the listed monthly rate, and larger tiers (32–128 GB) follow the same ladder on the pricing page.

The honest comparison: on raw specifications per euro, Vultr loses to the budget European hosts. netcup's VPS 1000 G12 — the box a recent Serverküche Nextcloud tutorial runs on — delivers 4 vCPU, 8 GB RAM, and 256 GB NVMe from €10.36/mo, and one current self-hosting guide points to Hetzner's CX21 at €7/mo as the budget pick. If everyone using your cloud lives in Europe and you're happy with fixed monthly billing, those are cheaper machines, and there's no way around that.

Where Vultr earns the difference: hourly billing with no lock-in, datacenters on five continents, instant resizing when the household grows, and optional automatic backups that add 20% to the instance price. DigitalOcean's matching 1 GB droplet checks in at $13.20/mo on the most recent pricing check, so at the entry tier Vultr is the cheaper of the two big-name options. 👉 [Compare the 1–8 GB tiers on Vultr's pricing page](https://www.vultr.com/pricing/?ref=7268283) if you want to sanity-check the numbers yourself.

## The five-container Docker Compose setup

The stack behind the most thorough current walkthrough is the same five services on one Ubuntu 24.04 box:

1. nextcloud:apache — the official image with web server and PHP bundled, so there's no separate PHP-FPM container to wire up
2. PostgreSQL 16 — holds metadata, on a named volume
3. Redis 7 — cache plus transactional file locking; setting REDIS_HOST and REDIS_HOST_PASSWORD makes the app configure locking on its own
4. A second copy of the Nextcloud image running only /cron.sh — dedicated background jobs
5. Caddy — terminates TLS, obtains certificates automatically, serves HTTP/3; the only container with published ports (80 and 443)

PostgreSQL and Redis get no published ports at all. That detail matters more than it looks:

> Docker publishes ports by writing its own iptables rules, which ufw never inspects — a published port stays reachable no matter what your firewall shows. Not publishing the database in the first place is the real protection.

A few more traps the tutorials tend to skip:

- **Heredoc quoting.** When generating the `.env` file, leave the EOF marker unquoted so the openssl commands inside actually run. When writing `compose.yaml`, quote it as `'EOF'` so ${POSTGRES_PASSWORD} lands in the file literally for Compose to expand later. Get this backwards and you'll either store unexpanded variables or leak a command substitution.
- **Fix the Docker subnet.** The stack pins a fixed subnet (172.28.0.0/16 in the walkthrough) precisely so TRUSTED_PROXIES can name Caddy instead of trusting a guessed Docker IP range. If that range collides with your network — check with `ip route` — pick another private /16 and update both places.
- **DNS before first boot.** Caddy orders its certificate via the ACME HTTP challenge on port 80. If the domain's A record doesn't already point at the server, the certificate order fails and Caddy backs off for minutes. Run `dig +short cloud.example.com` before you bring the stack up.
- **Think twice about All-in-One.** Nextcloud's AIO package brings its own TLS and its own ports and wants to handle the reverse-proxying itself. Behind an existing Caddy or Traefik setup, the classic nextcloud image is the controllable choice.

Secrets live in a `.env` file (chmod 600 it) generated with `openssl rand -hex`, which keeps three passwords out of any compose.yaml you might later paste into a ticket. Copy the admin password somewhere safe when you generate it — resetting it later means running `occ user:resetpassword admin` inside the container.

## First-boot settings people get wrong

Three settings cause most of the post-install pain, and all three take a couple of minutes to fix.

Without overwriteprotocol set to https, Nextcloud behind any HTTPS proxy enters a redirect loop. Set it once:

shell
docker compose exec -u www-data app php occ config:system:set overwriteprotocol --value="https"


Accessing the instance from a domain that isn't in trusted_domains triggers the "Access through untrusted domain" error. Add domains as they come up:

shell
docker compose exec -u www-data app php occ config:system:set trusted_domains 2 --value="new-domain.com"


Background jobs deserve a check too. The dedicated cron container should be handling them; verify with `occ background:cron` and confirm the admin panel shows "Cron" under Administration → Basic settings. Leave the default AJAX scheduler in place and file scans, notifications, and cleanup only run when someone happens to load a page.

While you're in settings, add SMTP credentials. Without them, password-reset emails don't send — a problem you'll discover at exactly the moment you need them.

## Backups: capture both halves

A Nextcloud install has two halves, and a backup that grabs only one restores an empty cloud. You need the PostgreSQL dump and the Nextcloud file volume together, and the copies need to live somewhere that isn't the server itself — that's the whole point of having them.

Vultr's optional automatic backups (+20% of the instance price, toggled at deploy time) work as a first layer against a dead disk, but they're snapshots of the same machine, not off-server archives. A nightly pg_dump plus a restic or rclone job pushing the Nextcloud volume to cheap object storage covers the rest. Test a restore once before you trust any of it. 👉 [Open Vultr's Cloud Compute plans](https://www.vultr.com/pricing/?ref=7268283) and tick automatic backups when you deploy — it's the cheapest insurance you'll add to this stack.

## FAQ

### How much RAM do I need to self host Nextcloud?

512 MB handles a handful of files, 2 GB covers most households with libraries up to about 10,000 files, and past that plan for 4 GB with Redis caching. Large photo collections (100k+ files) want 8 GB with dedicated PostgreSQL and Redis.

### Can you self host Nextcloud for free?

The software is free; the hosting usually isn't. An old laptop or single-board computer can run it, but an always-on rented VPS with a stable address is what makes mobile apps and background sync reliable. At $11/mo for the entry tier, reliability is what the money buys.

### Is the one-click app good enough, or should I use Docker Compose?

The Marketplace app gets you a working Nextcloud in minutes and is fine for kicking the tires. The Compose stack gives you a dedicated cron container, explicit port control, and a configuration you can version and rebuild anywhere. If you plan to keep this long-term, start with Compose.

### Do I need a domain name?

For anything beyond local testing, yes. Caddy's automatic HTTPS needs a resolvable A record, and the mobile apps behave far better against a proper domain with valid certificates. Open ports 80 and 443; nothing else needs to face the internet.

### How long until I can actually use it?

Roughly an hour for the Compose route, including reading time. The long pole is the first sync: pushing tens of thousands of files from your devices can take hours, so start it before dinner, not before bed.

## Picking a plan, and two last checks

Match the plan to your situation:

- Just curious — 1 GB, hourly billing, destroy it when the experiment ends. Try the Marketplace one-click first if Compose feels heavy.
- Single user, modest library — 1 GB plus the swapfile. Don't pay for 4 GB you won't use.
- Household, photos, ten years of documents — 2 GB to start, 4 GB once the library passes ~10,000 files or someone insists on collaborative office editing.
- Max RAM per euro, everyone in the EU — the netcup and Hetzner tier of hosts wins that specific race on paper. You give up hourly billing and location choice to get it.

Two checks before the first boot: confirm the domain's A record resolves to your server (`dig +short cloud.example.com`), and confirm ports 80 and 443 are reachable. Then 👉 [create your Vultr account and deploy the 1 GB plan](https://bit.ly/VulTR) — if the household joins later, a resize is a few clicks and a reboot, not a migration.
