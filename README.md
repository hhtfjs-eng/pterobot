# PteroBot — Pterodactyl Discord Bot with AI

A complete, production-ready Discord bot that manages your **Pterodactyl Panel** servers — **100% inside Discord**. No website, no dashboard, no web server. Everything happens through slash commands, buttons, dropdowns and modals, with a **NaraRouter-powered AI assistant** that can chat *and* control your servers through a strictly whitelisted action system.

**v1.19.1 — PROBE INSIGHT: a red node now explains itself.** The v1.19.0 board probes each node's wings daemon directly (the panel API has no "online" flag), which is honest — but "n1 — Unreachable" never said *why*, and a node whose wings is merely firewalled from the bot host got the same red dot as a genuinely dead one. Three changes: **(1)** every failed probe is now **classified and printed** on the board, on `/nodes` and on the node detail — `port 8080: connection refused — the daemon port is closed or wings is down`, `the node FQDN does not resolve from where the bot runs`, `TLS certificate rejected — self-signed or scheme mismatch`, `HTTP 502 — the reverse proxy answered but wings behind it did not`, and so on (the FQDN itself is never shown). **(2)** A node whose servers are **RUNNING** is never called dead — only a live daemon can run servers, so it shows green **"Online (probe blocked)"** with a note (`probe blocked from the bot host • N running servers prove the daemon is alive — open the daemon port or set STATUS_PROBE=false to silence this`) instead of a false alarm. **(3)** Two new env knobs: **`STATUS_PROBE=false`** switches probing off entirely (nodes show Available unless in maintenance) and **`STATUS_PROBE_TIMEOUT`** (500-15000 ms, default 2500) tunes the per-probe budget for slow networks.

**v1.19.0 — THE LIVE STATUS BOARD: /status is now a self-updating network status page.** Run `/status` once and a hosting-page-style board appears: **overall network RAM/Disk bars**, a **per-node breakdown** with usage bars and `used • free • total` lines, **nodes available** and **servers running** counts in the header — and it **edits itself every 5 minutes** (`STATUS_REFRESH_MINUTES`, clamped 5-60): a DB-driven sweeper re-renders the message, so the numbers are never stale and the board **survives bot restarts** (the same restart-safe pattern as giveaways). The board carries two buttons — **🔄 Refresh now** (force a re-render) and **⏹ Stop auto-refresh** — usable by staff or whoever posted it. Posting `/status` again in the same channel **supersedes** the old board (its buttons freeze), each channel keeps one live board, and boards **retire automatically** after 24 hours or an hour of unreachable panel instead of hammering forever. Node reachability comes from real wings-daemon probes, RAM/Disk numbers are per-node allocations summed across the network, and FQDNs are never shown — just the short node labels.

**v1.18.0 — THE CLIMB ENGINE: the half-built-ruin era is over.** The v1.17.0 field report: "creative mode — pulling 8 kinds of material straight out of the creative inventory" worked, materials were ready — then "66 blocks placed, 82 failed" and the screenshot showed a foundation with sparse walls and one lonely roof beam. Root cause: the engine judged "close enough" by **raw distance to the block centre** — a roof block two up is "in reach- of the ground while **every placement ray was blocked by the walls below it**, so the bot never moved and the server refused packet after packet (and the click points were mirrored to the far side of the target cell, so even the face scoring judged the wrong side). v1.18.0 rebuilds the decision layer: every placement now needs a face that is **in reach AND visible** — a fine-grained **line-of-sight ray** from the eye to the real shared-face click point. When nothing is visible the **bot climbs the build itself**: validated stances (support under, air feet, **air headroom** — the v1.17.0 stance list never checked headroom, so those walks timed out) on wall tops and the previous roof course, reached with no-dig pathfinding — exactly how a player walks a roof course. A **jump-assist pass** clears wall-corner grazes from the ground (the apex eye gains ~1.2 blocks — with a plausibility filter: a top face below the apex is never tried, so roofs skip straight to climbing). Spots with no natural stance get a **temporary cobblestone scaffold ramp** built beside them, climbed, used, and **torn back down when the build finishes** (survival gathers a +24 cobble reserve and digs it back out; creative pays nothing). Standing inside the footprint now **steps aside one cell at the same height** instead of hiking to the ground. Result: complete builds — pitched roofs, ridges, glass, torches, the lot.

**v1.17.0 — BUILDS THAT ACTUALLY LOOK + CREATIVE MODE + IT FIGHTS BACK.** Three headline changes. **(1) The placement engine is rebuilt** — the "36 placed, 87 failed" / "0 placed, 24 failed" era is over: *foundation fills* probe the ground under every column of the plan before gathering (a wall over a dip used to float with nothing to place against), the *stance engine* walks to exact standing spots (beside the block at its level, or one above to place downward like a real roofer) with radius-1 goals instead of "somewhere within 2 of above the target", every placement **aims at the face actually facing the bot** and **waits for the bot to stand still** (mid-slide packets were being dropped), missing items fail fast instead of burning 12 sleeps, failed spots get a **retry pass**, and interior **rooms are hollowed before building** — hillside sites used to leave solid-dirt "houses" (the "random blocks" look). The **cozy house is a real house now**: oak-log corners, plank walls, glass windows, a working door, a **pitched plank roof**, porch step + torch, **crafting table + furnace inside**, and a straight staircase to the roof; materials are **derived from the blueprint itself** so the counts can never drift. **(2) Creative mode** — flip the bot to creative (gamemode command / panel console) and `/bot build` skips the whole mine-chop-smelt-craft chain: every material is pulled straight from the **creative inventory** into the hotbar and replenished as it is spent. Houses in seconds, zero gathering. **(3) Self-defense** — zombies, skeletons, spiders or **players** that hit the bot get its best weapon swung back at them: chase, 1.9 attack-cooldown timing, creeper retreat (it backs off before the blast), fight-then-resume. A scuffle **pauses** a running build instead of derailing it. `/bot combat on|off` toggles it (default on); `/bot status` shows it.

**v1.16.1 — GATHERING IS NOW VERIFIED: the real "planks came up short (0/16) — not enough logs" fix.** Root cause: the collector counted **blocks swung at**, not **items that reached the inventory** — chopped logs whose drops were missed still counted as "success", so the bot "gathered 6 logs" with an empty inventory and `/bot build cozy house` died at the very first step. Five fixes: **(a)** every gather loop (logs, stone→cobblestone, coal, sand) now targets the **inventory count** — lost drops just mean the bot chops another tree, and a dig-spree with an empty inventory ends in a clear "drops are being lost" error instead of a silent fake success; **(b)** the drop vacuum no longer surrenders ALL drops when ONE drop is unreachable (radius-0 goals were unreachable for drops against trunks/leaves — it now skips that drop and keeps collecting the rest); **(c)** trunk-top drops fall for ~1.5 s — the vacuum now lets them land before walking at mid-air coordinates, and gets a longer budget when items are the goal; **(d)** phantom "already gone" blocks (stale references after chunk reloads) are never counted as progress, and floating trunk tops left over after a chop sort LAST so the next real tree wins instead of burning a 20 s pathfind timeout; **(e)** the wooden-pickaxe step skips chopping entirely when carried planks + logs already cover the chain, and the planks error now says exactly where the chain stopped. `/bot build` and `/bot diamonds` gather-for-real now.

**v1.16.0 — CRAFTING ACTUALLY WORKS, /giveaway, broken-plugin auto-fix, creation sessions that survive restarts.** (1) **The real "crafting the stick kept failing — the server did not confirm the craft" fix.** Root-caused to THREE distinct bugs: **(a)** `bot.recipesFor` was called with the crafting table in the WRONG argument slot, so every 3×3 recipe (the wooden pickaxe!) was filtered out — "missing the materials" even with a table in front of the bot; **(b)** mineflayer's `bot.craft()` fires its whole 5+ click sequence back-to-back with zero pacing and zero verification (it even optimistically writes the expected result into the output slot locally and clicks it blindly) — on laggy panels the server silently rejects mid-sequence clicks (stale stateId), the grid never completes server-side and the crafted items never exist; **(c)** a full inventory makes the library TOSS the crafted item on the ground. The new craft core does every click itself: **paced clicks** (the server's stateId is always fresh), **waits for the server's own set-slot acks**, only takes a result the server is **actually offering in the output slot** (100% server-authoritative — the code never writes slot 0), cleans the **grid + cursor + window** on every attempt, guarantees **inventory room first (junk is tossed)**, and retries with growing spacing — so a rejected click is RECOVERED, not fatal. (d) **Planks now craft from the wood type you carry the MOST of** — every shaped recipe needs 2-4 planks of the SAME kind, and a mixed 4+4+4+4 inventory could not even shape a crafting table (the error now says exactly that). `/bot build cozy house` and `/bot diamonds` finally play the whole logs → planks → sticks → pickaxe chain themselves. (2) **`/giveaway`** — start (prize + duration + up to 20 winners + channel), a **Join button** everyone can toggle, winners **drawn automatically** when time is up, `/giveaway end` (draw now), `/giveaway reroll` (draw again) and `/giveaway list`. Fully **database-persisted: a bot restart never loses an active giveaway** (a 15 s sweep re-reads the DB, exactly like auto-suspend). (3) **Broken-plugin auto-fix (the ViaForwards.jar error)** — `Could not load plugin 'ViaForwards.jar' in folder 'plugins/.paper-remapped'` means the jar's build does not match your server version. **`/plugins check`** scans the log, lists the jars the server refused to load and removes them **in one click**; the AI can also do it (just ask — "fix the viaforwards error"), and the reply explains that ViaVersion + ViaBackwards are the correct combo for cross-version clients. (4) **Creation sessions survive restarts** — "noctis create a server" sessions are now stored in SQLite (a bot update no longer kills a creation mid-flow), the window is 30 minutes (was 15, and the `/server-create` wizard too), and clicking a menu whose session is gone **auto-restarts the flow with a fresh egg menu** instead of the old dead-end "say noctis create a server again". **69 commands (+/giveaway).**

**v1.15.0 — `/bot build <anything>`: FROM-SCRATCH BUILDER.** Say what you want in your own words — "**xp farm**", "**iron farm**", "**house**", "**tree farm**", "**bridge**", "**wall**", "**shelter**" — and the bot builds it **completely from scratch**: it checks its inventory, then **plays the whole supply chain itself** (chop logs → planks → sticks → craft a wooden pickaxe if needed → mine stone for cobblestone → collect sand → smelt glass in a furnace → punch sheep → craft wool beds → mine coal or make charcoal → craft torches), and only THEN walks the site **placing every block bottom-up**: floors first, walls layer by layer, roof last — with a **permanent outside staircase** on tall builds so both the bot and you can reach the top. Real engineering went into making this reliable with a plain mineflayer bot: **soft floors** keep existing terrain (a flat grass clearing is already a floor — no pointless dig-and-refill), **walking during a build never digs** (a digging pathfinder would chew through the very walls it just placed), the bot **steps out of any spot it is standing in** before placing (never entombs itself), every placement tries all six reference faces and is **verified to have landed**, `/bot stop` cancels cleanly mid-build, and **re-running the same build CONTINUES where it left off** — already-correct blocks are skipped, so a canceled house is finished by just running `/bot build house` again. The blueprints: a **cozy 7×5 house** (cobble corners, glass windows, working door, flat roof), an **emergency 3×3 shelter**, a **dark-room XP tower** (spawning room 13 blocks up over an open drop shaft into a walk-in collection chamber — stand inside and one-punch the mobs that fall), an **iron-golem platform** (7×7 elevated pad with 3 bed pads + outside staircase — bring 3 villagers and free iron starts), a **fenced tree farm** with 9 planted saplings, a **16-block bridge** in the facing direction, and an **8×3 wall**. Missing items? **Drop them next to the bot and re-run** — it picks them up and uses them. Autocomplete lists every blueprint; fuzzy matching means "make me an xp farm" just works. **Still 68 commands.**

**v1.14.0 — the bot crafts for real, never mines bedrock, digs ~2× faster.** (1) **Placing the crafting table can no longer kill the quest** — the old code lost the whole quest to ONE failed placement (a single "Server refused to place" — common on laggy panels — escaped the placement loop instantly, and the craft's table-fetch sat outside its retry loop: that is why the bot "couldn't place the crafting table and make pickaxes"). Now every one of the 8 placement spots gets its own error guard, the placement is **verified** afterwards, the bot **relocates to flatter ground and retries** (up to 3 rounds), craft retries cover the **whole chain** (table find → walk → craft → place → craft clicks), a **fresh table** is placed at the bot's feet from attempt 2, and the found-table path re-reads the block after walking (stale references made craft clicks misfire). (2) **BEDROCK IS NEVER MINED** — undiggable blocks (bedrock, barriers, command blocks) are refused **before the first swing**: `digTime` is Infinity for them, so the old bot stood at the world floor swinging at bedrock forever ("in Y-58 it was mining bedrock"). The descent now **stops at the floor** and branch-mines at that depth instead, reporting the depth it reached. (3) **~2× faster descent** — blocks are cleared **two at a time** when it is safe (2-block falls deal zero damage), with tighter fall polls. (4) **The furnace now takes EVERY output** — the old loop grabbed the first batch only, so 3 raw iron could yield 1 taken ingot and the iron pickaxe craft failed on "missing materials". (5) **HANDOFF: drop items next to the idle bot and it picks them up** — craft a pickaxe yourself, toss it over, the bot grabs it (idle vacuum every 9 s); **ingots you give the bot count toward the quest and skip smelting**, and the fuel list finally matches real plank names. **Still 68 commands.**

**v1.13.0 — the Minecraft player bot FIXED + FAST.** (1) **`/bot diamonds` can no longer die on "invalid operation"** — the quest's crafting is rebuilt: every craft retries 3× across recipe variants, resets any stuck window, walks next to the crafting table before using it (servers reject table use from >~6 blocks away), and **verifies the result landed in the inventory** before moving on; planks are crafted to the real need (16, not 8 — the old count was short of table + sticks + pickaxe, so the quest ran out mid-way and the failed craft click threw). (2) **"could not reach the oak log: Took to long to decide path to goal!" is gone** — the pathfinder's think timeout went from 5 s to **25 s** with 80 ms of thinking per tick (~2× faster path computation), goals use a near-radius instead of exact-block (walking *next to* the tree instead of *onto* it), blocks that genuinely can't be reached are **blacklisted and skipped** instead of failing the whole quest, and after 3 wanders the bot reports a clear, actionable error instead of hanging. (3) **The whole tree from ONE stop** — trunk mode chops every log within reach of the trunk base from a single position (never digging the floor under its own feet), and a built-in vacuum walks onto dropped items so logs never rot on the ground. (4) **The bot is visibly FASTER**: sprint + parkour movements, a ~30% movement-speed boost, ~2× faster path computation, and the **best tool auto-equipped for every dig** (stone is no longer punched by hand at ~6× the time — the whole stone → furnace → iron chain got faster). (5) **`mine oak_log` no longer glitches** — the line-of-sight goal that made the head twitch next to trees is gone, and the idle look-around never fires mid-swing while the bot is working (shown as "⚒️ working" in `/bot status`). The block collector is now built in-house end-to-end: **walk to reach → best tool → dig → vacuum**. **Still 68 commands.**

**v1.12.0 — /bot join FIXED (every panel, every version), version selector, AI channel, AI reads server files.** (1) **The player bot can join YOUR servers again** — the old join read the allocation's raw IP, and panels that list `0.0.0.0:<port>` (wildcard binding, no alias) made it dial `0.0.0.0` → instant `ECONNREFUSED` ("bot cant join servers"). The address is now resolved like a real client: **alias → real IP → node FQDN → panel host**, each one SLP-probed, and the first address that completes the Minecraft handshake wins — wildcard allocations can never break the join again (and `/ping`, `/list` and auto-detection got the same fix). The join reply now shows the real `host:port` and how it was resolved, and a failed join no longer double-posts a confusing "left the server" message. (2) **Join every version of Minecraft**: the server reports its exact version in the handshake, and the bot maps it onto a version mineflayer speaks — **by protocol number** (two labels sharing a protocol are wire-compatible: 1.21.7 → 1.21.8, 1.8.8 → 1.8), then closest-family (26.2 → 26.1), with the stand-in named in the reply — covering **1.8.8 through 26.1** out of the box. (3) **VERSION SELECTOR in `/server-create` and `noctis create a server`**: Minecraft eggs now ask **which version to install** (egg default, or any curated release from 1.8.8 to 1.21.11) — the choice is written into the egg's version variable (`MINECRAFT_VERSION`, `MC_VERSION`, `VANILLA_VERSION`… detected automatically), shown in the specs step and the creation result. (4) **The AI CHANNEL**: staff say **`noctis ai channel on`** in any channel and from then on the bot answers **EVERY message there** — no wake word, no ping, no `/ai` — with the full engine (memory, live diagnosis, auto-fix, panel actions). `noctis ai channel off` (said anywhere) disables it; `/settings` shows it. (5) **AI reads SERVER FILES**: say **`list files`**, **`read server.properties`**, **`show my logs`** (or any file-ish question) in `/ai`, `/smart`, a ping, `noctis …` or the AI channel — the REAL file tree/contents are pulled from the panel into the AI's context, so answers quote actual settings instead of guessing. (6) **`/plugins run` takes in-game `/commands`** — type the command WITH its leading slash (`/list`, `/essentials:kick Notch`) and the slash is stripped for you instead of being rejected. (7) **`/install` never dead-ends**: version-family matching (a build tagged 1.21 installs on 1.21.4 and vice versa, protocol-equal siblings preferred) and, when the server version/type cannot be detected, you now **pick them from menus** (version selector + loader selector + typed custom version) instead of being told to re-run the command. **Still 68 commands.**

**v1.11.0 — one ticket category, ticket categories, UNLIMITED spam + `noctis stop spam`.** (1) **Tickets live in ONE shared category** — the old code could end up with a fresh "Tickets" category per ticket (lost setting id / host migration / cache miss). Resolution is now bulletproof: the stored `ticket_category_id` is re-verified (cache **and** API fetch), an existing "Tickets"-style category is reused when the setting is lost, empty duplicate leftover categories are silently cleaned up, and only Discord's hard 50-channel cap ever spawns an overflow "Tickets 2" (so creation never fails). (2) **Ticket categories**: every ticket now carries a type — 🛠️ **Server Issue** • 💬 **General Support** • 💳 **Billing & Payments** • 🚨 **Report a User** • 📦 **Other**. The panel flow is pick-a-category (select menu) → subject modal; `/ticket create category:` picks it directly. The type prefixes the channel name (`server-0001`, `general-0002`…), shows in the header embed and transcripts, and is stored in the DB (old rows default to General Support). (3) **`/spam` has NO limit** — `count` goes up to 999999, and the new `forever:true` option spams **literally INFINITY** until stopped; big runs announce themselves ("🚀 Spamming FOREVER — say `noctis stop spam` to stop it"). (4) **`noctis stop spam`** — the new kill-switch trigger stops every running spam run (including forever ones) **and** every ping storm: staff and the requester stop everything, anyone can stop the storms pinging them; it works even with the voice-AI flag off, because stopping a storm is a safety action, not an AI feature. **Still 68 commands.**

**v1.10.0 — DM nuke, spam, sync servers, total nukes, offline-mode Minecraft bot.** (1) **`/dmnuke user:@member [limit]`** — wipes every message **the bot sent** in its DM channel with that user (up to 1000 scanned): staff can nuke any DM, everyone can nuke their own; the other person's messages are untouchable (Discord only lets you delete your own DM messages). (2) **`/spam message:<text> count:<n> [delay] [user] [channel]`** — staff tool that sends a message multiple times (up to 500, custom delay 300–10000 ms, optional real user ping in **every** message, optional target channel for admins). (3) **`/sync`** — sync servers from the panel: drops every cache, re-pulls the server list, auto-links your panel account, re-grants owned-server access and shows your servers instantly; `/sync all:true` (admin) re-grants for **every linked member**. (4) **`/nuke all:true`** — nukes **EVERY channel** in the server (admin + typed **NUKE ALL** modal confirmation) and leaves one fresh #general behind. (5) **`/servernuke`** — the TOTAL wipe: all channels, all deletable roles, all emojis, all stickers, optional **mass ban** of every member except owner/bots/you (admin + typed server-name confirmation; anti-nuke never punishes the bot itself, so these run clean). (6) **`/bot join auth:offline`** — the Minecraft player bot now joins **offline-mode servers with any username — no premium Minecraft account needed**; `auth:auto` (default) uses Microsoft auth only when `MC_ACCOUNT_EMAIL` + `MC_ACCOUNT_PASSWORD` are configured, `auth:microsoft` forces it (with a clear error when the env vars are missing). **68 commands.**

**v1.9.1 — /update reliability pass + UNLIMITED pings.** (1) **`/update` got a full self-repair engine** so it lands its edits far more often: (a) **whitespace-tolerant anchors** — an anchor that is right except for indentation/trailing spaces still applies (marked "≈ fuzzy match" in the result); (b) **2 automatic repair rounds** — when a plan fails to apply, parse or pass sanity, the exact errors (including the offending source window for syntax errors) go back to the editor AI for a corrected plan, no re-typing needed; (c) **JSON tolerance** — plans with `//` comments or trailing commas parse fine now; (d) **network resilience** — 429/5xx/dropped connections retry with back-off (honoring `Retry-After`), the timeout doubled to 4 minutes, and a `max_tokens` that a model rejects degrades gracefully (6000 → 4000 → 2000); (e) **stage-tagged errors** — every failure now says WHICH stage (locate / plan / repair / write) failed, shows a snippet of what the AI actually answered, and a disk-write failure (permissions) gets its own explanation; all inside a 12-minute budget so the Discord reply always arrives. (2) **The `noctis ping @user` trigger is UNLIMITED**: any number goes (`noctis ping @user 500 times`), and `noctis ping @user forever` (or `infinite` / `unlimited` / `endless` / `nonstop`) pings **literally forever** — with a **kill switch**: `noctis stop pinging` stops every running storm (staff and the requester can stop all; anyone can stop storms aimed at them), `/settings auto-ping off` kills them all instantly, and the bot also stops by itself if sends keep failing. Big storms announce themselves ("🚀 Pinging @user 500 times — say `noctis stop pinging` to stop it"). Also fixed: usernames ending in digits (`noctis ping john1234`) are no longer mangled by the count parser, and `once` / `twice` / `thrice` work as counts. **Still 64 commands.**

**v1.9.0 — a REAL Minecraft player bot, /history, "noctis create a server", bare "noctis".** (1) **`/bot`** drops an actual player into one of YOUR panel servers (address auto-resolved from the panel allocations) — powered by **mineflayer**: it walks around (`come` / `follow` / `goto`), **breaks blocks** (`mine block:oak_log count:4` — it picks the best tool), **places blocks** (`place item:cobblestone`), fights (`attack target:zombie`), chats (`chat hi!`), shows its inventory, drops/equips items, respawns and even **looks around idly** like a player waiting for something to happen. The star is **`/bot diamonds`** — the full tech-tree quest: it **chops trees → crafts a wooden pickaxe → mines stone → crafts a stone pickaxe + furnace → digs to diamond depth (Y -58, steering around lava) → collects + smelts iron → crafts an iron pickaxe → branch-mines until it finds DIAMONDS**, posting live progress in Discord. One bot per Discord server, controlled only by its starter + staff; whispers and name mentions in the Minecraft chat are relayed back. Needs the `mineflayer`/`mineflayer-pathfinder`/`mineflayer-collectblock` packages (the bot still boots without them — `/bot` then prints the exact npm command to fix it); **online-mode** servers need `MC_ACCOUNT_EMAIL` + `MC_ACCOUNT_PASSWORD` (real Microsoft auth). (2) **`/history`** — everything the bot remembers, on demand: `ai` (your AI conversation memory — the exact context `/ai` and `noctis <question>` see), `fixes` (executed panel actions & AI auto-fixes from the audit log), `updates` (admin: the `/update` self-edit history + the backups), `server` (one server's action timeline). (3) **`noctis create a server`** — say it and get a 3-click self-service creation: a select menu of eggs **ranked by optimization compatibility** (Paper ⭐ first, then Purpur, Spigot, Fabric, NeoForge… each with a compatibility blurb) → preset specs → name → created on the **least-allocated node**, auto-granted to your linked panel account — and the reply points you at **`/ai` or `/smart`** for everything after (optimize, plugins, anything). (4) **Bare `noctis`** — the wake word alone now answers with a quick hint, exactly like pinging the bot: one word and the bot is there; `noctis <question>` keeps behaving **exactly like `/ai` and `/smart`** (memory + live diagnosis + auto-fix). **64 commands.**

**v1.8.1 — /update now REMOVES and CHANGES too.** The self-editing command grew from "add things" into a full add / remove / change toolkit: (1) say `/update instruction:"remove /8ball"` and the editor AI **deletes the whole command** — its `// --- /name ---` banner plus the entire `COMMANDS.push({...});` block — via a new dedicated **`delete` edit mode**; (2) a mechanical **REFERENCES scan** lists every stray mention of the removed command elsewhere in the file (help text, `/settings` rows, component handlers, `module.exports`) so the editor cleans them too — a dangling reference to deleted code would crash the bot on boot, so nothing is left behind; (3) `/update action:` is a new option that **forces the operation** — `Auto` (default, detects from your wording), `Add`, `Remove` or `Change` — for when your instruction is ambiguous ("fix this", "get rid of that"); (4) the LOCATE stage now understands removal intent on its own ("delete 8ball", "get rid of the ship command") and the PLAN stage spells out the three playbooks (new command → insert at the marker; change → replace only the lines that change; remove → delete the block + clean the references); (5) safety rails grew with it: a single delete edit can never remove more than 1,200 lines, everything still passes `node --check` + structural sanity before writing, and every apply remains one-click **Undo**-able. **Still 62 commands.**

**v1.8.0 — /update: the bot that edits ITSELF.** Type `/update instruction:"add a /8ball command"` and the **editor AI** rewrites the bot for you — using a **SECOND, SEPARATE NaraRouter key** (`NARA_EDITOR_API_KEY` in `.env`) that is used ONLY for bot-editing and never touches the chat AI key (chat traffic can never burn the editor key's quota, and rotating either key leaves the other untouched; leaving it empty falls back to the main `NARA_API_KEY` with a ⚠️ warning in the reply). The pipeline is safety-first at every step: (1) **LOCATE** — the editor AI reads a compact map of the bot's own source (every command, function and section with line numbers) and decides what the request touches; (2) **PLAN** — it returns a strict JSON edit plan whose anchors are copied verbatim from real source excerpts; (3) **APPLY** — every anchor must match **exactly once** or the edit is rejected; (4) **VERIFY** — the result must pass `node --check` AND a structural sanity pass (VERSION/registry/login/exports intact, no duplicate command names); (5) **WRITE** — only then is `index.js` replaced, after a timestamped backup in `data/selfedit-backups/` (last 10 kept). Nothing that doesn't parse is ever written. Buttons on the result: **Restart now** (the bot exits so the panel reloads it), **Undo (restore backup)** and **Show full edits** (the raw JSON plan — as an attachment when it's big). `preview:true` shows the whole plan without writing a byte. Admin-only, one /update at a time, 8-edit limit per run, new commands are inserted at a dedicated marker so command registration always sees them, and `/settings` + the boot console show which editor key is loaded (dedicated vs shared, masked). **62 commands total.**

**v1.7.0 — the AI that fixes things, /afk, /announcement, /lvl add, auto-ping.** (1) **AI auto-diagnosis & auto-fix**: every AI path — `/ai`, `/smart`, `@Noctis …`, `noctis …` — first checks the **linked Pterodactyl user** on the panel, walks their servers, reads each one's **live state** (running / stopped / crashed / suspended, RAM %, CPU %) and injects the full diagnosis into the conversation. When something is fixable, the AI **fixes it in the same reply**: a stopped server gets `server_start`, a crashed one `server_restart` — executed immediately, no confirmation needed (destructive actions like kill/optimize still require the `/smart` confirmation buttons). Not linked yet? The AI tells you how to `/link` first. (2) **The "noctis ping @user" trigger**: say `noctis ping @user` and the bot pings that user in a **plain message — not an embed**; `noctis ping @user 10 times` pings them **10 times** (capped at 10); `him/her/them/me` or a bare username works too. Controlled by the new **`/settings auto-ping`** switch. (3) **`/lvl add user amount [xp|levels]`** — admins grant (or, with a negative amount, remove) XP or whole levels, with proper level-up announcements. (4) **`/afk [reason] [duration]`** — set yourself away; anyone who pings you gets your reason + how long you've been gone, and your next message auto-clears it with a welcome-back note. (5) **`/announcement title message [channel] [color] [ping] [role] [image]`** — staff broadcasts with a colored embed, optional `@everyone`/`@here`/role ping and image. (6) **All unit-conversion commands removed** (`/convert`, `/convert-*` — 23 commands, 500 tools) per the owner's request — the suite is now **274 tools in 17 suite commands, 61 commands total**. (7) **AI replies are plain messages now** — no embed, and the "🤖 DevAvoid AI" footer is gone from every AI answer.

**v1.6.1 — your animated emojis, everywhere.** (1) **`/status` no longer shows hostnames** — the node cards displayed the node's FQDN (e.g. `vitus-node1.devabyss.indevs.in`); that line is gone, and node names that are literally hostnames are automatically shortened to just the label (`vitus-node1`). (2) **`/help` got a modern redesign decorated with your OWN uploaded animated emojis**: a quote-block feature list, animated field headers, and every one of the 17 select-menu options gets an animated emoji when the guild has them — emojis are matched to categories **by name** (an upload called `nitroserver` lands on Servers, a `booststar` on stats), and anything left over is used round-robin so **every animated emoji you uploaded shows up**. Guilds with no animated emojis keep the classic static icons — nothing breaks. (3) **`/status` is now fully animated-emoji decorated too** — title, loading spinner, every field header, one animated emoji per node card and the totals row.

**v1.6.0 — community & AI control.** (1) **"noctis" voice triggers**: say **`noctis help him`** (or *her/them/@user*) anywhere — tickets, community channels, anywhere — and the AI **reads the recent chat history and replies**, helping that person; `noctis <question>` chats like a ping. Staff say **`noctis i will reply now`** and the bot **stops replying in that channel** until a staff member calls it back (`noctis start replying` / `noctis help …`). Only staff can pause/resume — members can only ask for help. (2) **Ticket AI auto-reply, staff-gated**: ON by default in ticket channels (including TicketTool-style ones) — the AI answers members automatically while staff are away; staff messages are never auto-answered; **only staff can enable/disable** it (per-ticket button, `/ticket ai toggle`, `/settings ticket-ai`). (3) **`/settings` is now the full control panel** — everything changeable from the command: ticket AI, ping AI, voice triggers, levels + level-up channel, log channel, ticket category, autolink. (4) **`/ip` "no allocations" fixed** — allocations are now read from every known response shape with an Application-API fallback, so the address always shows. (5) **`/status`** — animated panel overview: per-node RAM & storage bars (▰▰▱), servers hosted per node, panel totals. (6) **`/ping server:<name>`** now reports **min/avg/max of 3 probes**. (7) **Levels system**: 15-25 XP per message, level-up announcements, **`/rank`** cards and **`/leaderboard`**. (8) **Fun meters**: `/ship`, `/gayness`, `/lesbianess` (+ `/fun rates cute|simp|pp|iq|vibe`) — deterministic, the same person always gets the same score. (9) **All math commands removed** (`/math`, `/math-*`) per the owner's request — 82 commands total now.

**v1.5.0 — uploads fixed for real, live ping, mention AI, /timeout.** (1) **The installer's "The PUT method is not supported for route …/files/write" errors are gone**: the panel's write route is POST-only — every jar upload now uses POST, and **files bigger than ~768 KB go through `files/pull` instead**, meaning wings downloads them **directly from Modrinth/CurseForge into the server folder** (no request-body size limits at all — that's how the 20 MB FastAsyncWorldEdit jar gets in). After every run the installer **verifies** each file actually landed on the disk. The same audit found and fixed three more silent bugs in `/file rename` (wrong method AND body shape), `/file copy` and `/file delete` (wrong body shape) — verified against the panel's actual route definitions (v1.6 → v1.11). (2) **Auto-detect v3**: a running server is now asked **directly** — a real Minecraft Server List Ping over raw TCP returns the exact version the server reports, plus players and latency; offline servers fall back to `version_history.json`, rotated `.log.gz` logs and `world/level.dat` (NBT-parsed). Detection works even on a stopped, fresh Paper install. (3) **`/ping` shows the Minecraft servers' ping** — live latency, players, version and MOTD for one server (`/ping server:<name>`) or all of yours at once; `/list` answers instantly via the same status ping. (4) **Ping the bot with a message → it replies with the AI** (no `/ai` needed; `/ai`, `/smart` and every other command are unchanged — disable with `MENTION_AI_ENABLED=false`). (5) New **`/timeout user duration [reason]`** and **`/untimeout user`** top-level commands.

**v1.4.1 — auto-detect, fixed for real.** Two v1.4.0 bugs are squashed. (1) `/ip` and `/list` crashed with a "server option not found" error — the commands forgot to declare the `server` option their handlers read; both now take `server:` with autocomplete, exactly like `/optimize`. (2) `/optimize` said "Could not detect the server type / Minecraft version" on every server: the panel's `/startup` endpoint returns a different JSON envelope than the bot assumed, so its variables always read as *empty*. Detection was rebuilt end-to-end and now reads **six independent sources** — egg startup variables *and* the rendered startup command, root jar filenames (`fabric-server-mc.1.21.4-…`, `paper-1.21.4-…`, `forge-1.20.1-…`), `run.sh`/`@libraries` paths, `logs/latest.log`, the mods-vs-plugins folders with `libraries/` probes (so a Fabric build can never land in a Forge server), and finally the server's egg name via the admin API. If detection ever fails, the error embed now shows a **full report of what was tried** plus the exact command to fix it. Also fixed silently along the way: `/startup variables|set` (same envelope bug).

**v1.4.0 — the game-server upgrade.** Six new commands: **`/ip`** (the connection address), **`/ping`** (bot + panel latency), **`/list`** (live player list — sends the console `list` command and reads the log), **`/plugins list|run`** (auto-detects the plugins/mods folder and runs their console commands from Discord with output), **`/optimize`** (one click installs a curated performance pack from Modrinth — Lithium, ServerCore, Krypton, ModernFix, MemoryLeakFix, LazyDFU, Clumps, Chunky, spark, VMP, DashLoader, EntityCulling, LMD, RakNetify, LagFixer, FAWE + the Essentially-Optimized-Server modpack — auto-matched to the server's exact Minecraft version and loader, replacing old versions) and **`/install`** (search & install ANY Modrinth project — CurseForge too with a free API key). `/mod` gained `mute`/`unmute` aliases, and `/smart` now answers "my server lags" with both fixes: run `/optimize` or upgrade the plan.

**v1.3.2 — the 401 fix.** "/ai and /smart say: Something went wrong — NaraRouter API error 401: A valid API key is required" even though you set your key? That almost always means the key the bot LOADED is not the key you think you pasted — stale `.env` on the server, or paste damage (quotes, a line-wrap, invisible characters, a `Bearer ` prefix). The bot now **auto-cleans** the key at load, **verifies it against NaraRouter at startup** and prints the verdict with a **masked fingerprint** (`sk-abc…wxyz, 48 chars`) in the console; `/settings` shows the same fingerprint, and a 401 error embed now includes the fingerprint + a fix checklist so you can see exactly which key is loaded and fix it in seconds. Also new: `NARA_URL` env override.

**v1.3.1 — the sync fix.** Discord silently caps a single slash command at **8,000 characters**, and one oversized command makes the ENTIRE registration fail (that "Sync failed / Command exceeds maximum size (8000)" message — and the reason `/mc` & friends never showed up). The three giant hubs are now **split into one command per category**: `/convert-length`, `/convert-mass`… `/text-case`, `/text-style`… `/math-basic`, `/math-geometry`… plus a `/convert` / `/text` / `/math` index with a `tools` subcommand listing every command in that hub. Nothing else changed — same 933 endpoints, now they actually register.

**v1.3.0** pushed the utility suite past **900 real tools** (906 tools + 27 management commands = 933 command endpoints), turned `/help` into an **interactive command center** (category select menu with live per-hub counts), and gave every embed a polish pass — versioned footers, timestamps, state-colored server embeds with emoji field headers. New in the suite: zodiac & moon phase, world clock & timezone diff, color gradients & shade scales, Minecraft portal/beacon/MOTD/fuel tools, password & UUID generators, and a developer-humor section.

The whole bot is a **single `index.js`** — same style as the reference src it was modelled on. Copy it anywhere, `npm install`, `npm start`, done.

---

## Table of contents

1. [Requirements](#1-requirements)
2. [Creating the Discord application](#2-creating-the-discord-application)
3. [Enabling the required intents](#3-enabling-the-required-intents)
4. [Getting your Discord bot token](#4-getting-your-discord-bot-token)
5. [Getting your NaraRouter AI key](#5-getting-your-nararouter-ai-key)
6. [Getting your Pterodactyl API keys](#6-getting-your-pterodactyl-api-keys)
7. [Configuring the .env file](#7-configuring-the-env-file)
8. [Uploading the bot to Pterodactyl](#8-uploading-the-bot-to-pterodactyl)
9. [Installing dependencies](#9-installing-dependencies)
10. [Registering the slash commands](#10-registering-the-slash-commands)
11. [Starting the bot](#11-starting-the-bot)
12. [Pterodactyl startup configuration](#12-pterodactyl-startup-configuration)
13. [Permissions and roles](#13-permissions-and-roles)
14. [Command reference](#14-command-reference)
15. [AI features](#15-ai-features)
16. [Ticket system](#16-ticket-system)
17. [Troubleshooting](#17-troubleshooting)
18. [Security recommendations](#18-security-recommendations)

---

## 1. Requirements

| Requirement | Details |
|---|---|
| Node.js | **20 or newer** (Node 22 LTS recommended) |
| Pterodactyl Panel | 1.x with Client API enabled (most panels) |
| Two Ptero API keys | a Client key (`ptlc_…`) and an Application key (`ptla_…`) |
| NaraRouter account | optional — only for AI features (`https://router.bynara.id`) |
| Discord server | where you (or the bot owner) have **Manage Server** permission |
| Storage | ~10 MB + SQLite database (created automatically in `data/`) |

---

## 2. Creating the Discord application

1. Open the **[Discord Developer Portal](https://discord.com/developers/applications)**.
2. Click **New Application** → give it a name (e.g. `PteroBot`) → **Create**.
3. You land on **General Information**. Copy the **APPLICATION ID** — this is your `CLIENT_ID`.
4. Go to **Bot** (left sidebar) → **Add Bot** if needed.
5. **Invite the bot to your server** — replace `YOUR_APPLICATION_ID` in this URL with your Application ID, open it in your browser, pick your server, **Authorize**:

   ```
   https://discord.com/oauth2/authorize?client_id=YOUR_APPLICATION_ID&scope=bot+applications.commands&permissions=379984
   ```

   That permission number grants: View Channels, Send Messages, Embed Links, Attach Files, Read Message History, Add Reactions, Use External Emojis, and **Manage Channels** (needed to create ticket channels).

   > If the invite fails with **"Integration requires code grant."**: Developer Portal → **OAuth2** → Authorization Flow → turn **OFF** "Requires OAuth2 Code Grant" → Save → try the URL again. (See section 17.)

   > Alternative: Developer Portal → **OAuth2** → **URL Generator** → tick scopes `bot` + `applications.commands` → tick the permissions listed above → copy the generated URL.

---

## 3. Enabling the required intents

Still on the **Bot** page:

1. Scroll down to **Privileged Gateway Intents**.
2. Enable **MESSAGE CONTENT INTENT** (required so the AI can read ticket messages and DMs).
3. **SERVER MEMBERS INTENT** is *not* required.
4. Scroll further to **Authorization Flow** — **"Requires OAuth2 Code Grant" must stay OFF.** If it is ON, inviting the bot fails with the error **"Integration requires code grant."** Turn it OFF and press **Save Changes**.

> The bot declares this intent in code, so if the toggle is OFF the login fails with `Used disallowed intents` — it will not start at all until you enable it.

---

## 4. Getting your Discord bot token

1. On the **Bot** page click **Reset Token** → **Copy**.
2. This is your `DISCORD_TOKEN`. **Never share it, never commit it.**
3. Still on the Bot page, make sure **Public Bot** is OFF if you don't want strangers inviting it.

Then invite it to your server:
1. Go to **OAuth2 → URL Generator**, tick `bot` + `applications.commands`.
2. Give it at least: **Manage Channels** (tickets), **Send Messages**, **Embed Links**, **Attach Files**, **Read Message History**, **Manage Messages**.
3. Open the generated URL and invite the bot to your server.
4. Copy your **server ID** (enable Developer Mode in Discord settings, right-click the server → **Copy ID**) — this is `GUILD_ID`.

---

## 5. Getting your NaraRouter AI key

1. Go to **https://router.bynara.id** and sign in / create an account.
2. Open the **API Keys** section and create a key.
3. Copy it into `NARA_API_KEY` in your `.env` — **one single line, no quotes, no spaces**: `NARA_API_KEY=sk-…`
4. Set `NARA_MODEL` to `auto` or to a specific model name your account offers.

Since v1.3.2 the bot **verifies the key at startup** and prints a masked fingerprint like `sk-abc…wxyz (48 chars)` in the console (also visible in `/settings`). Compare it with the key shown in your NaraRouter dashboard — if they differ, the bot is loading an old `.env` (restart the bot on the panel after every `.env` edit) or the paste was damaged (the bot auto-strips quotes, line-wraps and invisible characters and tells you in the console when it did).

If you leave `NARA_API_KEY` empty, the bot runs fine — every AI command simply tells you AI is disabled. **Nothing else breaks.**

---

## 6. Getting your Pterodactyl API keys

You need **two different keys**:

**Client key (`ptlc_…`)** — acts as a normal panel user:
1. Log into your panel as the account that owns (or subuser-manages) the servers.
2. Click your avatar → **Account** → **API Credentials**.
3. **Create** a key, whitelist your panel domain/IP if asked, and copy the `ptlc_…` value into `PTERODACTYL_CLIENT_KEY`.

**Application key (`ptla_…`)** — admin-level, used by `/admin` and `/server-create`:
1. Log into the panel as an admin.
2. Open the **Admin** area → **API Keys** (or *Management → API Keys* on older panels).
3. Create an **Application** key and copy the `ptla_…` value into `PTERODACTYL_APPLICATION_KEY`.

> The bot uses exactly these two keys and nothing else. It never asks for your panel password.

---

## 7. Configuring the .env file

Copy `.env.example` to `.env` in the bot folder and fill in every value:

```env
DISCORD_TOKEN=your.bot.token
CLIENT_ID=123456789012345678
GUILD_ID=123456789012345678
NARA_API_KEY=your-nararouter-key
NARA_MODEL=auto
NARA_TICKET_MODEL=
PTERODACTYL_URL=https://panel.example.com
PTERODACTYL_CLIENT_KEY=ptlc_xxxxxxxxxxxxxxxx
PTERODACTYL_APPLICATION_KEY=ptla_xxxxxxxxxxxxxx
DATABASE_PATH=./data/bot.sqlite
AI_ENABLED=true
AI_TICKET_ENABLED=true
AI_AUTO_REPLY=false
AUTO_LINK=true
REACT_EVERYONE=true
REACT_ALL_EMOJIS=true
REACT_BOT_PING=true
EVERYONE_EMOJI=📢
HERE_EMOJI=🔔
LOG_CHANNEL_ID=
ADMIN_ROLE_ID=
STAFF_ROLE_ID=
SUPPORT_ROLE_ID=
BOT_NAME=PteroBot
BRAND_NAME=DevAvoid
```

| Variable | Meaning |
|---|---|
| `DISCORD_TOKEN` | bot token from section 4 |
| `CLIENT_ID` | application ID from section 2 |
| `GUILD_ID` | server ID — commands appear **instantly**; empty = global (up to 1 h) |
| `NARA_API_KEY` / `NARA_MODEL` | AI access; empty key disables all AI features. Paste the key on ONE line, no quotes — the bot cleans paste damage, verifies it at boot and shows a masked fingerprint in the console + `/settings` |
| `NARA_URL` | base URL of the NaraRouter API (default `https://router.bynara.id/v1`) — only change it if NaraRouter tells you to |
| `CURSEFORGE_API_KEY` | **optional** — free key from https://console.curseforge.com/ that unlocks CurseForge search/install in `/install`. `/optimize` and Modrinth installs need NO key |
| `NARA_TICKET_MODEL` | optional **faster model used only for ticket auto-replies** (empty = same as `NARA_MODEL`) |
| `NARA_EDITOR_API_KEY` | **v1.8.0 — a SECOND, separate NaraRouter key used ONLY by `/update** (the bot-editing AI). Never shared with the chat key; rotate/revoke either independently. Empty = falls back to `NARA_API_KEY` (the reply then warns the key is shared) |
| `NARA_EDITOR_MODEL` | optional different model for `/update` editing — e.g. a stronger code model (empty = `NARA_MODEL`) |
| `MC_ACCOUNT_EMAIL` / `MC_ACCOUNT_PASSWORD` | **v1.9.0, optional** — a Microsoft account that owns Minecraft, for `/bot join` on **online-mode** servers (real MS auth). **Offline-mode servers (v1.10.0 `auth:offline`, or simply no credentials configured) need NEITHER — no premium Minecraft account required** — the in-game name comes from the `/bot join username` option |
| `PTERODACTYL_URL` | your panel URL (no trailing slash needed) |
| `PTERODACTYL_CLIENT_KEY` | `ptlc_` user key for server/files/backups/etc. |
| `PTERODACTYL_APPLICATION_KEY` | `ptla_` admin key for `/admin` + `/server-create` |
| `DATABASE_PATH` | SQLite location (folder is created automatically) |
| `AI_ENABLED` | master switch for `/ai`, `/smart`, DM chat |
| `AI_TICKET_ENABLED` | ticket AI tools (summarize / suggest / auto-reply) |
| `AI_AUTO_REPLY` | if `true`, the AI answers customers in **new** tickets automatically |
| `AUTO_LINK` | if `true` (default), Discord users are auto-linked to panel users with the same username and get access to the servers they own (section 13) |
| `REACT_EVERYONE` | master switch for ping reactions — see the Utility section below |
| `REACT_ALL_EMOJIS` | if `true` (default), ping reactions use **every emoji uploaded to the server** (up to Discord's 20-reactions-per-message cap); `false` = single 📢/🔔 stamp |
| `REACT_BOT_PING` | if `true` (default), the bot also reacts when someone **pings it directly** (not just @everyone/@here) |
| `EVERYONE_EMOJI` / `HERE_EMOJI` | fallback emoji for the mass-ping reactions when the server has no custom emojis (any unicode emoji, default 📢 / 🔔) |
| `LOG_CHANNEL_ID` | channel that receives audit logs |
| `ADMIN_ROLE_ID` / `STAFF_ROLE_ID` / `SUPPORT_ROLE_ID` | permission tiers (section 13) |
| `BOT_NAME` | cosmetic name used in embed footers |
| `BRAND_NAME` | **your hosting brand** — the AI represents ONLY this brand, never recommends competitors, and routes pricing/free-server questions to tickets/staff (default `DevAvoid`) |

---

## 8. Uploading the bot to Pterodactyl

Host the bot itself as a server on your panel:

1. Create a new server with the **Node.js** egg (any recent Node.js egg; the standard "Node.js" egg with a `MAIN_FILE` variable works out of the box).
2. In the panel **Files** tab, upload `pterobot.zip` → right-click it → **Decompress** → delete the zip afterwards.
3. **IMPORTANT — files must sit DIRECTLY in `/home/container`, NOT inside a sub-folder.** After decompressing you must see exactly this when you open the Files tab root:

   `index.js`, `package.json`, `package-lock.json`, `.env.example`, `.gitignore`, `LICENSE`, `README.md`

   If instead you see a `pterobot/` (or any) folder containing them, the server cannot find `index.js` and will crash with `Cannot find module './index.js'`. Fix it in seconds — open the **Console** and paste:

   ```bash
   cd /home/container
   cp -r pterobot/. ./
   rm -rf pterobot
   ls -la
   ```

   The final `ls -la` must list `index.js` and `package.json` at the top level.
4. Create your `.env` file: in the panel Files tab → **New File** → name it `.env` → paste the section 7 content → save.

> Alternative: run it on any machine with Node.js 20+ — the bot is plain Node, nothing Ptero-specific is required to *run*.

---

## 9. Installing dependencies

Open the server **Console** in the panel (or a terminal if self-hosted) and run:

```bash
npm install
```

This installs `discord.js`, `better-sqlite3`, `axios` and `dotenv` (~90 packages). On a Node.js egg, build tools are usually included; if `better-sqlite3` fails to compile, restart the server once so the egg installs its build dependencies, or pick a Node.js egg ≥ 18.

---

## 10. Registering the slash commands

**You normally do NOT need this step** — the bot registers its commands automatically the first time it starts (and whenever they change). If you want to force it manually:

```bash
npm run deploy
```

That runs `node index.js --deploy`, registers all 17 commands to your `GUILD_ID` (instant) or globally (up to 1 hour), and exits.

---

## 11. Starting the bot

```bash
npm start
```

On Pterodactyl: just press **Start** on the server. You should see:

```
==============================================================
  PteroBot is online as YourBot#1234
  Guilds: 1 • Commands: 17
  AI: enabled (auto) • Ticket AI: enabled
  Panel: https://panel.example.com • Client key: set • App key: set
==============================================================
```

Type `/help` in Discord to confirm the commands are live.

---

## 12. Pterodactyl startup configuration

**Standard Node.js egg (recommended, no changes needed):** open the server **Startup** tab and set the `MAIN_FILE` variable to:

```
index.js
```

The egg then runs `npm install` automatically on every boot (because `package.json` exists) and starts the bot. Nothing else is needed — no Docker, no nginx, no reverse proxy, no domain, no SSL.

> Technical note: due to a quoting bug in the egg's startup script (`[[ "${MAIN_FILE}" == "*.js" ]]` — a quoted glob never matches), the egg actually launches `.js` files through `ts-node --esm`. The bot detects this automatically since v1.0.1, so it works either way.

**Custom startup command (only if your host lets you edit it):**

```
if [ ! -d node_modules ]; then npm install; fi; node index.js
```

> If your egg only offers `npm start`, that works too — `package.json` already maps `start` → `node index.js`.

---

## 13. Permissions and roles

The bot uses three permission tiers, resolved in this order:

| Tier | How to get it | What it allows |
|---|---|---|
| **Admin** | Discord **Administrator** permission, or `ADMIN_ROLE_ID`, or `/permissions grant` `admin` | everything, incl. `/admin`, `/server-create`, `/settings` |
| **Staff** | `STAFF_ROLE_ID`, or `/permissions grant` `staff` | all servers + tickets |
| **Support** | `SUPPORT_ROLE_ID`, or `/permissions grant` `support` | tickets only |
| **Member** | everyone else | only servers granted to them, `/ai`, tickets they open |

### Auto-link (recommended — no manual user management)

**You link the admin's panel once, and everyone else is handled automatically:**

1. A Discord user's **username matches their panel username** (case-insensitive) → the first time they use any bot command they are **automatically linked**.
2. Once linked, they **automatically get access to every server their panel account owns** — no `/permissions server-grant` needed, ever.
3. `/server-create` lists **every panel user** as a possible owner (with a *Type username / email* button for large panels) — customers never need to run `/link` first, and the new server is **auto-granted** to its owner.
4. Run `/admin sync-users` once after setup to link everyone who is already in your Discord server right now.

> The rule is simple: **tell your customers to use the same username on Discord as on the panel.**

Controls:
- `AUTO_LINK=true/false` in `.env` (default `true`) — global default.
- `/settings autolink enabled:true|false` — per-Discord-server toggle.
- Manual `/link` (or `/admin link`) always overrides an auto-link; run it again to re-point a wrong match.
- Every auto-link is written to the audit log (`account.autolink`).

**Manual server access** (for exceptions) is explicit — an admin runs:

```
/permissions server-grant user:@player server:Survival
```

Now `@player` can use `/server status`, `/file list`, `/backup list` … on that server only. `/permissions list` shows everything at a glance. The `ai` permission can be granted/denied per user the same way.

> Discord role IDs: Server Settings → Roles → right-click the role → Copy ID.

---

## 14. Command reference

**68 commands.** Server arguments support **autocomplete** — start typing the server name.

### 🤖 AI
| Command | Description |
|---|---|
| `/ai <message>` | chat with the AI — it **remembers your conversation** (last 16 messages). **Ticket-style chat:** your message is echoed as **your own message** (your name + avatar, plain text — never an embed), the bot shows **"⟳ PteroBot is thinking…"** while it works, then answers as a **plain message — no embed, no AI footer** (v1.7.0). Your echoed prompt gets a ✅ reaction when answered (❌ on error). |
| `/ai-reset` | wipe your AI conversation history |
| `/history ai [user] [page]` | **v1.9.0** — see your **AI conversation memory** (exactly what `/ai` and `noctis <question>` see when they answer); staff can view anyone's; `/history fixes [user]` = your executed panel actions & AI auto-fixes; `/history updates` (admin) = the `/update` self-edit history + backups; `/history server <server>` = one server's action timeline |
| `/smart <request>` | one-shot natural language, e.g. *“restart my survival server”* — same ticket-style chat experience as `/ai`. **Say “my server lags”** and it offers BOTH fixes: run `/optimize` (it can even execute the installer for you after a Confirm click) or upgrade the plan via a ticket |
| *(auto)* **self-diagnosis & fix** | **v1.7.0 — every AI path** (`/ai`, `/smart`, `@Noctis …`, `noctis …`) **first checks your linked panel user, walks your servers and reads each one's live state** (stopped / crashed / RAM / CPU). The diagnosis is injected into the conversation, and the AI **fixes fixable problems in the same reply** — a stopped server gets started, a crashed one restarted — immediately, no confirmation. Destructive actions (kill, optimize) still need the `/smart` Confirm buttons. Not linked? It explains `/link` first. |
| *(voice)* `noctis help him` | **v1.6.0 — no command needed.** Say **`noctis help him`** (or *her / them / @user*) in ANY channel — tickets, community chat, anywhere — and the AI **reads the last 30 messages of that channel** and replies, helping that person. Members can ask for help; **only staff can pause/resume** the AI. |
| *(voice)* `noctis i will reply now` | **staff-only** — the bot **stops replying in that channel** instantly (also stops the ticket AI auto-reply there). Say `noctis help him`, `noctis start replying` or `noctis come back` (staff) to bring it back. Persisted across restarts. |
| *(voice)* `noctis <question>` | anything else after the wake word chats with the AI like a **ping with a message** — e.g. `noctis why is my server lagging?` — and if the answer is a fixable problem, the fix runs immediately (v1.7.0). |
| *(voice)* `noctis ping @user [N times \| forever]` | **v1.7.0 + v1.9.1 — plain-text ping trigger, NO LIMIT.** `noctis ping @user` pings that user in a **normal message — not an embed**; `noctis ping @user 500 times` (or `500x` / just `500` / `twice`) pings them **500 times** — any number goes; `noctis ping @user forever` (or `infinite` / `unlimited` / `endless` / `nonstop`) pings them **literally forever**. `him/her/them/me` or a bare username works too. **Kill switch:** `noctis stop pinging` stops every running storm (staff + the requester stop all; anyone can stop storms aimed at them) — `/settings auto-ping off` also kills them instantly. Gated by `/settings auto-ping`, 10 s cooldown between requests. |
| *(voice)* `noctis ai channel on / off` | **v1.12.0 — THE AI CHANNEL.** Staff say **`noctis ai channel on`** in ANY channel → from then on the bot answers **EVERY message there** — no wake word, no ping, no `/ai` — with the full engine (per-user memory, live diagnosis, auto-fix, panel actions, **server file reading**). `noctis ai channel off` (said anywhere) disables it; bare `noctis ai channel` shows where it is. 5 s per-user cooldown; shown in `/settings`. |
| *(in any AI chat)* `list files` / `read <file>` / `show my logs` | **v1.12.0 — server file sight.** Ask file questions in `/ai`, `/smart`, a ping, `noctis …` or the AI channel and the bot pulls the **REAL file tree / contents** off the panel into the AI's context: `list files [of <server>]`, `read server.properties`, `read the file plugins/config.yml on survival`, `show my logs` (last 60 lines of latest.log) — answers quote actual settings instead of guessing. |

The AI **only represents YOUR brand** (set `BRAND_NAME` in `.env`, default `DevAvoid`): it never names or recommends other hosting providers, and anything brand-specific it doesn't know (pricing, plans, free servers) is answered with *"open a ticket / ask a staff member"* — like a real support agent.

### 🎫 Tickets
| Command | Description |
|---|---|
| `/ticket panel` | (staff) posts a **Create Ticket** panel with a button + subject form — the modern ticket-bot experience |
| `/ticket create <subject>` | opens a private channel (owner + support roles only); users with an open ticket are pointed to it instead |
| `/ticket close [reason]` | closes the ticket, DMs + logs the transcript, deletes the channel |
| `/ticket claim` | staff claims the ticket |
| `/ticket add <user>` / `/ticket remove <user>` | manage who can see the ticket |
| `/ticket rename <name>` | rename the channel |
| `/ticket transcript` | export the ticket as a `.txt` file |
| `/ticket ai <summarize\|suggest_reply\|toggle>` | AI ticket tools (section 16) |

Every ticket opens with a **live control panel** (Ticket V2 style): 🔒 Close (with confirmation) • 🙋 Claim • 🤖 AI Auto-Reply ON/OFF **(staff-only toggle)** • 📋 Transcript • and a **priority menu** (Low/Medium/High/Urgent). The header updates itself as the ticket is claimed, re-prioritized or the AI is toggled.

**AI auto-reply in tickets (v1.6.0):** ON by default — the AI answers **members** automatically in ticket channels (including TicketTool-style channels made by other bots) while staff are away. Staff messages are never auto-answered. **Only staff can enable/disable** it (header button, `/ticket ai toggle`, or guild-wide `/settings ticket-ai`) — members cannot. Staff can also pause it per channel with `noctis i will reply now`.

### 🖥 Servers & Nodes
| Command | Description |
|---|---|
| `/server list` | servers you can manage |
| `/sync [all]` | **v1.10.0** — sync servers from the panel: drops every cache, re-pulls the fresh server list, auto-links your panel account, re-grants owned-server access and shows your servers back instantly; `all:true` (admin) re-grants **every linked member** at once |
| `/server info <server>` | details + **connection IP:port address** + all allocations (primary 📌 + extra ports) + limits (GB) + power buttons ▶⏹🔄⛔📊 — embed is **colored by live state** (green=running, yellow=starting, red=offline, purple=suspended) with emoji field headers |
| `/ip <server>` | **the connection address, fast** — primary IP:port in big text + alias, notes, extra allocations, power buttons. **v1.6.0:** allocations are read from every panel response shape, with an **Application-API fallback** — the "no allocations" error is gone |
| `/ping [server]` | bot latency (Discord REST + gateway + panel API) + **live Minecraft server ping** — the **selected** server is probed 3× and shows **min/avg/max ms** plus players, version and MOTD; without `server:` all of yours are pinged at once |
| `/status` | **v1.6.0 — animated panel overview**: staff see per-node **RAM & storage bars** (▰▰▱), how many **servers are hosted** on each node and panel totals — **hostnames/FQDNs are never shown** (v1.6.1), node names that look like hostnames are shortened to the label; every field header + node card is decorated with the guild's own animated emojis (v1.6.1); everyone else sees the public summary (servers hosted, nodes, your servers, uptime) |
| `/list <server>` | **who is online right now** — instant live status ping (counts + player names on Paper) with the console `list` + log-reading method as fallback |
| `/server status <server>` | live state + quick resources |
| `/server resources <server>` | memory, disk, CPU, uptime, network |
| `/server start/stop/restart <server>` | power signals (direct API, no AI tokens spent) |
| `/server kill <server>` | requires pressing **Confirm** (destructive) |
| `/server suspend <server>` | (admin) suspend a server — the owner can no longer start it |
| `/server unsuspend <server>` | (admin) re-enable a suspended server |
| `/server reinstall <server>` | (admin) reinstall from the egg defaults |
| `/server delete <server>` | (admin) **permanently** delete a server + files — double confirmation |
| `/server auto-suspend <server> <duration>` | (admin) schedule automatic suspension — e.g. `12h`, `7d`; `0` cancels |
| `/server auto-suspend-list` | (admin) all scheduled auto-suspensions with countdowns |
| `/nodes` | **node status for everyone** — 🟢/🔴 reachability (probed directly on wings), version, RAM & disk capacity bars, maintenance state (30s cached — instant) |
| `/nodes node:<id>` | (staff) one node in detail — capacity, overallocation, server count. **Node FQDNs/IPs stay hidden here** — only each server's own connection address is shown, in `/server info` |

### 👤 Panel users
| Command | Description |
|---|---|
| `/user create <username> <email> [for] [password] [first_name] [last_name]` | (admin) add a user to the Pterodactyl panel in one step — no modal. Pick **`for:@member`** and they are instantly linked **and DMed their username + password** (password spoilered — click to reveal). Blank password = random one. Without `for`, a Discord member with the same username is auto-linked. |
| `/user creds [user]` | (admin) **login receipts, kept like ticket transcripts** — blank = list the latest receipts (who, for which member, when); `user:@member` = **re-DM them their saved login details**. Passwords are stored encrypted (AES-256-GCM, key derived from `DISCORD_TOKEN` — never in the database). If a member loses the DM, this re-sends it in one click. |

### 🛡 Moderation (Olympus-style)
| Command | Description |
|---|---|
| `/mod kick <user> [reason]` | kick a member |
| `/mod ban <user> [reason] [delete_days]` | ban (optionally delete up to 7 days of their messages) |
| `/mod softban <user>` | ban + unban — purges their messages but they can rejoin |
| `/mod unban <user_id>` | lift a ban by raw user ID |
| `/mod timeout <user> <duration> [reason]` | mute via Discord timeout (max 28 d) — `10m`, `1h`, `1d`… |
| `/timeout <user> <duration> [reason]` | **the quick top-level mute** (v1.5.0) — same engine as `/mod timeout`, no /mod needed; `duration:0` clears it |
| `/untimeout <user>` | **remove the timeout instantly** (v1.5.0) — same as `/mod untimeout` |
| `/mod mute <user> <duration> [reason]` | **same as timeout** — the mute alias |
| `/mod untimeout <user>` | remove the timeout (unmute) |
| `/mod unmute <user>` | **same as untimeout** — the unmute alias |
| `/mod warn <user> <reason>` | warn (stored per server) — the user is DM'd best-effort |
| `/mod warnings <user>` / `/mod clear-warnings <user>` | view / wipe a member's record |
| `/mod purge <amount> [user]` | bulk delete up to 100 messages, optionally only one user's |
| `/mod slowmode <seconds>` | set slowmode (0 = off) |
| `/nuke [all]` | delete & recreate the current channel — **wipes every message** (double confirmation + cooldown); `all:true` (admin) nukes **EVERY channel** in the server after a typed **NUKE ALL** modal confirmation, then creates one fresh #general |
| `/servernuke [channels] [roles] [emojis] [stickers] [ban_members] [recreate]` | **v1.10.0, admin — the TOTAL wipe**: deletes all channels, all deletable roles (never @everyone/managed/bot roles), all emojis, all stickers, optionally **bans every member** except the owner, bots and you, and creates a fresh #general for the report — confirmed by typing the server's exact name into a modal |
| `/spam <message> [count] [delay] [user] [channel]` | **v1.10.0, staff** — send a message multiple times (up to 500, delay 300–10000 ms); `user:` pings that member in **every** message; other channels than the current one need admin |
| `/dmnuke <user> [limit]` | **v1.10.0** — wipe every message **the bot sent** in its DM channel with that user (scans up to 1000): staff can nuke any user's DMs, everyone can nuke **their own** — the other person's messages cannot be deleted by the bot |
| `/lock [channel] [all]` / `/unlock [channel] [all]` | lock/unlock a channel or the **whole server** (@everyone SendMessages) |
| `/antinuke enable [threshold] [window] [punishment]` | **anti-nuke protection**: anyone (except owner + whitelist) who performs multiple destructive actions (channel/role/emoji/sticker deletes, mass bans) within the window is auto-punished — strip roles / kick / ban. Detected via the audit log. |
| `/antinuke status\|threshold\|window\|punishment\|whitelist-add\|whitelist-remove\|whitelist-list` | tune the protection |

> The moderation tools need the classic Discord permissions (Kick/Ban/Moderate Members/Manage Channels/Manage Roles/Manage Messages) **plus View Audit Log** for anti-nuke. Grant the bot's role those permissions in your server settings — every command checks both your permissions and the bot's, and respects the role hierarchy.

### 🧮 Utility suite (v1.7.0 — 274 tools in 17 commands)

Discord caps a single slash command at **8,000 characters** (and one oversized command fails the whole registration), so the `/text` hub ships **split into one command per category**: type `/text-` and the picker lists them all. `/text` itself is an index — its `tools` subcommand lists every command in the hub. The other five hubs stay single commands. Every tool is real functionality, no placeholders. **The former `/math` hub (13 commands, 137 tools) was removed in v1.6.0 and the entire `/convert` hub (24 commands, 500 tools) was removed in v1.7.0 — both at the owner's request.**

| Hub | Tools | Highlights |
|---|---|---|
| `/text-*` (11 commands + index) | 147 | case, 𝕕𝕚𝕤𝕔𝕠𝕣𝕕-𝕤𝕥𝕪𝕝𝕖𝕤 (bold/italic/script/fullwidth/smallcaps/zalgo…), leetspeak, **ROT13, Vigenère, Atbash, A1Z26 ciphers**, hex/base32/base58/binary encode+decode, **MD5/SHA-1/SHA-256 hashes**, CRC32, roman numerals, number-to-words, slugify, JSON pretty/minify/validate, CSV parse, text stats — e.g. `/text-case upper`, `/text-style bold`, `/text-hash sha256` |
| `/fun` | 38 | **button games: Rock-Paper-Scissors, RPS-Lizard-Spock, Guess-my-number 1-100** (range narrows with every click), coinflip, dice `3d6`, Magic 8-Ball, Magic Conch, ship, rate, **trivia/jokes/quotes/fortunes/riddles with spoiler-hidden answers**, would-you-rather, truth-or-dare, deep questions & conversation topics, **dev-humor group: buzzword startup pitches, commit messages, excuses, MC death messages, dev laws, fake-hack** — plus the v1.6.0 **`rates` group (fixed per person, same user = same score): cute, simp, pp, iq, vibe-check** |
| `/util` | 28 | userinfo/serverinfo with banners, **reaction polls (up to 9 options) + poll results with bar charts**, **reminders that actually ping you** (`/util reminders remind in:10m what:…` — DM or channel, persisted in SQLite), snowflake decoder, first-message, random member, bot stats, **password generator (spoiler-hidden), UUID v4, one-time codes, random numbers, lorem ipsum** |
| `/time` | 22 | Discord timestamp builder (`<t:…:R>`), date math, days-between, age, countdowns, **zodiac signs, next weekday, business-day count, year-progress bar, moon phase, world clock + timezone diff (with autocomplete over common zones)** |
| `/color` | 23 | hex/RGB/HSL/CMYK conversions, **WCAG contrast grading (AAA/AA)**, complementary/analogous/triad harmonies, lighten/darken, **weighted color mix, 5-stop gradients with CSS output, shade scales**, CSS-name lookup with **autocomplete over all 140 CSS colors**, live color swatches |
| `/mc` | 16 | **real Mojang API**: UUID lookup, skin/head/cape renders (Crafatar), server status via mcsrvstat (players, version, MOTD, icon), XP-level math, tick converter, chunk lookup, 3D distance, **nether portal coords (1:8), beacon range/cost, /give command builder with custom name & lore, MOTD §-code styler, furnace fuel table** |

> Game buttons expire after 30 minutes; reminders are checked every 15 seconds and survive restarts (SQLite); delivered reminders are garbage-collected after 7 days.

### 🏆 Levels & fun meters (v1.6.0 / v1.7.0)
| Command | Description |
|---|---|
| `/rank [user]` | level card — level, total XP, progress bar to the next level, server rank position, messages counted |
| `/leaderboard [page]` | top 10 chatters by XP with 🥇🥈🥉 medals |
| `/lvl add <user> <amount> [xp\|levels]` | **v1.7.0, admin** — grant (or, with a negative amount, remove) XP or whole levels; proper level-up announcements fire for grants |
| `/ship <user1> [user2]` | compatibility meter + generated ship name — deterministic (the same pair always ships the same) |
| `/gayness [user]` | rainbow meter 🏳️‍🌈 — fixed per person, with a verdict tier |
| `/lesbianess [user]` | sapphic meter 💜 — fixed per person, with a verdict tier |
| `/fun rates cute\|simp\|pp\|iq\|vibe [user]` | five more deterministic people-meters |
| *(automatic)* XP | members earn **15-25 XP per message** (one award per minute); level-ups are announced in the same channel or a fixed one set with `/settings levels channel:` |
| `/settings levels <enabled> [channel]` | (admin) toggle the levels system + set the level-up announce channel |

### 🎮 Player bot (v1.9.0 — mineflayer)
| Command | Description |
|---|---|
| `/bot join [server] [host] [port] [username] [version] [auth]` | a **REAL player** joins one of YOUR servers — the address is auto-resolved from the panel allocations (pick `server:` with autocomplete). `username` (default `Noctis`) sets the in-game name on offline-mode servers; `host`/`port` is the staff-only manual path; `version` forces a Minecraft version (default auto-detect). **`auth` (v1.10.0): `offline` joins offline-mode servers with any username — NO premium Minecraft account needed; `auto` (default) uses Microsoft auth only when `MC_ACCOUNT_EMAIL` + `MC_ACCOUNT_PASSWORD` are set; `microsoft` forces it.** One bot per Discord server; only its starter + staff can control it |
| `/bot come [player]` / `/bot follow [player]` | walk to a player (default: you, matched by Discord name) / keep following them (distance 3) |
| `/bot goto <x> <y> <z>` | walk — and dig — to coordinates |
| `/bot mine <block> [count]` | mine blocks by name (it equips the best tool and picks up the drops), e.g. `block:oak_log count:4` |
| `/bot place <item> [x] [y] [z]` | place a block from the inventory — at coords, or on the ground in front of the bot |
| `/bot attack <target>` | attack a mob or player once — e.g. `target:zombie` |
| `/bot combat [mode]` | **self-defense (v1.17.0, default ON)** — anything that hits the bot (zombies, skeletons, spiders, players…) gets its best weapon swung back: chase, proper attack-cooldown timing, creeper retreat, fight-then-resume. A scuffle **pauses a running build** instead of derailing it. `mode:on` / `mode:off` toggles |
| `/bot chat <message>` | say something in the Minecraft chat (the bot's name shows in-game) |
| `/bot inventory` / `/bot drop` / `/bot equip` | see the inventory, drop items, hold/wear an item (hand / off-hand / head / torso / legs / feet) |
| `/bot status` / `/bot stop` / `/bot respawn` / `/bot leave` | health, food, position, activity, self-defense state • stop any task • respawn after death • leave the server |
| `/bot diamonds` | **THE QUEST** — wooden pickaxe → stone → smelted iron → DIAMONDS: chops trees, crafts, digs to Y -58 (steering around lava), branch-mines, live progress in Discord. `/bot stop` cancels any time |
| *(automatic)* life | the bot **looks around and swings idly** while waiting, relays **whispers + name mentions** from the Minecraft chat back to Discord (rate-limited), auto-announces joins/kicks/deaths, **fights back when attacked (v1.17.0)**, and leaves cleanly when the bot shuts down |

> The player bot needs three extra packages: run `npm install mineflayer mineflayer-pathfinder mineflayer-collectblock` in the bot folder (they are already in `package.json`, so a plain `npm install` after updating covers it). The bot boots fine without them — `/bot` then shows the exact fix command.

### 💤 AFK & broadcasts (v1.7.0)
| Command | Description |
|---|---|
| `/afk [reason] [duration]` | set yourself away — anyone who **pings you** gets your reason + how long you've been gone (plain message, max one notice per target per minute); your **next message auto-clears it** with a welcome-back note. `duration` accepts `30m`, `2h`, `1d`… and auto-expires the status. Running `/afk` again also clears it. |
| `/announcement <title> <message> [channel] [color] [ping] [role] [image]` | **staff broadcast** — a colored embed titled `📢 <title>` with your markdown message, optional `@everyone` / `@here` / role ping (plain text above the embed) and an image URL. Default color is blurple; color names and hex codes both work. |

> Scores are **deterministic** — a seeded hash of the user ID, not random: the same person always gets the same result, like classic fun bots. They are jokes, not statements — the embeds say so.

### 🎨 Embeds & Presence
| Command | Description |
|---|---|
| `/embed <title> <description> [color] [fields] [footer] [image] [thumbnail] [channel] [timestamp]` | (admin) **Olympus-style rich embed builder** — post a polished embed to any channel. Colors by name (green, red, yellow, blurple, orange, purple, pink, teal, gold, white, black) or hex (`#5865F2`). Fields: one per line as `Name | Value` (up to 25). Optional big image + corner thumbnail, custom footer, timestamp. |
| `/react <message> <emoji> [channel]` | (staff) add any emoji reaction to any message — paste the **message link** (right-click → Copy Message Link) or a raw message ID. Works with unicode emoji and custom server emojis. |
| `/vc join [channel]` | the bot **joins a voice channel and just sits there** — mic muted + deafened, no audio, pure presence (needs the `@discordjs/voice` + `libsodium-wrappers` packages from `package.json` — a fresh `npm install` after updating covers it). Default channel: the one you are in. If the join fails, the error message tells you **exactly which stage stalled** (see Troubleshooting for the UDP note). |
| `/vc leave` / `/vc status` | make the bot leave / show where it is sitting + live connection diagnostics |
| *(automatic)* ping reactions | **every emoji you uploaded to the server** (up to Discord's 20-reaction cap, animated last) is added to any @everyone / @here ping — and to any message that **pings the bot directly**. Servers with no custom emojis fall back to 📢 / 🔔. Ticket channels are excluded. Configure with `REACT_EVERYONE` / `REACT_ALL_EMOJIS` / `REACT_BOT_PING`. |

### 📁 Files
| Command | Description |
|---|---|
| `/file list <server> [path]` | paginated directory listing |
| `/file read <server> <path>` | read text files in paginated code blocks |
| `/file create <server> <path>` | modal with content |
| `/file write <server> <path>` | modal **pre-filled with the current file content** |
| `/file upload <server> <path> <attachment>` | upload any Discord attachment (≤ 8 MB) |
| `/file rename` / `/file copy` | rename / duplicate |
| `/file delete` | requires **Confirm** — path-traversal protected |
| `/file download` | signed download link (folders are tar'd automatically) |

### 💾 Backups / Databases / Network / Startup / Subusers
| Command | Description |
|---|---|
| `/backup list\|create\|info\|download\|delete` | backups; delete requires **Confirm** |
| `/database list\|info\|create\|delete` | passwords are **never** shown; DB hosts are hidden by policy |
| `/network list\|primary\|assign\|unassign` | allocations — **only ports are shown**, host IPs/FQDNs stay hidden |
| `/plugins list <server> [folder]` | **every plugin/mod jar** on the server (auto-detects `plugins/` vs `mods/` + the Minecraft version & loader) with sizes and dates **v1.12.0: `/plugins run` takes in-game style commands — a leading `/` is stripped for you (`/list`, `/essentials:kick Notch`)** |
| `/plugins run <server> <command>` | **run their console commands from Discord** — e.g. `essentials:kick Notch`, `chunky start`, `spark tps` — the log output is shown back; audit-logged |
| `/optimize <server> [preview] [restart] [version] [loader]` | **one-click performance pack from Modrinth**: Lithium, ServerCore, Krypton, ModernFix, MemoryLeakFix, LazyDFU, Clumps, Chunky, spark, VMP, DashLoader, EntityCulling, LMD, RakNetify, LagFixer, FastAsyncWorldEdit + the *Essentially Optimized Server* modpack. Auto-detects the Minecraft version + loader (egg startup vars → server log → folder sniffing), installs only **exact compatible builds**, replaces old versions, skips what is already installed, and can restart the server afterwards. `preview:true` is a full dry-run |
| `/install query:<mod> <server> [version] [loader]` | search **Modrinth** (and **CurseForge** when `CURSEFORGE_API_KEY` is set) for ANY mod/plugin/modpack → pick from a menu → see the exact matched file → confirm → installed. Modpacks (`.mrpack`) are expanded and every server-supported file inside is installed **v1.12.0: version-family matching** (a 1.21 build installs on 1.21.4 and vice versa, protocol-equal siblings preferred) and **interactive version + loader pickers** when detection fails — the flow never dead-ends |
| `/startup view\|variables\|set` | startup variables, **validated against egg rules before** hitting the panel |
| `/subuser list\|add\|update\|remove` | presets (full/power/files/backups/readonly) + custom multi-select |

### 🛠 Admin (Application API)
| Command | Description |
|---|---|
| `/admin servers [page]` | all panel servers |
| `/admin server <id> <info\|suspend\|unsuspend\|reinstall\|delete>` | delete requires **Confirm** |
| `/admin users [page]` / `/admin user <create\|info\|update\|delete> [id\|email]` | panel user management via modals |
| `/admin nodes [page]` / `/admin node <id>` / `/admin locations` | infrastructure |
| `/admin nests` / `/admin eggs <nest>` / `/admin allocations <node> [available_only]` | egg & allocation browser |
| `/admin link <user> <ptero_id> <email>` | link another member's account |
| `/admin sync-users` | **auto-link every panel user** to the Discord member with the same username + auto-grant their servers |
| `/admin sync` | **force re-registration of every slash command** — the one-click fix when a command shows in `/help` but not in your command picker (see Troubleshooting) |
| `/server-create` | **5-step creation wizard** — owner → node → nest → egg → **specs** (Starter 1 GB / Basic 2 GB / Pro 4 GB / Ultra 8 GB, Custom form, or Advanced). **Memory & disk are entered in GB** (decimals like `4.5` work, `512mb` too). Custom is a **single form**: server name + RAM / disk / CPU + optional auto-suspend → **created immediately**. The IP/port is **auto-assigned by the panel** (the wizard never asks about IPs). The Advanced path keeps full control: docker image → name/startup → limits (GB) → egg variables → review. The new server is auto-granted to its owner. **v1.12.0: Minecraft eggs now ask which VERSION to install** (egg default or any curated release 1.8.8 → 1.21.11 — written into the egg's version variable) |
| `/settings …` | **the full control panel (v1.6.0 + v1.7.0)** — `view` (everything at a glance, incl. which editor key `/update` uses) • `ticket-ai <enabled>` (staff-level: AI auto-reply in ticket channels) • `mention-ai <enabled>` (AI answers pings-with-a-message) • `voice <enabled>` (the "noctis …" triggers) • **`auto-ping <enabled>` (v1.7.0 — the "noctis ping @user [N times]" trigger)** • `levels <enabled> [channel]` (XP system + level-up announce channel) • `log-channel` • `ticket-category` • `autolink <enabled>` |
| `/update instruction:<change> [action] [preview]` | **v1.8.0 + v1.8.1 + v1.9.1 — the bot edits ITSELF: add, remove or change.** Describe the change in plain English (`add a /8ball command`, `remove /8ball`, `make /ping show the node too`) and the **editor AI** — a **separate** `NARA_EDITOR_API_KEY` — locates the code, plans verbatim-anchored edits (**replace / insert / delete**; removes also clean stray references so nothing dangles), passes `node --check` + structure sanity, **backs up and rewrites index.js**. v1.9.1: whitespace-tolerant anchors, **2 automatic repair rounds** fed the exact errors, JSON comment/trailing-comma tolerance, 429/5xx retries, 4-minute timeout, stage-tagged error messages. `action:` forces `Auto`/`Add`/`Remove`/`Change`; `preview:true` = dry-run. Buttons: **Restart now** / **Undo** / **Show full edits**. Admin-only; backups in `data/selfedit-backups/` |
| `/permissions …` / `/link …` | bot permission & account management |
| `/help` | **interactive command center** — a category select menu with 18 pages (AI, servers, files, tickets, moderation, users, embeds, levels & fun, admin + one live page per utility hub showing its groups & tool counts). Every page is a colored embed with versioned footer & timestamp. **v1.6.1**: modern redesign — quote-block feature list on the home page, and every select-menu option + page title is decorated with the guild's own **uploaded animated emojis** (matched by emoji name, round-robin otherwise). |

---

## 15. AI features

Everything the AI does goes through **one endpoint** (`https://router.bynara.id/v1/chat/completions`) and a **whitelisted action protocol**. The AI cannot run code, cannot call arbitrary APIs, cannot touch anything not on the list:

```
server_list, server_info, server_status, server_resources,
server_start, server_stop, server_restart, server_kill,
server_optimize, backup_list, backup_create, database_list
```

When you ask the AI to *do* something, it answers with a JSON block like:

```json
{"action":"server_restart","server":"Survival","reason":"crashed — restarting"}
```

The bot then: **validates the action exists → re-checks YOUR permissions → resolves the server → executes it via the real API → reports the real result**. `server_kill` and `server_optimize` always require a human **Confirm** button; every other action (including the auto-fix starts/restarts) executes immediately. Every executed action is written to the audit log.

**Auto-diagnosis & auto-fix (v1.7.0):** before EVERY AI reply — `/ai`, `/smart`, `@Noctis …`, `noctis …` — the bot checks the panel account linked to your Discord ID (`/link`), lists the servers assigned to you and reads each one's **live state** (running / stopped / crashed / suspended) plus RAM % and CPU %. The result is injected into the conversation as a `LIVE PANEL DIAGNOSIS` block with an **AUTOFIX RULE**: if a server is stopped, the AI is instructed to start it (`server_start`) in the same reply; if it crashed, to restart it (`server_restart`) — both execute immediately with no confirmation. Unlinked users are told how to `/link` first; suspended servers are routed to staff (open a ticket). The diagnosis is cached for 90 seconds so rapid-fire chats stay fast, and it can never break the AI call — on any error the chat continues without it.

Token saving: simple operations (`/server start`, `/file list`, …) call the Pterodactyl API **directly** — they never spend AI tokens. AI conversation context is truncated to the last 16 messages / ~12k characters, and `/ai` has a 4-second cooldown per user.

**Brand safety (v1.0.5):** every AI prompt — `/ai`, `/smart`, DMs and ticket auto-replies — starts with the same rulebook built from `BRAND_NAME`: the assistant represents *only your hosting service*, never mentions or recommends any other provider (free or paid), never invents prices/plans, and routes "how do I get a (free) server"-type questions to *"open a ticket (`/ticket create`) or ask a staff member"*. General Minecraft/plugin help is still fully answered — just without advertising competitors.

---

## 16. Ticket system

- `/ticket create` (or the **panel button** from `/ticket panel`) makes a private channel visible only to the opener + support/staff/admin roles (a **Tickets** category is auto-created, or set your own via `/settings ticket-category`). Users who already have an open ticket are pointed to it instead of creating a second one.
- The opening message is a **live control panel** (Ticket V2 style): 🔒 **Close** (asks for confirmation, then exports the transcript, DMs it to the owner, posts it to the log channel and deletes the channel) • 🙋 **Claim** • 🤖 **AI Auto-Reply ON/OFF** • 📋 **Transcript** • **priority select menu**. The header embed updates itself whenever the ticket is claimed, re-prioritized or the AI is toggled.
- Closing works from **both** the button (with confirmation) and `/ticket close [reason]` — and it is reliable even in busy tickets, because the bot acknowledges Discord *before* building the transcript (fixed in **v1.0.5**).
- `/ticket ai summarize` — AI recap of the whole ticket.
- `/ticket ai suggest_reply` — AI drafts the next staff reply.
- `/ticket ai toggle` (or the 🤖 button) — enable/disable AI **auto-reply** for that ticket. New tickets start with the `AI_AUTO_REPLY` default. When enabled, the customer's messages are answered by the AI **automatically — no /ai needed** — with a clear "AI auto-reply" footer, while staff can always jump in. The customer's conversation context is kept in memory for instant follow-ups.
- **Speed tip:** ticket auto-replies use a trimmed prompt and an optional faster model — set `NARA_TICKET_MODEL` in `.env` to a small/fast model from your NaraRouter account to make replies near-instant.

---

## 17. Troubleshooting

| Symptom | Fix |
|---|---|
| **`/ip` says "no allocations"** | fixed in **v1.6.0** — some panels return the network endpoint in a different JSON shape (or reject it). The bot now normalizes every known shape AND falls back to the Application API (`include=allocations`) to resolve the address. Update `index.js` and restart; if it still shows, check that the server really has an allocation on the panel (or assign one with `/network assign`). |
| **The bot doesn't answer `noctis help him`** | (1) The phrase must START the message (`noctis help him`, `@Noctis, help him` — not "…ask noctis for help"). (2) Check `/settings voice` is enabled and `AI_ENABLED`/`NARA_API_KEY` are set. (3) If a staff member said `noctis i will reply now` in that channel, the AI stays paused there until staff call it back (`noctis start replying`). (4) There is an 8s per-channel cooldown between AI replies. |
| **The AI still replies in a ticket after "noctis i will reply now"** | The pause is per-CHANNEL and persisted. If the AI still answers, the replies are coming from another channel — or someone re-enabled the ticket AI (`/settings ticket-ai`, the ticket header button, `/ticket ai toggle` are all staff-only). |
| **A member says "noctis stop" and nothing happens** | By design: only staff (support/staff/admin roles or Manage Messages permission) can pause the AI. Members can ask for help but never switch the AI off. |
| **`/rank` / `/leaderboard` say "Levels disabled"** | An admin disabled them: `/settings levels enabled:true` turns the system back on (existing XP is kept). |
| **`/status` shows the public summary instead of node RAM/storage** | The full per-node view requires staff permissions — and a working `PTERODACTYL_APPLICATION_KEY` (the Application API is where node memory/disk come from). Check `/admin nodes` works, then re-run `/status`. |
| npm install finishes, then server goes offline with **exit code 0 and NO bot output** | Fixed in **v1.0.1** — Node.js eggs launch `.js` entries through `ts-node --esm`, which v1.0.0 did not detect, so the bot never actually started. Replace `index.js` with the v1.0.1 file (see `package.json` → `version`) and Start again |
| `npm warn allow-scripts ... better-sqlite3` in console | Harmless on npm 11.17 (the script still runs). ONLY if the bot prints `FATAL: better-sqlite3 failed to load`: Console → `npm approve-scripts better-sqlite3` → `npm rebuild better-sqlite3` → Start again |
| `(node:...) [DEP0180] DeprecationWarning: fs.Stats ...` | Harmless deprecation notice from a dependency on Node 25 — ignore |
| `Error: Cannot find module './index.js'` on boot | files are not at the container root — either the zip was never decompressed, or it decompressed into a sub-folder. Console: `cd /home/container && ls -la`; if you see a `pterobot/` folder run `cp -r pterobot/. ./ && rm -rf pterobot`, if you see only the zip right-click → **Decompress** in Files; then confirm `index.js` + `package.json` are at the top level and the Startup tab `MAIN_FILE` is `index.js` |
| `npm install` never runs on boot | `package.json` is missing from `/home/container` — same cause and fix as above (the egg only installs when it finds `package.json` at the root) |
| `FATAL: DISCORD_TOKEN is not set` | create `.env` (copy from `.env.example`) and fill it in |
| `Failed to login: Used disallowed intents` | enable **MESSAGE CONTENT INTENT**: Discord Developer Portal → your app → **Bot** tab → Privileged Gateway Intents → toggle ON → **Save Changes** → Start again (see section 3) |
| `Failed to login: An invalid token was provided` | reset the token in the Developer Portal, update `.env` |
| Inviting the bot fails with **"Integration requires code grant."** | Developer Portal → your app → **OAuth2** (left sidebar) → scroll to **Authorization Flow** → turn **OFF** "Requires OAuth2 Code Grant" → **Save Changes** → open the invite link again |
| Banner shows `Guilds: 0` | the bot has not been invited to your server yet — see section 2 (invite URL) and the code-grant fix above |
| Ticket could be created but **closing did nothing / showed "interaction failed"** | fixed in **v1.0.5** — the close flow now acknowledges Discord before building the transcript. Replace `index.js`/`package.json` with the current files |
| The AI recommends other hosting providers (Aternos, Minehut, …) | v1.0.5 brand rules prevent this. Make sure `BRAND_NAME` in `.env` matches your brand (default `DevAvoid`) and you are running the current `index.js` |
| Ticket AI auto-replies are slow | set `NARA_TICKET_MODEL` in `.env` to a small/fast model from your NaraRouter account (v1.0.5 also removed the slow channel re-fetch before every reply) |
| Commands don't appear (but `/help` lists them) | run **`/admin sync`** — it force-re-registers every command and reports the count. v1.1.0 also re-registers once automatically on first boot after the update and retries failed registrations. Then press **Ctrl+R** in Discord (global commands are cached by your client for up to 1 hour). Setting `GUILD_ID` in `.env` makes commands appear instantly instead. |
| **`Sync failed — Command exceeds maximum size (8000)`** (`APPLICATION_COMMAND_TOO_LARGE`) | fixed in **v1.3.1**. Discord caps a single slash command at 8,000 characters, and ONE oversized command makes the whole registration fail (so no new commands appear at all — that's also why `/mc` was "not working"). The oversized hubs are now split into one command per category (`/convert-length`, `/text-case`, `/math-basic`…). Update to the current `index.js`, restart, and the bot re-registers automatically (or run `/admin sync`). The bot also checks every command's size locally before talking to Discord, so this failure mode can never silently return. |
| `/vc join` says “Voice not installed” | run `npm install` in the server console (the new `@discordjs/voice` + `libsodium-wrappers` packages are in `package.json`) and Start again |
| `/vc join` says **“The operation was aborted”** (bot appears in the channel, then leaves) | the voice connection stalled — v1.1.1 tells you **exactly which stage** in the error and logs `[vc]` progress lines in the console. Two causes: ① **outbound UDP blocked** by your host/container — Discord voice *requires* UDP; ask your host to allow outbound UDP (or check the container's network/firewall settings). ② missing crypto package — run `npm install` so `libsodium-wrappers` is present. Connect permissions are **not** the issue if the bot briefly appeared in the channel. |
| Server created via `/server-create` instantly goes offline with **`invalid reference format: repository name (library/Java 25) must be lowercase`** | fixed in **v1.1.1** — the wizard was sending the egg's image *label* ("Java 25") instead of the real image (`ghcr.io/parkervcp/yolks:java_25`). New servers are now created with the correct image. **For a server already broken by this:** panel → that server → **Startup** tab → pick the correct **Docker Image** (e.g. `Java 25` → the yolks image) → Save → Start; or simply delete and re-create it via `/server-create`. |
| `Pterodactyl Client API is not configured` | `PTERODACTYL_URL` + `PTERODACTYL_CLIENT_KEY` missing in `.env` |
| HTTP 401 from the panel | key invalid or revoked — create a new one (section 6) |
| HTTP 403 from the panel | key doesn't have permission for that action (client keys are per-user) |
| Ticket channel creation fails | bot needs **Manage Channels** permission in that category |
| AI says "**NaraRouter API error 401** — A valid API key is required" | the key IS set, but the service does not recognize the one the bot loaded. Since **v1.3.2**: ① the bot **auto-cleans** paste damage (wrapping quotes, line-wraps, invisible characters, a `Bearer ` prefix) and logs what it cleaned; ② it **verifies the key at startup** — look for `[ai] NaraRouter key verified ✓` or the `⚠ REJECTED` block in the console; ③ the 401 embed and `/settings` show a **masked fingerprint** of the loaded key — compare `sk-abc…wxyz` with your NaraRouter dashboard. Mismatch → the panel is loading an OLD `.env`: edit `.env` in the server Files tab, `NARA_API_KEY=sk-…` on **one line, no quotes**, then **Restart** the bot. Still rejected with the correct fingerprint → regenerate the key on the NaraRouter site (the old one may be revoked/expired) and restart. Also make sure you did not paste a Pterodactyl `ptlc_` key or a Discord token — the bot warns about both at startup |
| AI never answers in tickets | enable **Message Content Intent** (section 3) + `AI_TICKET_ENABLED=true` |
| `/ip` / `/list` said "server option not found" | fixed in **v1.4.1** — both commands now declare the `server:` option (required, autocomplete). Update to the current `index.js`, restart, and press Ctrl+R in Discord if the picker still shows the old command |
| "/optimize says **The PUT method is not supported for route …/files/write**" | fixed in **v1.5.0** — the panel's write route is POST-only and the installer was sending PUT. Uploads now use POST, and files over ~768 KB are downloaded **by wings itself** (`files/pull`) straight from the CDN into the server folder, so size limits can never bite. Update `index.js` and re-run `/optimize` |
| `/optimize` still can't detect the Minecraft version | since **v1.5.0** detection also asks the server **live** (real Minecraft status ping over TCP) when it's running, and when stopped reads `version_history.json`, rotated `.log.gz` logs and `world/level.dat`. Only a server that never started AND has no world can defeat this — start it once, or pass `version:1.21.4` + `loader:` once |
| `/ping` shows "running (not answering pings yet)" | the server process is up but the Minecraft port isn't answering the status handshake yet — it's still booting (large worlds take a minute), or the allocation port is firewalled. `/server status` shows the process state; the bot/panel latencies still work regardless |
| `/optimize` says "Could not detect the Minecraft version / server type" | since **v1.4.1** this is rare — detection reads startup variables, the rendered startup command, root jar names, `run.sh`, `logs/latest.log`, mods/plugins folders with `libraries/` probes and the egg name. If it still cannot, the error embed shows a **detection report** of every source it tried — pass the `version:` (e.g. `1.21.4`) and `loader:` (paper/purpur/fabric/forge/neoforge/quilt) options exactly as the Fix line suggests. A **vanilla** server cannot take optimization mods at all — switch to Paper or Fabric first |
| `/startup variables` shows no variables / `/startup set` says "Unknown variable" | fixed in **v1.4.1** — the panel's `/startup` endpoint returns a list envelope the bot didn't understand before; variables now display and validate correctly |
| `/optimize` skips a mod with "no build for Minecraft X" | correct behavior — that project has **no release for your exact version + loader** (e.g. Krypton is Fabric-only, spark's Modrinth project is mods-only). The report lists every skip with its reason; nothing incompatible is ever installed |
| `/list` shows a console tail instead of players | the log's player-list line wasn't found (some setups log differently) — the tail shows what the server answered; `/plugins run command:list` shows the raw output too |
| `/install` finds no CurseForge results | CurseForge search needs a (free) API key — set `CURSEFORGE_API_KEY` in `.env` (https://console.curseforge.com/) and restart. Modrinth always works without a key |
| Mods installed but server won't start | run `/optimize preview:true` and check the "Replaced" section — two versions of the same mod can conflict (the installer avoids this automatically); also confirm the mod matches your loader (fabric mods on fabric, plugins on paper). The panel console shows the exact startup error |
| `better-sqlite3` install error | restart the server (egg installs build tools) or use a newer Node.js egg |
| `/server-create` crashes at node selection with `page is not defined` | fixed in **v1.0.4** — replace `index.js` with the current file |
| `/server list` shows nothing | no servers granted — either get auto-linked (same Discord + panel username, section 13), run `/link`, or admin: `/permissions server-grant` |
| Auto-link matched the wrong user | run `/link` with the correct panel ID + email to override, or disable with `/settings autolink enabled:false` |
| Bot restarts wipe nothing | correct — SQLite lives in `data/`, which survives restarts (don't delete it) |
| `noctis ping @user` does nothing | (1) Check `/settings auto-ping` is enabled. (2) The phrase must start with the wake word (`noctis ping @user` / `@Noctis, ping him` — not "can you ping him"). (3) 10-second cooldown between requests — wait a moment. (4) `him/her/them` needs a recent other human message in the channel to resolve the target. (5) A ping storm not stopping? say **`noctis stop pinging`** (or `/settings auto-ping off`) — see `🛑 Stopped N ping storms`. |
| The AI doesn't auto-fix my server | (1) You must be **linked** to the panel (`/link`) — the diagnosis starts with your linked account. (2) The bot can only fix servers **assigned to you** (`/server list`); suspended servers are staff-only. (3) Destructive fixes (kill / optimize) always ask for a Confirm button via `/smart`. (4) The diagnosis is cached 90 s — a state change you just made may take a minute to be re-read. |
| `/lvl add` says "Levels disabled" | an admin must enable the system first: `/settings levels enabled:true` |
| `/afk` notice didn't show when I pinged someone | one notice per target per channel per minute (anti-spam) — wait 60 s, or check that the member is actually AFK (their status may have timed out via `duration`) |
| Where did `/convert-*` go? | **removed in v1.7.0** at the owner's request (all unit-conversion commands). The remaining suite hubs are `/text-*` `/fun` `/util` `/time` `/color` `/mc` |
| `/update` says "No editor key" | set `NARA_EDITOR_API_KEY=sk-…` in `.env` (ONE line, no quotes) — or leave it empty to use the main `NARA_API_KEY` — then **restart the bot**. `/settings` view and the boot console show which key is loaded (masked) |
| `/update` rejected the plan ("not found" / "matches N places") | v1.9.1 already retries this for you: anchors that only differ in whitespace apply anyway, and failed plans get **2 automatic repair rounds** with the exact errors. If it STILL fails, press **Show what the AI tried** to see the plan, then re-run the same `/update` with slightly different wording ("add a slash command /8ball…") — or set `NARA_EDITOR_MODEL` to a stronger code-tuned model |
| `/update` passed — now what? | press **Restart now** on the result (or restart on the panel). Changed/new commands re-register automatically on boot — press **Ctrl+R** in Discord if the picker lags. Changed your mind? **Undo (restore backup)** puts the previous code back, then restart again |
| Can /update break the bot? | it can only write code that passes `node --check` + structural sanity, every apply is backed up (`data/selfedit-backups/`, last 10) and **Undo** restores one click. Worst case, upload the newest backup file as `index.js` and Start — the panel start command never changes |
| I removed a command but it still shows in the picker | the command was deleted from the code, but Discord caches the picker — **restart the bot** (re-registration happens on boot) and press **Ctrl+R** in Discord. If it still shows, run `/admin sync` |
| `/update` removed the wrong thing / I regret it | press **Undo (restore backup)** on the result — it copies the pre-edit `index.js` back from `data/selfedit-backups/` — then restart. The 10 most recent backups are always kept |
| `/bot join` says "Missing packages" | run `npm install mineflayer mineflayer-pathfinder mineflayer-collectblock` in the bot console (a plain `npm install` after updating also works — they are in `package.json`), then Start again. The bot intentionally boots without them |
| `/bot join` times out / "Connection closed before joining" | the Minecraft server is not running or not reachable from the bot container — start it first (`/server start`). **v1.12.0 already resolves wildcard `0.0.0.0` allocations automatically** (alias → real IP → node FQDN → panel host, probed); if it STILL fails, the join error lists every address it tried — use the `host:` option (staff) with a known-good public address |
| `/bot join` kicked: "Failed to verify username" | the server is in **online mode** — set `MC_ACCOUNT_EMAIL` + `MC_ACCOUNT_PASSWORD` in `.env` (a Microsoft account that owns Minecraft), restart, and join again (real Microsoft auth; the username option is ignored) |
| `/bot diamonds` failed / stopped | read the last progress line — common causes: no trees or stone within range (move with `/bot goto`), lava blocked the descent (safety stop), the bot died (it respawns — run the quest again). `/bot stop` cancels cleanly any time |
| `noctis create a server` did nothing | (1) The phrase must START the message and contain "create/make/new + server" ("noctis create a server"). (2) You must be **linked** to a panel account (`/link`) — the reply tells you how. (3) 20s cooldown per user. Full control over nodes/eggs stays with the admin `/server-create` wizard |
| Just saying "noctis" does nothing now | it does — the bare wake word replies with a quick hint (10s cooldown per user). If you see nothing, check the bot has permission to send messages in that channel |

---

## 18. Security recommendations

1. **Never** share `.env`, your Discord token, or your `ptlc_`/`ptla_` keys. If exposed: reset the Discord token and delete + recreate both panel keys.
2. The bot **never stores API keys** in SQLite and the audit log **redacts** any `ptlc_`, `ptla_`, password or Bearer token before writing. The one deliberate exception (v1.1.1): panel login receipts from `/user create` keep the generated password — **encrypted at rest with AES-256-GCM**, keyed from `DISCORD_TOKEN` (which lives only in `.env`), so the database alone can never reveal them, and `/user creds` can re-DM them like ticket transcripts.
3. Database passwords are never requested from the panel — they can never be shown.
4. Keep `AI_AUTO_REPLY=false` until you trust the AI's answers in tickets.
5. Give the bot the **minimum** Discord permissions it needs (Manage Channels only where tickets live).
6. Restrict `/admin` usage by keeping `ADMIN_ROLE_ID` tight; remember Discord **Administrator** permission always maps to bot-admin.
7. Prefer a `ptlc_` key created on a dedicated "bot" panel user (owner/subuser of exactly the servers it should see) instead of your personal admin account.
8. Review the audit log regularly (`audit_logs` table or the `LOG_CHANNEL_ID` feed) for unexpected power/delete actions.
9. File operations are path-traversal protected (`..` is rejected before any panel call).
10. Host the bot on your own Pterodactyl panel or a server you control — it is a single Node process with no web surface to attack.
