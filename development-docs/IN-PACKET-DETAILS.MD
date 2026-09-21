# Incoming Packet Details — `WorldMode.handleNetworkPacket`

This document catalogs every server→client (map-server) packet handled by
**`WorldMode.handleNetworkPacket`** in `game/world_packets.go:53-1359` — the
packet dispatch chain that runs once per received packet while the player is
in-game (on a map, as opposed to at the login/character-select screen, which
is handled by a separate chain in `game/login.go`).

It assumes you already understand the general networking machinery described
in `development-docs/ARCHITECTURE.md` (§"Networking"): packet framing via
`network.Framer`, the per-frame `Client.Pump`/`DrainPackets` handoff between
the socket goroutines and the game loop, and the
`network.ParseXxx(pkt) (Result, bool, error)` convention where `bool` means
"this packet's opcode matched" and `error` means "matched but malformed."
This document goes one level deeper: what each individual packet means, when
the server sends it, and exactly what the client does with it.

**Outgoing (client→server) packets are a different topic and not covered
here.** Each `network/*_packets.go` file also contains `BuildXxxPacket`
helpers and `Client.SendXxx` methods for the outbound direction; those are
mentioned below only where they close the loop on a request/response pair
(e.g. a shop-buy request that produces the `ShopResult` this document
documents the *receiving* side of).

A recurring pattern worth knowing up front: many packet families were
revised multiple times across RO's packet-version history, and `goro` is
pinned to the **2008-09-10** pre-renewal packet version (see
`network/packet.go`'s `PacketLengths2008()`). Several `ParseXxx` functions
therefore `switch` on more than one opcode — e.g. `ParseActorEntry` alone
handles **twelve** different opcodes (`0x0078` through `0x0856`) because RO's
"an actor appeared" packet was re-shaped many times as the client added
guild emblems, levels, and robe/costume slots. Where that happens below, the
newest/most relevant variant for 2008-09-10 is called out, not every legacy
opcode.

## Pre-filtered packet families

Three packet families are peeled off **before** the main dispatch chain even
starts (`game/world_packets.go:54-65`), each by a small dedicated helper.
They are not part of the ~150-packet chain this document catalogs in depth —
each is a separate subsystem with its own packet set:

- **Quests** — `m.handleQuestPacket(ctx, pkt)` (`game/quests.go:11`). Handles
  `network.ParseQuestList`, `ParseQuestMissions`, `ParseQuestAdd`,
  `ParseQuestDelete`, `ParseQuestHunt`, `ParseQuestActive`
  (`network/quest_packets.go`) — the quest journal's list/mission/hunt-progress
  packets (`ZC_ALL_QUEST_LIST`/`ZC_ADD_QUEST`/etc., opcodes `0x02B1`-`0x02B7`).
- **Mail** — `m.handleMailPacket(ctx, pkt, now)` (`game/mail.go:183`). Handles
  `network.ParseMailWindow`, `ParseMailList`, `ParseMailRead`,
  `ParseMailNew`, `ParseMailResult` (`network/mail_packets.go`) — the legacy
  2008-era mail system's window-open/list/read/new-mail/send-result packets.
- **Disconnect notices** — `handleDisconnectPacket(ctx, &m.ui.disconnectDialog, pkt)`
  (`game/disconnect_dialog.go:86`). Handles `network.ParseNotifyBan`
  (`SC_NOTIFY_BAN`, `0x0081`) and inter-server-disconnect notices
  (`network.IsInterServerDisconnect`, `network/disconnect_packets.go`) — server-forced
  disconnection/kick reasons, surfaced as a modal dialog rather than routed
  through the main chain.

If a packet you're looking for isn't in the tables below, check these three
files first before assuming it's unhandled.

---

## 1. Map & session lifecycle

These packets govern entering a map, changing maps, and leaving the game —
the events that can end the current frame early by transitioning modes
(`handleNetworkPacket` returns `(Mode, true)`).

| Packet | Parser | `file:line` | What it means |
|---|---|---|---|
| `ZC_NOTIFY_MAPINFO` | `ParseMapInfoNotify` | `game/world_packets.go:66-71` | Generic result code for a map-related server action; applied via `m.applyMapInfoNotify(notify)`. |
| `ZC_NOTIFY_MAPPROPERTY` | `ParseMapPropertyNotify` | `game/world_packets.go:72-77` | The current map's PvP/GvG/WoE property flag; `applyMapPropertyNotify(ctx, property)` updates `world.MapProperty`, which gates combat rules throughout `game` (see `IsPvP`/`IsGvG`/`PlayerCombatEnabled` in ARCHITECTURE.md). |
| `ZC_MAPMOVE`/`ZC_CHANGEMAPSERVER` (`0x0091`/`0x0092`/`0x0AC7`) | `ParseMapChange` | `game/world_packets.go:238-243` | The server is sending the player to a new map — either same-server (`0x0091`) or a different map server (`0x0092`/`0x0AC7`, which also carries the new server's IP/port). Triggers `m.startMapFadeOut(change, time.Now())` and **returns `(nil, true)`** — this stops the current frame's packet loop mid-drain (with the tail deferred, see `handleNetworkPackets`, `game/world_packets.go:17-48`) so the map transition can complete cleanly. |
| `ZC_ACCEPT_ENTER`/`ZC_ACCEPT_ENTER3` (`0x0073`/`0x02EB`) | `ParseMapAcceptEnter` | `game/world_packets.go:244-252` | The server has finished loading the player onto the new map and confirms the spawn tick/position. `applyMapAcceptEnter(ctx, enter)` applies it, then `sendLessEffectPreference(ctx)` re-sends the client's "less effects" setting (servers reset it per-map). If `m.pendingWarp` was set (an in-progress player-initiated warp), this **also returns `(m.nextWorldMode(), true)`** — building a fresh `WorldMode` for the new map. |
| `ZC_UPDATE_MAPINFO` (`0x0192`) | `ParseMapCellUpdate` | `game/world_packets.go:301-306` | A single GAT cell's walkability/type flags changed live (e.g. a guild dungeon barricade going up or down). `m.applyMapCellUpdate(ctx, update)` patches the loaded `*res.GAT` in place. |
| `ZC_RESTART_ACK` | `ParseRestartAck` | `game/world_packets.go:259-271` | Response to the player choosing "return to character select" after death or via the escape menu. If `m.ui.escapeMenu.ApplyRestartAck(ack)` reports the escape menu was waiting on this specific ack, the client starts a character-select fade-out and **returns `(nil, true)`**; if refused while the player expected character-select (`refusedCharacterSelect`), a console message explains the refusal instead. |
| `ZC_ACK_REQ_DISCONNECT` | `ParseQuitGameAck` | `game/world_packets.go:272-277` | Response to a client-requested "quit to desktop." `m.applyQuitGameAck(ctx, ack)` applies the allow/deny result. |

## 2. Movement & positioning

Ragnarok movement is server-authoritative but client-interpolated (see
`world.Actor`'s `FromX/FromY/ToX/ToY/MoveStartTick` fields in
ARCHITECTURE.md). These packets are how the server corrects or confirms
positions.

- **`ZC_NOTIFY_PLAYERMOVE`** (`ParseSelfMoveAck`, `game/world_packets.go:307-317`)
  — the server's acknowledgement that the local player's walk request was
  accepted, carrying the authoritative `FromX/FromY/ToX/ToY` and server tick
  the move started at. `applySelfMoveAck(ctx, ack)` reconciles local
  prediction against it, then `m.clearLocalActorAction(ctx)` and three
  `continuePending*` calls (`continuePendingAttack`, `continuePendingPickup`,
  `m.skills().ContinuePendingTarget`) resume any attack/pickup/skill-cast
  that was queued behind the walk completing — this is the hook that lets
  "click to attack a target out of range" work: the client walks toward the
  target and re-attempts the attack once each walk-ack arrives.
- **`ZC_STOPMOVE`** (`ParseActorSetPosition`, `:318-331`) — the server is
  forcibly placing an actor (local player or otherwise) at an exact
  position, overriding any in-progress interpolation. `isLocalActor(ctx, position.ID)`
  gates a debug log and, for the local player specifically, the same three
  `continuePending*` calls as above (a position correction can also unblock
  a queued action).
- **`ZC_HIGHJUMP`** (`ParseActorJumpPosition`, `:332-340`) — an
  instantaneous jump/teleport-style repositioning (used by some skills and
  NPC scripts), applied via `applyActorJumpPosition(ctx, position)`.
- **`ZC_ATTACK_FAILURE_FOR_DISTANCE`** (`ParseAttackFailureForDistance`,
  `:1328-1333`) — the server rejected an attack because the target moved out
  of range between the client's request and the server processing it; carries
  both parties' positions and the allowed range so the client can react (e.g.
  re-path). Applied via `m.applyAttackFailureForDistance(ctx, failure)`.

## 3. Chat & social messaging

### General/public/system chat

`ZC_NOTIFY_CHAT`, `ZC_NOTIFY_PLAYERCHAT`, `ZC_BROADCAST`, `ZC_BROADCAST2`,
`ZC_NPC_CHAT`, `ZC_MSG`, `ZC_MSG_VALUE`, `ZC_MSG_SKILL`, and `ZC_MSG_COLOR`
all fold into one parser, **`ParseChatMessage`**
(`game/world_packets.go:127-132`, `network/chat_packets.go:235-318`), and one
handler, `m.handleChatMessage(ctx, chat, now)`. This single packet family
covers: ordinary nearby-actor speech bubbles (`GID` + `Text`), the local
player's own outgoing echo, server-wide announcements (`Announcement: true`,
with color/font styling on the `ZC_BROADCAST2` variant), NPC name-tagged
chat, and three "look up message N in the client's string table" variants
(`ZC_MSG`/`ZC_MSG_VALUE`/`ZC_MSG_SKILL`, which carry a numeric `MessageID`
resolved against `msgstringtable.txt` via `res.Manager` rather than inline
text) plus a colorized system-message variant. `ChatMessage.HasColor`/`Color`
distinguish styled from plain messages.

### Whispers

| Packet | Parser | `file:line` |
|---|---|---|
| `ZC_WHISPER` | `ParseWhisperMessage` | `:163-169` |
| `ZC_ACK_WHISPER` | `ParseWhisperAck` | `:170-177` |
| `ZC_SETTING_WHISPER_PC`/`ZC_SETTING_WHISPER_STATE` | `ParseWhisperIgnoreAck` | `:178-183` |

An incoming whisper (`ZC_WHISPER`) is both echoed to the console
(`addWhisperMessage`) and routed into a per-sender whisper window
(`m.addWhisperWindowIncoming(ctx, whisper)`). `ZC_ACK_WHISPER` is the result
of the *local* player's own outgoing whisper (delivered/recipient offline/
recipient ignoring whispers/etc.) — the code comment at `:173-174` notes this
ack carries no recipient and can arrive out of order across map servers, so
it's logged to console without trying to match it to a specific conversation.
`ZC_SETTING_WHISPER_PC`/`_STATE` acknowledge a per-name or global
whisper-ignore toggle the player set.

### Chat rooms

A full lifecycle: create → board (list) entries appear/disappear for other
players → enter (with member roster) → members join/leave → room
metadata changes → role (owner) changes.

| Step | Packet | Parser | `file:line` | Handler |
|---|---|---|---|---|
| Create result | `ZC_ACK_CREATE_CHATROOM` | `ParseChatRoomCreateAck` | `:184-189` | `m.handleChatRoomCreateAck` |
| Room appears on map | `ZC_ROOM_NEWENTRY` | `ParseChatRoomBoard` | `:190-195` | `m.applyChatRoomBoard` |
| Room disappears | `ZC_DESTROY_ROOM` | `ParseChatRoomDestroy` | `:196-201` | `m.applyChatRoomDestroy` |
| Player enters, gets roster | `ZC_ENTER_ROOM` | `ParseChatRoomEnter` | `:202-207` | `m.handleChatRoomEnter` |
| Someone joins | `ZC_MEMBER_NEWENTRY` | `ParseChatRoomMemberJoin` | `:208-213` | `m.handleChatRoomMemberJoin` |
| Someone leaves (or is kicked) | `ZC_MEMBER_EXIT` | `ParseChatRoomMemberLeave` | `:214-219` | `m.handleChatRoomMemberLeave` |
| Title/limit/visibility changed | `ZC_CHANGE_CHATROOM` | `ParseChatRoomChange` | `:220-225` | `m.handleChatRoomChange` |
| Ownership transferred | `ZC_ROLE_CHANGE` | `ParseChatRoomRoleChange` | `:226-231` | `m.handleChatRoomRoleChange` |

### Emotions

**`ZC_EMOTION`** (`ParseEmotionNotify`, `game/world_packets.go:232-237`,
`network/emotion_packets.go:26-37`) — any actor (including the local player,
echoed back) played an emote (`/heh`, `/swt`, etc.). `m.applyEmotionNotify(ctx, emotion)`
triggers the emote bubble animation over that actor.

## 4. Friends

| Packet | Parser | `file:line` | Purpose |
|---|---|---|---|
| `ZC_FRIENDS_LIST` | `ParseFriendsList` | `:591-596` | Full friend list sent on login/map-enter. `network/friend_packets.go:53-98` notably tries **two** wire layouts (32-byte named entries vs. 8-byte compact `AccountID`+`CharID` pairs) since different packet-version servers send different shapes; `applyFriendsList(ctx, friends)` replaces the session's friend list. |
| `ZC_FRIENDS_STATE` | `ParseFriendState` | `:597-602` | A friend's online/offline state (and, on longer variants, their name) changed. `applyFriendState(ctx, friendState)`. |
| `ZC_REQ_ADD_FRIENDS` | `ParseFriendRequest` | `:603-608` | Someone sent the local player a friend request. `m.openFriendRequest(ctx, friendRequest)` opens the accept/decline prompt. |
| `ZC_ADD_FRIENDS_LIST` | `ParseFriendAddResult` | `:609-615` | Result of the local player's own outgoing add-friend request (success/already-friends/friend-list-full/etc.). `applyFriendAddResult` updates the list; `m.addFriendResultMessage(friendAdded)` posts a console message. |
| `ZC_DELETE_FRIENDS` | `ParseFriendDelete` | `:616-627` | A friend was removed (by either side). `applyFriendDelete(ctx, friendDeleted)` returns whether a local entry was actually removed, gating a "`<name>` removed from your friend list" system message. |

## 5. Party

Party membership, HP/position sharing, and party chat.

- **Create/invite flow**: `ZC_ACK_MAKE_GROUP` (`ParsePartyCreateResult`,
  `:628-633`) confirms party creation; `ZC_REQ_JOIN_GROUP`
  (`ParsePartyInviteRequest`, `:640-645`) is an incoming invite, opened via
  `m.openPartyInviteRequest`; `ZC_PARTY_JOIN_REQ_ACK`
  (`ParsePartyInviteAnswer`, `:646-651`) is the invitee's accept/decline,
  handled by `m.handlePartyInviteAnswer`.
- **Roster**: `ZC_GROUP_LIST` (`ParsePartyList`, `:634-639`) is the full
  member roster (name, map, role, state) sent on join;
  `ZC_ADD_MEMBER_TO_GROUP`/`_2` (`ParsePartyMemberJoin`, `:843-848`) and
  `ZC_DELETE_MEMBER_FROM_GROUP` (`ParsePartyMemberLeave`, `:849-856`) are
  incremental join/leave updates. `applyPartyMemberLeave` returns whether the
  removal actually applied — if not (e.g. a stale packet for a
  map-local-only restriction), the client posts "Cannot leave party on this
  map."
- **Live tracking**: `ZC_NOTIFY_HP_TO_GROUPM`/`_R2` (`ParsePartyMemberHP`,
  `:857-862`) and `ZC_NOTIFY_POSITION_TO_GROUPM`
  (`ParsePartyMemberPosition`, `:863-868`) stream each member's HP and map
  position for the party UI/minimap.
- **Settings**: `ZC_GROUPINFO_CHANGE` (`ParsePartyOption`, `:652-657`,
  the party's exp-share rule) and `ZC_PARTY_CONFIG`
  (`ParsePartyInviteConfig`, `:658-664`, the local player's "refuse
  invites" toggle — also triggers `m.ui.partySettings.Rebind(ctx)` to
  refresh the settings window).
- **Chat**: `ZC_NOTIFY_CHAT_PARTY` (`ParsePartyChat`, `:869-874`), routed
  into the console via `applyPartyChat`.

## 6. Guild

The largest single packet family (over 25 distinct incoming packets),
reflecting how much guild state RO tracks: roster, positions/permissions,
inter-guild relations, notices, and the guild emblem image.

### 6.1 Membership, identity & info

- **`ZC_UPDATE_GDID`** (`ParseGuildBelonging`, `:672-677`) — sent when the
  local player's own guild affiliation is established (on login or after
  joining/leaving), carrying guild ID, emblem version, and whether the
  player is master. `m.handleGuildBelonging(ctx, guildBelonging)`.
- **`ZC_GUILD_INFO`/`ZC_GUILD_INFO2`** (`ParseGuildInfo`, `:678-684`) — the
  guild's aggregate stats (level, member counts, exp, guild name, master
  name). `applyLocalGuildDetails` stores it; the client then proactively
  calls `m.requestActorGuildEmblem(ctx, guildInfo.GuildID, guildInfo.EmblemVersion)`
  to fetch the emblem image if it doesn't have this version cached yet — see
  §6.5.
- **`ZC_MEMBERMGR_INFO`** (`ParseGuildMembers`, `:685-690`) — the full member
  roster (name, job, level, position, online state). `applyLocalGuildMembers`.
- **`ZC_GUILD_MEMBER_INFO`/`ZC_GUILD_MEMBER_ADDED`** (`ParseGuildMemberInfo`,
  `:714-719`) — a single member added or updated. `applyLocalGuildMember`.
- **`ZC_GUILD_MEMBER_STATE`/`_2`** (`ParseGuildMemberState`, `:698-707`) — a
  member's online/offline state (and, on the `_2` variant, appearance data)
  changed; if the new state is offline (`State == 0`), the client also clears
  that member's minimap marker (`m.ui.minimap.ApplyGuildMemberPosition(accountID, -1, -1)`).
- **`ZC_NOTIFY_POSITION_TO_GUILDM`** (`ParseGuildMemberLocation`, `:708-713`)
  — live position updates for the guild minimap overlay.
- **Leaving/expulsion**: `ZC_ACK_LEAVE_GUILD` (`ParseGuildMemberDeparture`,
  `:745-750`, a member voluntarily left — name + reason), `ZC_ACK_BAN_GUILD`
  (`ParseGuildMemberExpulsion`, `:751-756`, a member was expelled — name +
  reason + expelling account), and `ZC_BAN_LIST`
  (`ParseGuildExpelHistory`, `:763-768`, the historical expulsion log shown
  in the guild window).
- **`ZC_ACK_DISORGANIZE_GUILD_RESULT`** (`ParseGuildDisbandResult`,
  `:757-762`) — result of a guild-disband request. `m.handleGuildDisbandResult`.
- **`ZC_RESULT_MAKE_GUILD`** (`ParseGuildCreationResult`, `:793-798`) —
  result of creating a new guild.

### 6.2 Positions, ranks & skills

- **`ZC_POSITION_INFO`/`ZC_ACK_CHANGE_GUILD_POSITIONINFO`**
  (`ParseGuildPositions`, `:727-732`) — the guild's position (rank)
  definitions: permission bitmask, hierarchy ranking, pay rate, and (on the
  ack variant) the position name inline. `network/guild_packets.go:453-503`
  handles both wire shapes under one parser.
- **`ZC_POSITION_ID_NAME_INFO`** (`ParseGuildPositionNames`, `:733-738`) —
  position-ID-to-name lookup table, kept separate from the numeric position
  data above in some packet shapes.
- **`ZC_ACK_REQ_CHANGE_MEMBERS`** (`ParseGuildMemberPositions`, `:720-726`)
  — confirms a batch reassignment of which position each member holds
  (`applyLocalGuildMemberPositions`, then `m.ui.guildWindow.Refresh(ctx)`).
- **`ZC_GUILD_SKILLINFO`** (`ParseGuildSkillInfo`, `:739-744`) — the guild
  skill tree's current levels and remaining skill points.
  `applyLocalGuildSkills`.

### 6.3 Relations & alliances

- **`ZC_GUILD_ALLIANCE_LIST`** (`ParseGuildRelations`, `:691-697`) — the
  full list of allied/hostile guild relations.
- **Alliance request flow**: `ZC_REQ_ALLY_GUILD` (`ParseGuildAllianceRequest`,
  `:811-816`, incoming alliance proposal, opened via
  `m.openGuildAllianceRequest`) → `ZC_ACK_REQ_ALLY_GUILD`
  (`ParseGuildAllianceResult`, `:817-822`, the outcome, via
  `m.handleGuildAllianceResult`).
- **Hostility**: `ZC_ACK_REQ_HOSTILE_GUILD` (`ParseGuildHostilityResult`,
  `:823-828`) — result of declaring another guild hostile.
- **Relation add/remove**: `ZC_ADD_RELATED_GUILD`
  (`ParseGuildRelationAdded`, `:836-842`) and `ZC_DELETE_RELATED_GUILD`
  (`ParseGuildRelationDeleted`, `:829-835`) — incremental updates to the
  alliance/hostility list, both refreshing `m.ui.guildWindow`.

### 6.4 Chat, notice, menu & invites

- **`ZC_GUILD_CHAT`** (`ParseGuildChat`, `:775-780`) — guild-channel chat
  message, routed to console via `applyGuildChat`.
- **`ZC_GUILD_NOTICE`** (`ParseGuildNotice`, `:769-774`) — the guild's
  bulletin-board subject/body text. `m.handleGuildNotice(ctx, guildNotice)`.
- **`ZC_ACK_GUILD_MENUINTERFACE`** (`ParseGuildMenuAccess`, `:665-671`) — a
  permission bitmask controlling which guild-window tabs/actions the local
  player's position allows; `applyLocalGuildMenuAccess` then
  `m.ui.guildWindow.Refresh(ctx)`.
- **Invite flow**: `ZC_REQ_JOIN_GUILD` (`ParseGuildInviteRequest`,
  `:799-804`, incoming invite, opened via `m.openGuildInviteRequest`) →
  `ZC_ACK_REQ_JOIN_GUILD` (`ParseGuildInviteAck`, `:805-810`, the outcome of
  the local player's own accept/decline, via `m.handleGuildInviteAck`).

### 6.5 Emblem

- **`ZC_GUILD_EMBLEM_IMG`** (`ParseGuildEmblemImage`, `:781-786`) — the raw
  emblem bitmap for a requested guild/version, requested via
  `m.requestActorGuildEmblem` (§6.1) and applied by
  `m.applyGuildEmblemImage(ctx, guildEmblem)`, which decodes and caches it
  for rendering on cloaks/guild windows (see `game/guild_emblem.go`).
- **`ZC_CHANGE_GUILD`** (`ParseGuildEmblemChange`, `:787-792`) — notifies
  that a specific actor's guild/emblem-version changed (e.g. they just
  joined a guild), prompting `m.applyGuildEmblemChange` to refresh that
  actor's displayed emblem, re-fetching the image if the version is new.

## 7. Trade

A full state machine: request → accept/decline → both sides add items →
per-add acknowledgement → both sides conclude → execute, or either side
cancels/undoes.

```
ZC_REQ_EXCHANGE_ITEM(2)      → open request dialog
ZC_ACK_EXCHANGE_ITEM(2)      → target's accept/decline result
ZC_ADD_EXCHANGE_ITEM         → other party added an item to the deal
ZC_ACK_ADD_EXCHANGE_ITEM     → server confirms *our own* item was added
ZC_CONCLUDE_EXCHANGE_ITEM    → one side locked in ("trade" button)
ZC_EXEC_EXCHANGE_ITEM        → both sides locked in; trade executes
ZC_CANCEL_EXCHANGE_ITEM      → either side canceled
ZC_EXCHANGE_ITEM_UNDO        → a lock-in was undone before execution
```

| Packet | Parser | `file:line` | Handler |
|---|---|---|---|
| `ZC_REQ_EXCHANGE_ITEM`/`_2` | `ParseTradeRequest` | `:875-880` | `m.openTradeRequest` |
| `ZC_ACK_EXCHANGE_ITEM`/`_2` | `ParseTradeResponse` | `:918-923` | `m.handleTradeResponse` |
| `ZC_ADD_EXCHANGE_ITEM` | `ParseTradeItem` | `:924-929` | `m.ui.tradeWindow.AddReceivedItem` |
| `ZC_ACK_ADD_EXCHANGE_ITEM` | `ParseTradeAddItemAck` | `:930-935` | `m.ui.tradeWindow.AddOwnItemAck` |
| `ZC_CONCLUDE_EXCHANGE_ITEM` | `ParseTradeConclude` | `:936-941` | `m.ui.tradeWindow.SetConcluded(ctx, tradeConclude.Other)` — `Other` distinguishes whether the *other* party or the local player locked in |
| `ZC_CANCEL_EXCHANGE_ITEM` | `ParseTradeCanceled` (bool-only) | `:942-946` | closes the trade window with a console error message |
| `ZC_EXEC_EXCHANGE_ITEM` | `ParseTradeExec` | `:947-952` | `m.handleTradeExec` |
| `ZC_EXCHANGE_ITEM_UNDO` | `ParseTradeUndo` (bool-only) | `:953-956` | `m.ui.tradeWindow.Undo(ctx)` |

`ParseTradeCanceled`/`ParseTradeUndo` are pure `bool` opcode checks (no
payload to decode) — the simplest possible packets, `network/trade_packets.go:164-170`.

## 8. Adoption

RO's baby/adoption feature: a married couple can propose adopting a
baby-class character.

- **`ZC_REQ_BABY`** (`ParseAdoptionRequest`, `:881-886`) — an incoming
  adoption proposal naming the prospective father/mother account IDs and the
  father's name, opened via `m.openAdoptionRequest(ctx, adoptionRequest)`.
- **`ZC_BABYMSG`** (`ParseAdoptionMessage`, `:887-892`) — a numeric result/
  status code for the adoption flow (e.g. proposal accepted/declined/
  already has a family), surfaced via `m.handleAdoptionMessage`.
- **`ZC_START_BABY`** (`ParseAdoptionStarted`, `:893-895`) — a bare
  boolean-opcode signal (no payload beyond the opcode) that the adoption
  ceremony has begun; the handler is a no-op placeholder
  (`if network.ParseAdoptionStarted(pkt) { return nil, false }`) — worth
  confirming with a maintainer whether client-visible behavior is intended
  here or whether this is only consumed for its side effect of *not*
  falling through to later parsers.

## 9. Vending & shops

Two distinct systems: **vending** (a player sets up a personal store other
players browse) and **NPC shops** (buy/sell dialogs with an NPC).

### Vending

| Packet | Parser | `file:line` | Purpose |
|---|---|---|---|
| `ZC_OPENSTORE` | `ParseVendingOpenRequest` | `:1010-1016` | Server is prompting the local player to configure their own vending store (max item slots); opens `m.ui.vendingWindow.OpenSetup`. |
| `ZC_STORE_ENTRY` | `ParseVendingBoard` | `:1017-1022` | Another player's vending sign appeared on the map (owner + shop name). `m.applyVendingBoard`. |
| `ZC_DISAPPEAR_ENTRY` | `ParseVendingBoardDisappear` | `:1023-1028` | A vending sign closed/disappeared. `m.applyVendingBoardDisappear`. |
| `ZC_PC_PURCHASE_ITEMLIST_FROMMC` | `ParseVendingItemList` (`Own: false`) | `:1029-1038` | Item list from *someone else's* store the player is browsing to buy from; opens `m.ui.vendingWindow.OpenBuy`. |
| `ZC_PC_PURCHASE_MYITEMLIST` | `ParseVendingItemList` (`Own: true`) | `:1029-1038` | The local player's own store's current item list; `m.ui.vendingWindow.ApplyOwnList`. |
| `ZC_PC_PURCHASE_RESULT_FROMMC` | `ParseVendingPurchaseResult` | `:1039-1044` | Result of a buy attempt from someone else's store. |
| `ZC_DELETEITEM_FROM_MCSTORE` | `ParseVendingSoldItem` | `:1045-1050` | One of the local player's own vended items just sold (stock decremented). |

Note the shared `VendingItemList` result type distinguishes "browsing
someone else's shop" from "viewing my own shop's live stock" purely via the
`Own` boolean set by which opcode matched — the two packets have the same
22-byte-per-item wire shape (`network/vending_packets.go:112-140`) but
slightly different per-item field order (`parseVendingItems` vs.
`parseOwnVendingItems`).

### NPC shops

`ZC_SELECT_DEALTYPE` (`ParseShopDealSelection`, `:1062-1067`) — an NPC
offered a buy/sell choice, opening the deal-type prompt. `ZC_PC_SELL_ITEMLIST`
(`ParseShopSellList`, `:1068-1073`) and `ZC_PC_PURCHASE_ITEMLIST`
(`ParseShopBuyList`, `:1074-1079`) open the respective sell/buy dialogs.
`ZC_PC_PURCHASE_RESULT`/`ZC_PC_SELL_RESULT` (`ParseShopResult`, `:1080-1088`)
report the outcome — on a successful sell (`result.Sell && result.Result == 0`),
the client also posts "The deal has successfully completed" to console.

## 10. Inventory & equipment

Covers pickup, item use, and the four equipment "minigame" flows (identify,
repair, refine, card composition) plus generic making/creation.

### Pickup & floor items

- **`ZC_ITEM_ENTRY`/`ZC_ITEM_FALL_ENTRY`** (`ParseFloorItemEntry`,
  `:341-346`) — an item appeared on the ground (either already there, or
  freshly dropped/`Falling: true`). `m.applyFloorItemEntry`.
- **`ZC_ITEM_DISAPPEAR`** (`ParseFloorItemDisappear`, `:347-352`) — a ground
  item was picked up or expired. `m.applyFloorItemDisappear`.
- **`ZC_ITEM_PICKUP_ACK`** (three opcode variants, `ParseItemPickupAck`,
  `:353-367`) — result of a pickup attempt. On success,
  `m.applyItemPickupAck(ctx, pickup)` returns the resolved item, the amount
  actually gained, and success; the client formats a blue pickup console
  message (`formatPickupConsoleMessage`) and shows the floating pickup
  notification (`m.ui.itemPickup.Show`). On failure, a red error message
  names the item and server result code.

### Using items

**`ZC_USE_ITEM_ACK`/`ZC_USE_ITEM_ACK2`** (`ParseUseItemAck`, `:368-385`) —
result of consuming an item. The handler is more involved than most: it
first resolves the depleted item's ID (falling back to a session inventory
lookup by index if the packet's `ItemID` is zero, `:373-377`), triggers any
visual/sound use-effect (`m.addItemUseEffect`), applies the inventory change
(`applyUseItemAck`), and — if the item's amount dropped to zero — clears it
from the hotkey/shortcut bar (`m.ui.shortcutBar.ClearDepletedItem`) so a
depleted potion doesn't leave a stale hotkey slot.

### Identify, repair, refine, composition, making

All four follow the same **list → (choose in UI) → ack** shape; only the
receiving side is documented here (the choice packet is outbound).

| Flow | List packet | Ack packet | List `file:line` | Ack `file:line` |
|---|---|---|---|---|
| Identify | `ZC_ITEMIDENTIFY_LIST` (`ParseItemIdentifyList`) | `ZC_ACK_ITEMIDENTIFY` (`ParseItemIdentifyAck`) | `:428-434` | `:435-445` |
| Card composition | `ZC_ITEMCOMPOSITION_LIST` (`ParseItemCompositionList`) | `ZC_ACK_ITEMCOMPOSITION` (`ParseItemCompositionAck`) | `:498-504` | `:505-513` |
| Repair | `ZC_REPAIRITEMLIST` (`ParseRepairItemList`) | `ZC_ACK_ITEMREPAIR` (`ParseRepairItemAck`) | `:468-474` | `:475-482` |
| Weapon refine | `ZC_NOTIFY_WEAPONITEMLIST` (`ParseWeaponRefineList`) | `ZC_ACK_WEAPONREFINE` (`ParseWeaponRefineAck`) | `:483-489` | `:490-497` |
| Arrow crafting | `ZC_MAKINGARROW_LIST` (`ParseMakingArrowList`) | *(no separate ack; consumed immediately)* | `:446-452` | — |
| Item creation (potions/etc.) | `ZC_MAKABLEITEMLIST` (`ParseMakableItemList`) | `ZC_ACK_REQMAKINGITEM` (`ParseMakingItemAck`) | `:453-459` | `:460-467` |

Every ack in this table ends by calling the relevant window's `ApplyAck` and
then `m.ui.inventoryBag.ClampScroll(ctx.Session)` (or the equivalent for
`cartWindow`/`storageWindow`) — inventory composition changed, so the
scrollable item grid's scroll offset is re-clamped to the new item count.
`MakingItemAck.Success()`/`.Alchemist()` (`network/item_packets.go:163-169`)
decode a shared result code: `0` = plain success, `2` = alchemist-bonus
success, `3` = alchemist-bonus failure, anything else = plain failure.

### Equipping

- **`ZC_REQ_WEAR_EQUIP_ACK`/`ZC_REQ_TAKEOFF_EQUIP_ACK`**
  (`ParseInventoryEquipAck`, `:528-534`) — result of equip/unequip.
  `applyInventoryEquipAck` updates the item's `Equipped` flag; `Unequip`
  distinguishes which action this ack is for.
- **`ZC_EQUIP_ARROW`** (`ParseEquippedArrow`, `:535-542`) — which inventory
  slot is the currently nocked arrow, for ranged classes. `applyEquippedArrow`.
- **`ZC_ADD_ITEM_TO_CART`/`_2`** and the storage equivalent are covered in
  §11 (they're conceptually storage/cart, not equip).

### Full inventory snapshots

**`ZC_NORMAL_ITEMLIST`/`ZC_EQUIPMENT_ITEMLIST`** (and their `2`/`3`/`4`
variants) (`ParseInventoryItemList`, `:514-520`) deliver the complete
inventory on login/map-enter; `network/item_packets.go:471-494` dispatches
across **nine** opcodes to one of four low-level entry-size parsers
(`parseNormalInventoryItems`, `parseNormalInventoryItems4`,
`parseEquipInventoryItems`, `parseEquipInventoryItems4`) depending on which
packet-version shape was negotiated. `applyInventoryItemList(ctx, items)`
replaces the session's inventory wholesale.
**`ZC_ITEM_THROW_ACK`/`ZC_DELETE_ITEM_FROM_BODY`** (`ParseInventoryItemDelete`,
`:521-527`) removes/reduces a specific slot (item dropped, consumed by a
server-side effect, etc.).

## 11. Storage & cart

Structurally parallel to inventory, but for the bank-like "storage" (Kafra)
and the merchant "cart," each with its own open/close, amount-limit, and
item-list/add/remove packets. A **storage password** gate sits in front of
storage access.

### Storage password

**`ZC_REQ_STORE_PASSWORD`** (`ParseStoragePasswordPrompt`, `:543-548`) — the
server is asking the client to present (or set, if `State ==
StoragePasswordNotSet`) a storage password; `m.handleStoragePasswordPrompt`
opens the appropriate prompt UI. **`ZC_RESULT_STORE_PASSWORD`**
(`ParseStoragePasswordResult`, `:549-554`) reports the outcome — success/
failure/too-many-failed-attempts, per the `StoragePasswordResultCode`
constants in `network/storage_password_packets.go:31-39`.

### Snapshots, amounts, open/close

| Concept | Storage packet | Cart packet | `file:line` |
|---|---|---|---|
| Full item list | `ZC_STORE_NORMAL_ITEMLIST`/`_EQUIP` (`ParseStorageItemList`) | `ZC_CART_NORMAL_ITEMLIST`/`_EQUIP` (`ParseCartItemList`) | `:555-563` / `:564-572` |
| Count/weight limits | `ZC_NOTIFY_STOREITEM_COUNTINFO` (`ParseStorageAmount`) | `ZC_NOTIFY_CARTITEM_COUNTINFO` (`ParseCartAmount`) | `:573-581` / `:582-590` |
| Item added | `ZC_ADD_ITEM_TO_STORE`/`_2` (`ParseStorageItemAdded`) | `ZC_ADD_ITEM_TO_CART`/`_2` (`ParseCartItemAdded`) | `:957-967` / `:968-977` |
| Item removed | `ZC_DELETE_ITEM_FROM_STORE` (`ParseStorageItemRemoved`) | `ZC_DELETE_ITEM_FROM_CART` (`ParseCartItemRemoved`) | `:994-1001` / `:1002-1009` |
| Closed | `ZC_CLOSE_STORE` (`ParseStorageClosed`, bool-only) | `ZC_CLOSE_CART` (`ParseCartClosed`, bool-only) | `:1051-1056` / `:1057-1061` |
| Add rejected (cart only) | — | `ZC_ACK_ADDITEM_TO_CART` (`ParseCartAddAck`) | `:978-993` |

Two behaviors worth calling out:

- The storage/cart item-list and item-added handlers all check
  `ctx.Session.Storage.Open` before re-opening the storage window
  (`:559-561`, `:577-579`, `:961-963`) — this is what lets the same
  `ZC_ADD_ITEM_TO_STORE` packet both silently update session state *and*
  refresh the visible window only if it's actually open.
- **`ZC_ACK_ADDITEM_TO_CART`** (`ParseCartAddAck`, `:978-993`) is the only
  packet in this section with distinct failure UX: `Result == 0` posts
  "Cart is overweight," `Result == 1` posts "Cart has too many items,"
  anything else posts a generic "Cart add failed" — each also logged via
  `glog.Warnf`.

## 12. Pets

The pet-taming and pet-management packet set.

Capture is a two-step exchange: **`ZC_START_CAPTURE`**
(`ParsePetCaptureStart`, `:386-391`) signals the capture minigame has begun
(`m.startPetCapture(ctx)`); **`ZC_TRYCAPTURE_MONSTER`**
(`ParsePetCaptureResult`, `:392-397`) reports whether the subsequent
capture attempt succeeded (`m.applyPetCaptureResult`).

Once a pet exists, its live state streams through:

| Packet | Parser | `file:line` | Purpose |
|---|---|---|---|
| `ZC_PROPERTY_PET` | `ParsePetProperty` | `:398-403` | Full pet stat snapshot (name, level, fullness/hunger, intimacy, equipped accessory). |
| `ZC_FEED_PET` | `ParsePetFeedResult` | `:404-409` | Result of feeding the pet an item. |
| `ZC_CHANGESTATE_PET` | `ParsePetStateChange` | `:410-415` | A discrete pet state transition (hatched, evolved, etc. — `Type`/`Data` are opaque codes interpreted by `m.applyPetStateChange`). |
| `ZC_PET_ACT` | `ParsePetAction` | `:422-427` | The pet performed a "performance"/trick action. |
| `ZC_PETEGG_LIST` | `ParsePetEggList` | `:416-421` | The list of pet-egg inventory indexes available to hatch, opened for selection via `m.applyPetEggList`. |

## 13. Homunculus & mercenary

Two similar "player-summoned companion" systems (Alchemist homunculi and
mercenary contracts), each with a property snapshot, param deltas, and a
skill list/update pair — deliberately mirrored packet shapes, per
`network/companion_packets.go`.

- **Property**: `ZC_PROPERTY_HOMUN` (five opcode variants spanning older and
  newer clients, `ParseHomunculusProperty`, `game/world_packets.go:1169-1174`)
  and `ZC_MER_INIT`/`ZC_MER_PROPERTY`/`ZC_PROPERTY_MERCE`
  (`ParseMercenaryProperty`, `:1206-1211`) each deliver a full stat block
  (name, level, HP/SP, combat stats). The homunculus parser in particular
  (`network/companion_packets.go:131-219`) dynamically adjusts field widths
  (16- vs. 32-bit HP/SP, 32- vs. 64-bit exp) per opcode via a small
  `homunculusPropertyLayout` lookup table — a good example of one Go struct
  absorbing several wire-format generations.
- **Param deltas**: `ZC_HO_PAR_CHANGE`/`_2` (`ParseHomunculusParamChange`,
  `:1188-1193`) and `ZC_MER_PAR_CHANGE` (`ParseMercenaryParamChange`,
  `:1212-1217`) are incremental single-stat updates (param ID + new value)
  rather than full snapshots — cheaper for frequent HP/SP ticks.
- **Feed/state**: `ZC_FEED_MER` (`ParseHomunculusFeedResult`, `:1175-1181`,
  despite the "MER" in the opcode name this is the homunculus feed result)
  and `ZC_CHANGESTATE_MER` (`ParseHomunculusStateChange`, `:1182-1187`).
- **Skills**: `ZC_HO_SKILLINFO_LIST`/`ZC_MER_SKILLINFO_LIST`
  (`ParseHomunculusSkillInfoList`/`ParseMercenarySkillInfoList`,
  `:1194-1199`/`:1218-1223`) and their per-skill update counterparts
  (`:1200-1205`/`:1224-1229`) share the same 37-byte `SkillInfo` entry
  format as the player's own skill list (§16) via the shared
  `parseCompanionSkillList`/`parseSkillInfoEntry` helpers.

## 14. Actor state & appearance

Everything about how another actor (or the local player, as seen by others)
looks and behaves in the world, excluding movement (§2) and HP/status
(§15).

| Packet | Parser | `file:line` | Meaning |
|---|---|---|---|
| `ZC_NOTIFY_VANISH` | `ParseActorVanish` | `:1089-1103` | An actor left visibility (walked off-screen, died, teleported, etc. — `Reason` distinguishes these). `m.applyActorVanish` removes it from `world.Actors`; the handler additionally clears any pending attack/lock/focus state (`m.pendingAttack`, `m.lockedAttackID`, `m.attackFocusID`) that referenced the vanished actor's ID, so the client doesn't keep trying to act on a gone target. |
| `ZC_RESURRECTION` | `ParseActorResurrection` | `:1104-1109` | An actor was revived. `m.applyActorResurrection`. |
| `ZC_CHANGE_LOOK`/`ZC_CHANGE_LOOK2` | `ParseActorLookChange` | `:1110-1120` | A single appearance attribute changed (weapon, headgear slot, hair color, etc. — `Type` selects which). The handler first tries `m.applySkillUnitLookChange` (some ground-skill-unit visuals reuse this same packet); if that doesn't claim it, `applyActorLookChange` applies it to a regular actor and, if it affects the *local player's* rendered sprite, `m.reloadPlayerSpriteView(ctx, ...)` rebuilds the composited billboard (see GRAPHICS.md §6 for what that rebuild does). |
| `ZC_CHANGE_DIRECTION` | `ParseActorDirectionChange` | `:1121-1126` | An actor's facing direction (and head direction) changed without moving. `applyActorDirectionChange`. |
| `ZC_STATE_CHANGE`/`ZC_STATE_CHANGE3` | `ParseActorStateChange` | `:1127-1132` | Body/health/effect state bitmasks changed (e.g. sitting, hidden, cursed) — distinct from the timed status-effect packets in §15. `m.applyActorStateChange`. |
| `ZC_BLADESTOP` | `ParseActorBladeStop` | `:1133-1138` | The Blade Stop skill's lock between two actors activated/deactivated. `m.applyActorBladeStop`. |
| `ZC_NOTIFY_ACT`/`_2`/`_3`, `ZC_NOTIFY_SKILL`/`_2` | `ParseActorActionNotify` | `:1139-1144` | A combat action occurred: a melee hit (with damage/hit-count/motion timing) or a skill-driven hit, depending on opcode. `m.applyActorActionNotify` drives damage-number display and hit animation timing (source/target attack-motion speed is included so the client can time the swing animation against the server's authoritative timing). |
| `ZC_NOTIFY_MONSTER_HP`/`ZC_HP_INFO_TINY` | `ParseActorHPUpdate` | `:1145-1150` | HP/MaxHP update for a non-player actor (typically a monster health bar); the "tiny" variant is a compressed single-byte HP encoding (`HP = byte*5`, `MaxHP` fixed at 100) used for bandwidth-sensitive high-actor-count situations. `m.applyActorHPUpdate`. |

## 15. Status effects & stats

Player character stats (STR/AGI/etc.), derived combat stats, and timed
status-effect icons.

- **`ZC_PAR_CHANGE`/`ZC_STATUS_CHANGE`** (`ParseParameterChange`,
  `:1340-1348`) — the single most common "your character's stat X changed
  to value Y" packet (HP, SP, base/job level, zeny, weight, stat points, and
  more — `VarID` selects which, matched against the `StatusXxx` constants in
  `network/status_packets.go:12-42`). `m.applyParameterChange` writes it into
  `session.Session`; if the changed var is HP (`network.StatusHP`), the
  handler additionally calls `m.clearLocalDeathStateIfAlive(ctx)` — HP going
  positive again is how the client detects a revive when no explicit
  resurrection packet arrives for the local player.
- **`ZC_NOTIFY_EXP`** (`ParseExpNotify`, `:133-138`) — an exp gain event
  (amount, base or job exp, and a "reason" `ExpType` such as
  kill/quest/script). `addExpNotifyMessage(&m.ui.console, ctx.Resources, notify)`
  formats a console message using the resource manager's string tables.
- **`ZC_STATUS`** (`ParseStatusSnapshot`, `:1151-1156`) — the full
  stats-window snapshot: base stats, their upgrade cost, and every derived
  combat stat (atk/matk/def/mdef/hit/flee/crit/aspd, each with base+bonus
  split). `applyStatusSnapshot`.
- **`ZC_STATUS_CHANGE_ACK`** (`ParseStatusChangeAck`, `:1157-1162`) —
  specifically the result of the local player spending a stat point
  (STR/AGI/etc. up), routed straight to `m.ui.statsWindow.ApplyStatusChangeAck`
  rather than through session state generically.
- **`ZC_RECOVERY`/`ZC_RECOVERY2`** (`ParseRecovery`, `:1334-1339`) — an
  instantaneous HP/SP/other-stat recovery event (potion, skill heal), kept
  separate from the generic parameter-change packet, presumably so the
  client can play a distinct recovery visual/sound. `m.applyRecovery`.
- **`ZC_MSG_STATE_CHANGE`/`_2`/`_3`** (`ParseStatusEffectChange`,
  `:1163-1168`) — a timed status effect (buff/debuff icon) activated or
  deactivated, with the newer variants adding a duration and up to three
  opaque effect-specific values (used for e.g. showing remaining time or a
  stack count on the icon). `m.applyStatusEffectChange`.

## 16. Skills

Skill list/tree management, casting, ground-placed skill units, and related
notifications. See also §13 for the homunculus/mercenary skill lists, which
reuse the same wire format.

- **`ZC_SKILLINFO_LIST`** (`ParseSkillInfoList`, `:1230-1236`) — the
  player's full learned-skill list, sent on login/level-up. Each entry is a
  fixed 37-byte record (`skillInfoEntryLen`, `network/skill_packets.go:13`)
  decoded by the shared `parseSkillInfoEntry` helper — ID, type bitmask,
  level, SP cost, range, name (Korean-EUC-KR-decoded via
  `decodeROFixedString`), and whether it's upgradable. `applySkillInfoList`
  then `m.ui.skillWindow.ClampScroll(ctx.Session)`.
- **`ZC_SKILLINFO_UPDATE`/`ZC_ADD_SKILL`** (`ParseSkillInfoUpdate`,
  `:1237-1243`) — a single skill was added or its level/cost/range changed
  (e.g. after allocating a skill point).
- **`ZC_AUTORUN_SKILL`** (`ParseAutoRunSkill`, `:1244-1249`) — server-driven
  "auto cast this skill now" (used by some scripted sequences).
  `m.skills().ApplyAutoRun(ctx, auto)`.
- **`ZC_MONSTER_INFO`** (`ParseMonsterInfo`, `:1250-1255`) — a monster's info
  card (class, level, size, HP, def/mdef, race, and per-element damage-rate
  table) — typically requested by right-clicking/inspecting a monster.
  `m.applyMonsterInfo(ctx, info, now)`.
- **`ZC_AUTOSPELLLIST`** (`ParseAutoSpellList`, `:1256-1261`) — the choice
  list for a skill that lets the player pick a sub-spell (e.g. Sage's
  Autospell); up to 7 candidate skill IDs. `m.applyAutoSpellList`.
- **`ZC_WARPLIST`/`ZC_WARPLIST2`** (`ParseWarpPointList`, `:1262-1267`) —
  the list of remembered warp-point map names for a Warp Portal-family
  skill, opened for selection. `m.applyWarpPointList`.
- **`ZC_ACK_REMEMBER_WARPPOINT`** (`ParseRememberWarpPointAck`,
  `:1268-1273`) — result of the player saving the current map as a new warp
  point. `m.applyRememberWarpPointAck`.
- **`ZC_ACK_TOUSESKILL`** (`ParseSkillFailAck`, `:1274-1279`) — a skill-use
  attempt failed (insufficient SP, wrong weapon, on cooldown, etc.), with a
  `Cause` code the UI maps to an error message. `m.applySkillFailAck`.
- **`ZC_USESKILL_ACK`** (`ParseSkillCastNotify`, `:1280-1285`) — a skill cast
  began (source, target or ground position, skill ID, and cast-time delay),
  driving the cast bar/animation. `m.applySkillCastNotify`.
- **`ZC_SKILL_ENTRY`** — note this opcode name is reused for **two**
  different parsers at different offsets: `ParseGroundSkillNotify`
  (`:1286-1291`, a skill announcement with a level field) and
  `ParseSkillUnitEntry` (`:1292-1297`, a persistent ground-unit spawn like a
  trap or firewall tile, with a `Visible` flag). Both are tried in sequence
  in the dispatch chain; `game/world_packets.go`'s comment-free ordering
  means whichever `ParseXxx` is listed first "wins" if both ever matched the
  same bytes — worth double-checking against `network/skill_packets.go` if
  you touch either.
- **`ZC_SKILL_DISAPPEAR`** (`ParseSkillUnitDisappear`, `:1298-1303`) and
  **`ZC_SKILL_UPDATE`** (`ParseSkillUnitUpdate`, `:1304-1309`) — a ground
  skill unit (trap, hazard tile) expired or changed state.
- **`ZC_NOTIFY_EFFECT`/`_2`** (`ParseSpecialEffectNotify`,
  `:1310-1315`) — a canned visual/sound effect keyed by a small integer ID
  (level-up flash, refine success/failure, game-over, etc. — see the
  `SpecialEffectXxx` constants in `network/effect_packets.go:16-25`).
  `m.applySpecialEffectNotify`.
- **`ZC_MVP`** (`ParseMVPNotify`, `:1316-1321`) — announces the local player
  earned MVP credit for a kill. `m.applyMVPNotify`.
- **`ZC_USE_SKILL`/`ZC_USE_SKILL2`** (`ParseSkillNoDamageNotify`,
  `:1322-1327`) — a non-damaging skill (buff/heal/debuff) resolved against a
  target. `m.applySkillNoDamageNotify`.

## 17. Monster info

Covered under §16 (`ZC_MONSTER_INFO`, `ParseMonsterInfo`) — grouped there
since it's parsed in `network/skill_packets.go` alongside the skill types,
even though conceptually it's a monster-inspection packet rather than a
skill one.

## 18. Warp points

Covered under §16 (`ZC_WARPLIST`/`ZC_WARPLIST2` and
`ZC_ACK_REMEMBER_WARPPOINT`) for the same reason — both are defined in
`network/skill_packets.go` because warp points are granted via a skill
(Warp Portal).

## 19. NPC dialog & cutin

- **NPC dialog** (`ParseNPCDialog`, `game/world_packets.go:278-286`,
  `network/npc_packets.go:58-108`) is one parser covering seven dialog-box
  sub-kinds via `NPCDialogKind`: `ZC_SAY_DIALOG` (a line of text),
  `ZC_WAIT_DIALOG` ("next" prompt), `ZC_CLOSE_DIALOG`, `ZC_MENU_LIST` (a
  `:`-delimited choice list, split by `splitNPCMenuOptions`),
  `ZC_CLEAR_DIALOG`, `ZC_OPEN_EDITDLG` (numeric input box), and
  `ZC_OPEN_EDITDLGSTR` (text input box). `m.ui.npcDialog.Apply(dialog)`
  drives the dialog window; if the dialog is closing/clearing *and* the
  dialog window reports itself no longer open, the handler also clears any
  active NPC cut-in image (`m.ui.npcCutin.Clear()`) — cut-ins are shown
  alongside dialog and should disappear with it.
- **`ZC_SHOW_IMAGE2`** (`ParseNPCCutin`, `:287-294`) — an NPC cut-in
  portrait/scene image (identified by filename) at a screen position
  (`NPCCutinLeft`/`Center`/`Right`/`Window`/`Windowless`, or `255` to
  clear). `m.applyNPCCutin` loads and displays it; a missing/unavailable
  image logs a warning rather than erroring the whole packet handler.

## 20. Minimap

**`ZC_COMPASS`** (`ParseMinimapCompass`, `:295-300`) — places or updates a
minimap marker (an NPC/event/quest compass point), keyed by an `ID` byte so
the same marker can be moved/removed later. `m.ui.minimap.ApplyCompass(...)`.
(Guild-member minimap markers are a separate packet, §6.1's
`ZC_NOTIFY_POSITION_TO_GUILDM`.)

## 21. Misc / UI preference

Small standalone preference-sync packets that don't belong to a larger
family:

- **`ZC_CONFIG_NOTIFY`** (`ParseShowEquipConfig`, `:896-903`) — the
  server-side "show my equipment to others" preference state; written
  straight into `ctx.Session.ShowEquip` with no further processing.
- **`ZC_LESSEFFECT`** (`ParseLessEffect`, `:904-911`) — the server-side
  "reduce visual effects" preference; written into `ctx.Session.LessEffects`.
  Note the client also *sends* this same logical setting right after
  `ZC_ACCEPT_ENTER` (§1, `sendLessEffectPreference(ctx)`) since servers reset
  it per map-enter — this incoming packet is the server's echo/confirmation.
- **`ZC_EQUIPWIN_MICROSCOPE`** (`ParseViewedEquipment`, `:912-917`) — the
  full equipment loadout of another player the local player inspected
  (right-click "View Equipment"). `m.ui.viewEquipWindow.Open(ctx, viewedEquip, m)`.
- **`HC_NOTIFY_ZONESVR`**/similar login-adjacent packets are *not* in this
  chain — see `game/login.go` instead; `ZC_ACCEPT_ENTER` (§1) is the first
  world-server packet this chain actually processes.

## 22. Actor spawn (`ActorEntry`)

**`ParseActorEntry`** (`game/world_packets.go:1349-1356`,
`network/actor_packets.go:163-478`) is both the most frequently-triggered
packet in this entire chain (every actor that enters view sends one) and the
one with by far the most wire-format variants — **twelve** opcodes
(`ZC_NOTIFY_STANDENTRY` `0x0078`, `ZC_NOTIFY_ENTRY` `0x0079`/`0x007A`,
`ZC_NOTIFY_MOVEENTRY` `0x007B`, `ZC_NOTIFY_STANDENTRY_NPC` `0x007C`,
`ZC_NOTIFY_MOVE` `0x0086`, `ZC_NOTIFY_STANDENTRY2` `0x01D8`,
`ZC_NOTIFY_NEWENTRY2` `0x01D9`, `ZC_NOTIFY_MOVEENTRY2` `0x01DA`,
`ZC_NOTIFY_NEWENTRY3` `0x02ED`, `ZC_NOTIFY_STANDENTRY3` `0x02EE`, plus
"modern" (`0x022C`/`0x02EC`) and "variable, with/without robe slot"
(`0x07F7`/`0x0856`) shapes handled by the helper functions
`parseActorMoveEntryModern`/`parseActorMoveEntryVariable`) — reflecting how
many times RO's client added new visible-equipment slots (headgear layers,
robes/garments, guild emblem, character level) to this one packet across the
game's history.

Regardless of which opcode arrives, the handler is uniform
(`game/world_packets.go:1349-1356`):

1. `m.clearActorDeath(entry.ID)` and `m.clearActorVanish(entry.ID)` — an
   actor re-entering view means any stale death/vanish bookkeeping for that
   ID is now wrong and must be cleared first.
2. `m.upsertNetworkActor(ctx, entry)` — inserts a new `world.Actor` or
   updates an existing one in place (same ID re-entering view, e.g. after a
   temporary out-of-range period).
3. `m.applyWarpPortalEntry(ctx, entry)` — a separate check specifically for
   warp-portal-type NPCs, since those need extra click-target/visual
   handling beyond a normal actor.

Unlike almost every other packet in this document, this arm does **not**
`return nil, false` explicitly at its end — it falls through to the
function's final `return nil, false` at `game/world_packets.go:1358`, since
it's the last arm in the chain. Functionally identical, but worth knowing if
you're adding a new arm after this one (don't assume yours will run — check
whether `ActorEntry`'s fallthrough behavior needs preserving, or move your
new arm earlier in the chain).

---

*For how to add a new packet arm to this chain, see
`development-docs/ARCHITECTURE.md`'s "How to..." section
("Add a new incoming packet type").*
