# StreetCred — v1 Design

**Date:** 2026-10-15
**Status:** Approved in brainstorming, revised after the grill-me review, pending written-spec review

## 1. Purpose

StreetCred is a World of Warcraft: Forever addon that lets players rate each other on **reliability** and check a player's reputation before grouping or trading with them. It exists because WoW Forever removes dungeon summoning stones: travelling to a dungeon takes a long time, so being kicked, abandoned by a tank who leaves, or ninja-looted costs players far more.

StreetCred measures **reliability, not popularity**. Being boring, unfriendly or "not cool" is not a reason that exists in the system.

### Success criteria

- A player can target someone (or type their name) and see a trustworthy reputation within seconds, **without installing anything except the addon**.
- Rating takes one click for 👍 and a couple of checkboxes for 👎.
- Players who don't have the addon still get a reputation, because only the rater needs the addon.
- A single bad actor cannot fake ratings in someone else's name or flood a target with negatives.
- It's ready to release on CurseForge for the WoW Forever launch (about 20 days from the start of the project).

## 2. Platform facts (verified on the beta, build 1.60.1.70374)

Confirmed with a throwaway probe addon on 2026-10-15:

| Fact | Value |
|---|---|
| `## Interface:` | `16001` |
| `WOW_PROJECT_ID` | `18` (`WOW_PROJECT_CAMELOT`) |
| Lua | 5.1 |
| Engine | Modern (Retail-derived) client; DBM applies its Midnight-era restriction checks to Forever |
| Addon messages | `C_ChatInfo.RegisterAddonMessagePrefix` / `SendAddonMessage` work |
| Custom channel | `JoinTemporaryChannel` works; a client receives its own `CHANNEL` addon messages |
| Message size | 208 B delivered; **300 B reported success but was silently dropped**, so the payload limit is ≤255 B and we split messages ourselves |
| Player identity | No realms. Every character has a **primary and secondary name**; the pair is unique and permanent. `UnitFullName` returns `(primary, secondary)`; the addon-message sender arrives as `"Primary Secondary"` (separated by a space) |
| Secret values | `issecretvalue` exists; target names were **not** secret in the open world |
| Lockdown API | `C_ChatInfo.InChatMessagingLockdown()` and the `ADDON_RESTRICTION_STATE_CHANGED` event exist; lockdown was false in the open world |
| Present APIs | `C_FriendList`, guild roster, `GetNumGroupMembers`, `IsInInstance`, `GetInstanceInfo`, `UnitLevel`, `C_PartyInfo.GetLootMethod` (global `GetLootMethod` is gone), `C_EncodingUtil`, `C_Timer`, `C_EventUtils.IsEventValid`, `Settings.RegisterCanvasLayoutCategory` |
| Present events | `GROUP_ROSTER_UPDATE`, `GROUP_LEFT`, `PLAYER_ENTERING_WORLD`, `ZONE_CHANGED_NEW_AREA`, `TRADE_SHOW`, `TRADE_CLOSED`, `TRADE_ACCEPT_UPDATE`, `UI_INFO_MESSAGE`, `CHAT_MSG_ADDON`, `ENCOUNTER_START/END`, `PLAYER_REGEN_DISABLED/ENABLED`, `CHAT_MSG_CHANNEL_NOTICE` |

### Assumptions still to verify (can't be tested before launch: no group available on the beta)

The design is defensive about each of these, and all of them live in the adapter.

- A1. `PARTY`/`RAID` addon messages behave as on Retail.
- A2. Whispers must be addressed by the full `"Primary Secondary"` name (a whisper to the primary name only didn't arrive).
- A3. Whether lockdown applies inside dungeons, and whether unit names are secret there.
- A4. The custom channel's scope (assumed region-wide) and any server-side throttling of large channels.
- A5. The name format returned by guild roster and friends-list APIs (assumed to be the same `"Primary Secondary"` string, or something we can normalize to it).
- A6. How to read the trade partner's name (`UnitName("NPC")` during `TRADE_SHOW`, as on Retail) and how a completed trade is signalled (`UI_INFO_MESSAGE` with `ERR_TRADE_COMPLETE`).

## 3. Key decisions

| Decision | Choice | Why |
|---|---|---|
| Distribution | **Addon only.** No companion app, website or server in v1 | Anyone can install it from CurseForge. Addons can't make network calls, so a server would require a desktop app, which was a dealbreaker |
| Rating propagation | **Raters share only their own ratings.** Relayed or forwarded ratings are rejected | The sender identity is stamped by WoW and can't be forged, so every rating is authentic |
| Who can rate whom | Only players you **grouped or traded with in the last 24 hours**, and you must be **level 10 or higher** | Stops drive-by and throwaway-alt ratings |
| Ratings per pair | **One active rating per (author, target).** It can be changed or withdrawn | Prevents stacking |
| Rating content | 👍 = no extra input. 👎 = **at least one reason checkbox required**. **No free text anywhere** | Ratings stay tiny; no comments to moderate |
| Negative reasons | ninja looting · left mid-run · kicked without reason · no-show/AFK · trade scam · deliberate griefing | Reliability only. Toxic behavior is deliberately excluded |
| Reason weighting | All negative reasons count equally | Simple and predictable |
| Group aggregation | **One group = one vote**, decided by majority (a tie counts as 👎). Groups are identified by a roster fingerprint | Stops group ganging and premade boosting. Interim rule for v1; see §13 |
| Changing ratings | Withdraw at any time. Change only while eligible (within 24 h of the last interaction). A new interaction allows a new rating that replaces the old one | Second thoughts are allowed; revenge flips aren't |
| Language | English at release; every visible string goes through a locale table | Translations can be added later; synced data doesn't depend on language |
| Rater anonymity | Raters' names are hidden in the UI. Ratings from your guildmates and friends are highlighted | Gives a trust signal without public naming. True anonymity is technically impossible, which the addon description says |
| Rating trigger | `/streetcred` manually, plus an end-of-group prompt **only after real content** | Gets enough ratings coming in without spamming players |
| Retention | Ratings expire **90 days** after their timestamp. Hard cap of **300k stored ratings**, oldest dropped first | Keeps saved data under about 20 MB; old behavior stops counting |
| Score display | **Colored tier word + raw percentage + counts**, e.g. `Solid · 80% (5 groups, 12 raters)` | The word is the quick check; the numbers help with borderline cases |

**Out of scope for v1:** the mouseover tooltip line, a website/server/companion app, comments, positive tags, rating relays, cryptographic signatures, a waiting period for new raters, per-rater daily caps, a toxic-behavior flag, and translations.

**Release policy:** the full v1 scope ships; the addon is released when it's done, not cut down to hit launch day. Implementation is ordered core-first (storage, scoring, sync), then UI and polish.

## 4. Architecture

```
UI (popup, end-of-group prompt, slash command, settings)
        │
        ▼
CORE — pure Lua 5.1, never calls a WoW API
  RatingStore   all known ratings; my own ratings; author sync state
  Interactions  who I grouped/traded with and when (eligibility)
  GroupSession  detects "real content" and builds the prompt roster
  Score         Bayesian score → tier
  SyncProtocol  HELLO/WANT/BATCH/RATE logic, request suppression, merging
  Codec         message encode/decode, chunking/reassembly
  SendQueue     rate limiting + lockdown hold
        │  (one adapter interface)
        ▼
ADAPTER — the only WoW-specific code
  Events · Comm (send/receive, channel join) · Storage (SavedVariables)
  Social (guild + friends) · Clock (GetServerTime) · PlayerInfo (own name, level, unit names)
```

- Core modules communicate only through the adapter interface. Tests replace it with a fake adapter.
- Modules share state through the addon's namespace table (`local _, ns = ...`). A small test loader builds an equivalent namespace for `busted`.

### File layout

```
StreetCred/
  StreetCred.toc            ## Interface: 16001, SavedVariables: StreetCredDB
  src/core/                 codec.lua, send_queue.lua, rating_store.lua, interactions.lua,
                            group_session.lua, score.lua, sync.lua
  src/adapter/wow.lua       real adapter
  src/ui/                   popup.lua, group_prompt.lua, settings.lua, slash.lua
  src/init.lua              wires adapter → core → UI
spec/                       busted unit tests
spec/sim/                   fake adapter + multi-player simulator + scenario tests
.github/workflows/ci.yml    busted + luacheck
```

## 5. Data model

### Player ID

`"Primary Secondary"`: the exact string the client gives as the addon-message sender, built from `UnitFullName` as `primary .. " " .. secondary`. Because the pair is unique and permanent, no GUID is needed.

### Rating

| Field | Type | Notes |
|---|---|---|
| author | PlayerId | **Set by the receiver from the message sender**, never read from the message itself |
| target | PlayerId | |
| verdict | `1` / `-1` / `0` | 0 = withdrawn |
| reasons | bitmask | Only for `-1` and must be non-zero. Bits: 1 ninja, 2 left_mid_run, 4 kicked_no_reason, 8 no_show_afk, 16 trade_scam, 32 griefing |
| context | `D`/`R`/`G`/`T` | dungeon, raid, open-world group, trade |
| instanceId | integer or 0 | From `GetInstanceInfo`; 0 for G/T |
| group | 8 hex chars | Group fingerprint: a hash of the sorted `PlayerId`s present the last time the target was in the rater's group (both names for a trade). Every member of a group computes the same value independently |
| time | integer | Author's `GetServerTime()` |
| seq | integer | The author's counter; goes up by 1 on every new or changed rating |

### Saved data

```lua
StreetCredDB = {
  schema = 1,
  mySeq = 0,
  -- each rating is one compact string: "verdict;reasons;context;instanceId;group;time;seq"
  ratings = { [target] = { [author] = "-1;1;D;389;a91c03f2;1760500000;43" } },
  mine = { [target] = "..." },            -- my own ratings, used for BATCH replies
  authors = { [author] = { haveThrough = 0, lastSeen = 0 } },
  interactions = { [playerId] = { time = 0, context = "D", group = "a91c03f2" } },
  firstSync = 0,                          -- server time of the first channel join (for the "Syncing…" state)
  settings = { groupPrompt = true, shareRatings = true },
}
```

- Ratings are decoded only when needed. Scores are cached per target and recomputed only when one of that target's ratings changes.
- **Storage cap:** above 300k stored ratings, the ratings with the oldest `time` are dropped first.

- **Expiry:** when the addon loads and on every merge, ratings with `time < now − 90 days` are removed.
- **Withdrawn ratings** (verdict 0) are kept as markers until they expire, so `BATCH` replies carry withdrawals to clients that were offline. They're ignored by the score.
- **Timestamps more than 1 day in the future** are rejected.
- **Unknown `schema`:** the old table is moved to `StreetCredDB_backup` and a fresh store is started.

## 6. Sync protocol

### Transport

- Prefix `STRC`.
- **Channels (modelled on AutoLayer):** the static `StreetCred` plus **2 daily channels** named `SCred` + an 8-hex-digit hash of the server's UTC date in two formats (`yyyy-mm-dd`, `dd-mm-yyyy`). The hash is a small built-in Lua function (no LibDeflate).
  - **Join them all at once**, so addon users are almost always the owners and an empty channel is never left for a griefer to claim.
  - **Join tomorrow's channels early,** from 3 h before the UTC date changes.
  - **Leave old channels** after the date changes. Our channels must not end up in the General/Trade slot numbers; if one does, leave it and rejoin 30 s later.
  - **A channel that fails to join** (password-locked, banned) is marked failed for the rest of the day and skipped.
  - **Hide** every joined channel from all chat frames.
  - **Send to every channel that works; drop duplicates when receiving** (by author + seq, or by message ID for HELLO/WANT).
- Every message: `1|TYPE|field;field;...` (`1` = protocol version). Unknown versions are dropped.
- Payloads over 240 B are split: `1|C|msgId;idx;total|<part>`. Reassembly gives up after 30 s.
- All sends go through the **SendQueue**: a token-bucket rate limit (starting at 800 B/s and 10 messages of burst; to be tuned) that **holds everything while `InChatMessagingLockdown()` is true** and releases on `ADDON_RESTRICTION_STATE_CHANGED` / `PLAYER_REGEN_ENABLED`.

### Messages

| Type | Fields | Sent |
|---|---|---|
| `RATE` | target; verdict; reasons; context; instanceId; group; time; seq | When the player submits, changes or withdraws a rating. Sent to the channel |
| `HELLO` | seq (author's `mySeq`) | On login; every 60 ± 15 min after that (**30 ± 10 min for the first 7 days after release**, set by a release-date constant); and to PARTY/RAID on `GROUP_ROSTER_UPDATE` (at most once per minute) |
| `WANT` | author; fromSeq | When an author's `HELLO` seq is above our `haveThrough` |
| `BATCH` | through; then one or more ratings | An author's reply to `WANT`s. Sent to the channel and split into chunks |

### Rules

1. **Authenticity:** for `RATE` and `BATCH`, the author is always the message sender. Ratings are never forwarded on someone else's behalf, so there is nothing to forge.
2. **Merge:** a rating replaces the stored one for the same (author, target) only if its `seq` is higher.
3. **`haveThrough`:** a live `RATE` with `seq == haveThrough + 1` moves it forward by one. A complete `BATCH` sets it to `through`. Any other `RATE` is stored, but `haveThrough` stays put.
4. **Request suppression:** before sending a `WANT`, wait a random 0–10 s. If in that time we see someone else's `WANT` for the same author with `fromSeq ≤` ours, don't send ours.
5. **Author batching:** an author collects `WANT`s for 5 s and then broadcasts one `BATCH` containing all of its non-expired ratings with `seq ≥` the lowest requested `fromSeq`. The same range isn't re-broadcast within 60 s.
6. **Sharing opt-out:** with `shareRatings = false`, the client sends no `RATE`, `HELLO` or `BATCH`, but still receives.
7. **Junk-sender muting:** a sender with more than 10 invalid messages within 10 min is ignored locally for 1 h.
8. **Backing off under load:** if more than 50 channel messages per second arrive over a 1-minute window, the `HELLO` interval doubles (up to 4 h). It returns to normal once the rate falls back.

## 7. Eligibility and group sessions

- **Interactions:**
  - While grouped, every member is recorded with `time = now` on each `GROUP_ROSTER_UPDATE` and every 60 s.
  - The trade partner is captured on `TRADE_SHOW` and recorded with context `T` when the trade completes.
  - Entries older than 24 h are pruned.
- **Group fingerprint:** each interaction stores the fingerprint of the roster at that moment (the sorted member `PlayerId`s, including yourself and the target). The stored value is overwritten on every roster update while the target is present, so it ends up as "the roster the last time the target was here".
- **Can rate X** = own level ≥ 10 **and** an interaction with X within 24 h **and** X isn't yourself.
- **Can change a rating of X** = can rate X. **Can withdraw** = always. A new rating replaces your previous one for that target and takes the current interaction's fingerprint.
- **Group session:** starts when a group forms or the player joins one, and ends on `GROUP_LEFT`. It counts as **real content** if the group was inside a party or raid instance together at any point, **or** the session lasted at least 15 minutes. If so, the prompt appears once the player is out of combat. It lists every member who was present during the session.

## 8. Score

For a target, take every stored, non-expired rating with verdict ±1 and **group them by fingerprint into votes**:

- **Each group is one vote:** 👍 if most of that group's ratings are 👍, otherwise 👎 (a **tie counts as 👎**). Each trade fingerprint is its own vote.
- With `pos` 👍 votes and `n` votes in total:
  - `n < 5` → **Unknown** (gray, no percentage). Within 75 min of `firstSync`, or of a login after more than 7 days away, **Syncing…** is shown instead of Unknown.
  - Otherwise `score = (pos + 9) / (n + 10)` (a Bayesian average: every player starts with 9 imaginary 👍 and 1 imaginary 👎).
- **Trusted** ≥ 0.90 (green) · **Solid** ≥ 0.82 (light green) · **Mixed** ≥ 0.77 (yellow) · **Risky** < 0.77 (red).
- Display: `Tier · round(100·pos/n)% (n groups, m raters)`, where `m` is the number of individual ratings.
- Combining ratings into votes is a single function in `score.lua` (`aggregateVotes`), so the interim rule can be replaced later (§13) without touching storage, sync or the UI.

Reference cases (in votes; these are also unit tests):

| Votes | Shown | Score | Tier |
|---|---|---|---|
| 4👍 1👎 | 80% (5 groups) | 0.867 | Solid |
| 75👍 25👎 | 75% (100 groups) | 0.764 | Risky |
| 5👍 0👎 | 100% (5 groups) | 0.933 | Trusted |
| 2👍 3👎 | 40% (5 groups) | 0.733 | Risky |
| 8👍 2👎 | 80% (10 groups) | 0.850 | Solid |
| 40👍 10👎 | 80% (50 groups) | 0.817 | Mixed |
| 20👍 2👎 | 91% (22 groups) | 0.906 | Trusted |
| 3👍 1👎 | 75% (4 groups) | — | Unknown |

Vote-combining cases (also unit tests):

| Ratings | Votes |
|---|---|
| 4 raters in one group, all 👎 | 1👎 |
| One group: 3👍 1👎 | 1👍 |
| One group: 2👍 2👎 | 1👎 (tie) |
| 4 raters in one premade, all 👍, no other ratings | 1👍, so still Unknown |

The thresholds are constants in `score.lua`, to be tuned after launch.

## 9. UI

- **All visible text** goes through a locale table (`L["..."]`); English only in v1. Tier colors are the same in every language.
- **`/streetcred`** or **`/scred`** (current target), or with a name (`/scred Primary Secondary`):
  - The popup shows the name, colored tier word, percentage and count, 👍/👎 totals, a breakdown of negative reasons (`Ninja looting ×1`), "★ N guildmates/friends rated 👍/👎", "Data from N raters", and your own rating.
  - If you can rate: **[👍] [👎]**. 👎 expands the reason checkboxes and needs **[Submit]**; nothing is saved without an explicit Submit.
  - If you've already rated: **[Change] [Withdraw]**.
  - Once the rating window has passed, only **[Withdraw]** remains.
  - Opening it on yourself shows your own score without rating buttons.
- **`/streetcred debug`** prints sync counters (messages in/out, dropped by reason, queue depth, authors known).
- **End-of-group prompt:**
  - Shows the instance name and duration, then one row per member with [👍] [👎]. 👎 expands the reasons inline.
  - **[Submit]** saves every rated row in one go. **[Skip]** or ✕ dismisses it.
  - Never opens in combat.
- **Completed trade:** no prompt, just one chat line: `StreetCred: rate your trade with X — /streetcred X`.
- **Settings panel** (`Settings.RegisterCanvasLayoutCategory`): turn the group prompt on or off, and turn rating sharing on or off.

## 10. Error handling

- No StreetCred error should reach the player. Event handlers are wrapped so a failure in one module can't break the others.
- Bad incoming messages (wrong version, can't be parsed, sender ≠ author, incomplete chunks, invalid fields such as a `-1` with no reasons) are dropped silently and counted for `/streetcred debug`.
- If joining the channel fails, it's retried at 30 s, 60 s, 120 s and so on (at most every 10 min). Syncing within groups keeps working.
- Rating windows refuse to open in combat. Outgoing messages wait in the queue during lockdown.

## 11. Testing

- **Unit (busted, Lua 5.1):** codec round-trips and chunking at the 255 B limit; score and vote-combining reference tables; group fingerprints (the same roster gives the same hash regardless of order); daily channel names; eligibility (level, 24 h window, self, change vs. withdraw); merge and `haveThrough` rules; expiry and the 300k cap; request suppression timing; junk-sender muting; schema migration and backup.
- **Simulation:** the fake adapter runs N virtual players with a shared channel, sender stamping, the 255 B drop rule, lockdown toggling and login/logout. Required scenarios:
  1. A rates B while C is offline; C logs in later and catches up through HELLO/WANT/BATCH.
  2. A forged `RATE` whose fields claim to come from A but is sent by D is attributed to D, not A.
  3. Ten clients all missing A's ratings send at most a few `WANT`s, and A sends one `BATCH`.
  4. A rating sent during lockdown goes out after lockdown ends.
  5. A changed rating replaces the old one everywhere, and withdrawing removes it from the score.
  6. A group session with an instance entry produces the prompt; a 5-minute questing group doesn't.
  7. Clients split across different working channels (one channel locked) still receive every rating exactly once.
  8. Four group members who all rate the same target 👎 produce one 👎 vote on every client.
- **In-game smoke test (beta):** load the addon with no errors, check `/streetcred` on a target, and check that a rating made on one character reaches a second character over the channel.
- **CI:** GitHub Actions runs `busted` and `luacheck` on every push.

## 12. Release

Published on CurseForge (and Wago) using the BigWigs packager GitHub Action, with `## Interface: 16001`. The addon description explains the reliability-only purpose, that rater names are hidden in the UI but not truly anonymous, and the 90-day retention.

The throwaway `SCProbe` addon is removed from the beta install before release work starts.

## 13. Future brainstorming topics (not in v1)

- **Corroboration:** several matching ratings from one group shouldn't count as several votes, but they should make that group's vote more *credible*, relative to how many members of that group had the addon.
- **Split groups:** what it means when 2 members of a group call a player unreliable and 2 call them reliable, and whether a tie should really count as 👎.
- **Server-backed v2:** an optional website or companion app if StreetCred takes off, closing the gap where determined abusers prepare alts in advance with a modified addon.
