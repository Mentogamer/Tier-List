# 🏆 Mento's Tier Test Bot

Discord tier-testing bot for Minecraft PvP communities: profiles, per-gamemode queues, private test tickets,
tier + tester roles, `/result` and `/high-result`, permanent history, logs, and a web tier list / profile panel.

**Stack:** Node.js 20/22 LTS · TypeScript · discord.js v14 · SQLite (better-sqlite3, versioned SQL migrations) · Express · vanilla JS site.
Bot and website share one database file, so a Discord result shows on the site immediately.

## 1. Discord Developer Portal (once)
1. https://discord.com/developers/applications → **New Application**.
2. **Bot** tab → *Reset Token* → copy it (`DISCORD_TOKEN`). Enable **Server Members Intent** (required for tester-role sync).
3. **General Information** → copy *Application ID* (`CLIENT_ID`). Enable Discord Developer Mode, right-click your server → *Copy Server ID* (`GUILD_ID`).
4. Optional website login: **OAuth2** → copy *Client Secret* (`DISCORD_CLIENT_SECRET`) and add redirect `http://localhost:3000/auth/callback`.

## 2. Invite the bot
Replace `YOUR_CLIENT_ID`:

```
https://discord.com/oauth2/authorize?client_id=YOUR_CLIENT_ID&permissions=268553232&scope=bot%20applications.commands
```
Permissions: Manage Roles, Manage Channels, View Channels, Send Messages, Embed Links, Attach Files, Read Message History.
**Drag the bot's role ABOVE the tier/tester roles** (Server Settings → Roles), otherwise it cannot assign them.

## 3. Windows setup from zero (PowerShell)
```powershell
winget install OpenJS.NodeJS.LTS          # then close and reopen PowerShell
cd path\to\mentos-tier-test-bot
npm install
copy .env.example .env
notepad .env                              # fill in DISCORD_TOKEN, CLIENT_ID, GUILD_ID, SESSION_SECRET
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"   # paste output as SESSION_SECRET
npm run build
npm run migrate                           # creates data\tiertest.db + seeds tiers and the 9 default gamemodes
npm run register                          # registers slash commands (also automatic on bot start)
npm start                                 # bot + website together
```
Separate processes if you prefer: `npm run start:bot` and `npm run start:web`. Website: http://localhost:3000.
Linux/macOS: identical, use `cp` instead of `copy`. Development with auto-reload: `npm run dev`.

## 4. First-time server setup (in Discord, as admin)
```
/setup-tier-testing          creates everything below (safe to run again; reuses what exists)
/tier-role-create uhc        optional: the full LT5…HT1 ladder (setup only guarantees LT5 + HT5; others are created on demand by /result)
/tester-role-create-all      creates every "🧪 <Gamemode> Tester" role
/tester-role-add uhc @User
/high-result-permission add @User
```
`/setup-tier-testing` creates / reuses:
- **REQUEST TEST**: `#profile` (profile panel), `#result`, `#high-result`, `#tier-list` — nothing else
- **WAITLIST**: one channel per gamemode (`#sword` … `#cart`), each with its own persistent queue panel. Gamemode channels created by an earlier version inside REQUEST TEST are **moved** to WAITLIST, not duplicated
- **TIER TEST TICKET**: tickets created by `/drag` (unchanged)
- **TIER TEST LOGS**: `#tier-test-logs`, `#queue-logs`, `#ticket-logs`, `#tester-logs`, `#result-logs` (unchanged)
- a mentionable role per gamemode (`@UHC`…) and `LT5-<Gamemode>` / `HT5-<Gamemode>` tier roles. An existing role such as `HT5 UHC` is reused (names are compared ignoring dashes/case)
- the report lists missing bot permissions and role-position problems.

## 5. How things work
- **Tiers:** stored per player *per gamemode*. A player with no recorded result in a gamemode is **Not Tested** there; a recorded LT5 shows LT5. Nothing is ever assigned without a result. There is no previous-tier question.
- **Profile (#profile):** modal for the Minecraft IGN → multi-select of gamemodes; nothing is saved until both are done. Gamemode roles follow the selection; changing it never erases results.
- **Queues (WAITLIST):** FIFO, persistent, one panel per channel (OPEN/CLOSED, members, count, Join/Leave). Join needs a profile that includes the gamemode, an open queue, no duplicate entry, no active ticket. `/queue-open` pings the gamemode role once; `/queue-close` keeps entries; `/queue-clear` empties; `/drag <gamemode>` creates the private ticket for the first player (entry removed only after the ticket exists; failures keep the player's place).
- **/result `tier` `player` `gamemode`:** testers of that gamemode or admins. Saves the result (player ID, IGN, gamemode, tier, tester, timestamp, ticket ID if the player has an active ticket for it), updates the tier, swaps **only that gamemode's** tier role (`LT5-UHC` → `HT5-UHC`), posts the embed in `#result`, logs, replies privately. If Discord rejects the role change the result is still saved, the reply says the role was **not** assigned, it is flagged `failed` in the database and `/result-retry` (and the bot at startup) retries it. Announcements are claimed atomically so a retry never posts twice.
- **High results:** `/high-result tier player gamemode` needs the separate permission (`/high-result-permission`). Tiers at or above the configured minimum also count as high (default **HT3**; change with `/high-result-criteria`, `none` disables) and then need that permission too. High results are announced in `#high-result` **and** `#result`; LT5/HT5 are not high by default.
- **/result-correct** (admin) adds a correcting result that references the original; the old record stays, the correction is announced and logged.
- **/tier-list:** admins, queue managers or testers publish/refresh `#tier-list`: players grouped by tier per gamemode from each player's **latest** result, "Not Tested" for players who picked a gamemode but were never tested. Messages are edited in place (split when long) and refreshed automatically after results.
- **Restart safe:** all state is in SQLite; at start the bot repairs tickets and panels, re-syncs tester roles, retries failed role updates and unsent announcements.

## 6. Commands
Admin: `/setup-tier-testing /panel-setup /gamemode-add /gamemode-remove /tier-role-create /tier-role-delete /tester-role-create /tester-role-create-all /tester-role-add /tester-role-remove /tester-role-list /high-result-permission add|remove|list /high-result-criteria /result-correct /web-setup`
Queue managers / testers: `/queue-open /queue-close /queue-clear /drag /result /result-retry /tier-list`; `/high-result` with the High Result permission
Everyone: `/gamemode-list /profile /profile-edit /queue-status /queue-list /queue-leave /test-info /history /stats /help`

## 7. Database / migrations
`src/db/migrations.ts` holds numbered SQL migrations (tracked in `schema_migrations`). Add a new entry to `MIGRATIONS` and run `npm run build && npm run migrate` (also runs on every start). Tables: users, profiles, profile_gamemodes (migration 2; migration 3 adds result announcement/role-retry/correction columns), gamemodes, tiers, profile_last_tiers, player_tiers, role_mappings, tester_assignments, permissions, queues, queue_entries, test_tickets, tests, test_results, bot_config, web_sessions. Back up by copying `data/tiertest.db` while the bot is stopped.

## 8. Production notes
Put the site behind HTTPS (Caddy/nginx), set `WEB_PUBLIC_URL=https://…` and `OAUTH_REDIRECT_URI`, add that redirect in the portal. Use `pm2` or a service to keep it running.
