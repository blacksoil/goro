# Outgoing Packet Details — `network.Client.SendXxx`

This document catalogs every outgoing (client→server) packet `goro` can
send — every `func (c *Client) SendXxx(...) error` method across the
`network` package (136 of them, plus the low-level `Client.Send([]byte)`
primitive every one of them ultimately calls). It is the outbound
counterpart to [`IN-PACKET-DETAILS.MD`](IN-PACKET-DETAILS.MD), which
catalogs the server→client side of `WorldMode.handleNetworkPacket`.

Unlike that document — which was scoped to just the in-game dispatch
chain — **this one covers the client's entire outbound API surface**,
including packets sent during login/character-select (`game/login.go`),
which `IN-PACKET-DETAILS.MD` does not touch at all. Each section below
notes whether its packets are login-time, in-game, or (rarely) both.

See `development-docs/ARCHITECTURE.md` ("Networking") for the general
machinery: `network.Client` owns one TCP connection, a `sendCh` outbound
queue (size 256), and a `writeLoop` goroutine that drains it. Every
`SendXxx` method builds wire bytes via a matching `BuildXxxPacket` helper
in the same `network/*_packets.go` file, then calls `c.Send(packet)` (or
the lower-level `c.enqueue` directly, for the one packet that also needs
to start the map keepalive — see §2). `c.Send` just wraps `c.enqueue(data,
nil)` — this document does not re-list it as a packet.

**A useful shortcut for opcode names**: nearly every `SendXxx` method logs
`glog.Debugf("sent CZ_XXX opcode=0x%04X ...")` (or `CA_XXX` for the
login-server packets) on success — the opcode names quoted throughout this
document come directly from those log lines, which is also why running
with `--net-trace` or debug logging is a reliable way to watch exactly
which packets a given UI action produces.

---

## 1. Login & character-select handshake

*Login-time only.* Unlike every other section, this one is best understood
as a single ordered sequence rather than an unordered packet list — it's
the first thing that happens and nothing in `IN-PACKET-DETAILS.MD` covers
it. All of it happens in `game/login.go`.

<pre>
1. Connect TCP to the login server address from clientinfo.xml
2. CA_LOGIN         SendAccountLogin(username, password, version, clientType)
   ── server replies with a char-server list (handled elsewhere, not in this doc)
3. Connect TCP to the chosen char server
4. CA_ENTER         SendCharServerEnter(accountID, authCode, userLevel, sex)
   ── server replies with the character list
5a. CH_SELECT_CHAR  SendSelectCharacter(slot)      — picking an existing character
5b. CH_MAKE_CHAR    SendMakeCharacter(character)   — creating a new one
5c. CH_DELETE_CHAR  SendDeleteCharacter(charID, key) — deleting one
   ── server replies with a zone/map-server address (ZoneServerNotify)
6. Connect TCP to the map server
7. CZ_ENTER2        SendMapServerEnter(accountID, charID, authCode, clientTick, sex)
   ── this call also starts the map keepalive goroutine, see §2
8. CZ_NOTIFY_ACTORINIT  SendLoadEndAck() — sent once the client finishes
   loading the map scene, not part of the handshake itself; see §2
</pre>

| Step | Packet | Method | `file:line` | Notes |
|---|---|---|---|---|
| Account login | `CA_LOGIN` | `SendAccountLogin` | `client.go:149` | `clientType` comes from `loginClientType(conn.LangType)`; called from `game/login.go:1019` right after `ctx.Network.Connect` succeeds. |
| Char-server enter | `CA_ENTER` | `SendCharServerEnter` | `client.go:159` | Called from `LoginMode.connectCharServer` (`game/login.go:1129`). |
| Login-server keepalive | — | `SendLoginServerKeepalive` | `client.go:169` | Periodic ping (`loginServerPingInterval`) while sitting at the account/character-service screen, driven by `LoginMode.maybeSendLoginServerPing` (`game/login.go:1057`). Not itself a distinct opcode name in the debug log (no `glog.Debugf` on success). |
| Char selection | `CH_SELECT_CHAR` | `SendSelectCharacter` | `client.go:173` | `game/login.go:820`, on confirming character selection. |
| Char creation | `CH_MAKE_CHAR` | `SendMakeCharacter` | `client.go:177` | `game/login_character_create_window.go:118`. Logs full requested stat block (`Str`/`Agi`/.../`Luk`, hair color/style) on send. |
| Char deletion | `CH_DELETE_CHAR` | `SendDeleteCharacter` | `client.go:190` | `game/login_character_select_window.go:170`. `key` is the deletion confirmation the account's email/birthdate flow produces. |
| Char-server keepalive | `PING` | `SendPing` | `client.go:228` | `LoginMode.maybeSendCharServerPing` (`game/login.go:1103`), sent every `charServerPingInterval` while on the character list/creation screen. |
| Map-server enter | `CZ_ENTER2` | `SendMapServerEnter` | `client.go:254` | Called twice: once during the initial handshake (`game/login.go:1150`) and again from `game/world.go:1439` when a warp moves the player to a *different* map server (`change.ServerMove`) — see §2. Uses `c.enqueue` directly (not `c.Send`) so it can capture the resulting `net.Conn` and pass it to `c.startMapKeepalive(conn)`. |

## 2. Map lifecycle, movement & combat actions

*In-game, plus one login-adjacent packet.*

- **`CZ_NOTIFY_ACTORINIT`** (`SendLoadEndAck`, `client.go:202`) — tells the
  server the client has finished loading the current map scene and is
  ready to start receiving actor/item updates. Sent from three places in
  `game/world.go`: once when entering a map headlessly (`:504`, bot/CI
  runs skip scene loading entirely), once after the full scene load
  completes normally (`:562`), and once when a warp reuses an
  already-loaded map (`reuseLoadedMap`, `:1420`) rather than reloading
  everything from scratch. Corresponds to no single documented inbound ack
  in `IN-PACKET-DETAILS.MD` — the server's response is simply the stream of
  `ActorEntry`/`FloorItemEntry`/etc. packets starting to arrive.
- **`CZ_REQUEST_MOVE`** (`SendWalkToXY`, `client.go:273`) — requests
  walking to a tile. `WorldMode.requestWalk` (`game/movement.go:153`)
  builds this from a mouse click or bot command, after checking the player
  isn't dead, isn't hidden (unless they know Tunnel Drive), and the target
  is in map bounds; a cooldown (`walkRequestCooldown`) throttles repeat
  requests. See `IN-PACKET-DETAILS.MD` §2 for the corresponding
  `ZC_NOTIFY_PLAYERMOVE` ack that reconciles the client's local prediction.
- **`CZ_CHANGE_DIRECTION`** (`SendChangeDirection`, `client.go:287`) — a
  "turn in place" request, used when the target cell is adjacent and the
  player just needs to face it rather than walk (`game/movement.go:236`).
- **`CZ_REQUEST_ACT`** (`SendActionRequest`, `client.go:298`) — a single
  opcode covering both combat and sit/stand, disambiguated by the `action`
  byte (`network/login_packets.go:71-73`, despite the filename these
  constants are used in-game): `ActionAttack = 7`
  (`WorldMode.sendAttackAction`, `game/battle.go:483`, triggered by
  clicking a hostile target), `ActionSitDown = 2` /
  `ActionStandUp = 3` (`client.SetSitting`, `client/chat.go:57`, triggered
  by the `/sit`/`/stand` console commands or their keybindings).
- **`CZ_RESTART`** (`SendRestart`, `client.go:206`) — "return to character
  select" (after death) or a full respawn-in-place, disambiguated by
  `restartType`; sent from `ui/escape_menu.go:136,159`. See
  `IN-PACKET-DETAILS.MD` §1 for the `ZC_RESTART_ACK` response.
- **`CZ_REQ_DISCONNECT`** (`SendQuitGame`, `client.go:217`) — "quit to
  desktop," sent from `ui/escape_menu.go:181`.

## 3. NPC dialog interaction

*In-game.* All defined in `network/client.go`, all driven by
`ui/npc_dialog.go` in response to the dialog window's Next/Close/menu/input
controls, and all mirror the seven-variant `ZC_SAY_DIALOG` family
documented in `IN-PACKET-DETAILS.MD` §19.

| Packet | Method | `file:line` | Trigger |
|---|---|---|---|
| `CZ_CONTACTNPC` | `SendNPCContact` | `client.go:309` | Clicking an NPC to start a conversation (`game/world.go:1552`). |
| `CZ_REQ_NEXT_SCRIPT` | `SendNPCNext` | `client.go:320` | Clicking "next"/anywhere to advance a `ZC_WAIT_DIALOG` line (`ui/npc_dialog.go:253`). |
| `CZ_CLOSE_DIALOG` | `SendNPCClose` | `client.go:331` | Closing the dialog window (`ui/npc_dialog.go:322`). |
| `CZ_CHOOSE_MENU` | `SendNPCMenuChoice` | `client.go:342` | Picking an option from a `ZC_MENU_LIST` (`ui/npc_dialog.go:347`). |
| `CZ_INPUT_EDITDLG` | `SendNPCNumberInput` | `client.go:353` | Submitting a numeric-input dialog (`ui/npc_dialog.go:295`). |
| `CZ_INPUT_EDITDLGSTR` | `SendNPCStringInput` | `client.go:364` | Submitting a text-input dialog (`ui/npc_dialog.go:297`). |

## 4. Chat & social messaging

*In-game.*

### The chat dispatcher

Most console-typed chat does not call a `network` method directly — it
goes through **`client.SendChat(ctx, input)`** (`client/chat.go:12`,
called from `ui/console.go:388`), a small router that inspects the input
text before deciding which packet to send:

<pre>
input starts with "%"        → SendPartyMessage (strip the "%")
input starts with "$"        → SendGuildMessage (strip the "$")
"/sit" / "/stand"             → client.SetSitting → SendActionRequest (§2)
"/pvpinfo"                    → SendPvPInfoRequest
"/w NAME MESSAGE" / "/whisper" → SendWhisper
anything else                 → SendGlobalChat
</pre>

This is why `SendPartyMessage` (`network/party_packets.go:447`) and
`SendGuildMessage` (`network/guild_packets.go:1088`) have no direct
`.SendPartyMessage(`/`.SendGuildMessage(` call sites anywhere in `game`/`ui`
— they're passed as `func(string) error` values into `client.sendGroupChat`
(`client/chat.go:108`) rather than called by name, which is worth knowing
if you ever grep for callers and come up empty.

### Direct sends

| Packet | Method | `file:line` | Trigger |
|---|---|---|---|
| `CZ_REQUEST_CHAT` | `SendGlobalChat` | `chat_packets.go:517` | Default case of `client.SendChat`; also `game/song_talk.go:33` for the Bard/Dancer song-request flow. |
| `CZ_WHISPER` | `SendWhisper` | `chat_packets.go:531` | `/w`/`/whisper` console command, or the whisper window's send box (`game/whisper_window.go:37`). |
| `CZ_SETTING_WHISPER_PC` | `SendWhisperIgnore` | `chat_packets.go:545` | Per-name whisper block/unblock (`game/world.go:1095`, `ui/console.go:785`). |
| `CZ_SETTING_WHISPER_STATE` | `SendWhisperIgnoreAll` | `chat_packets.go:559` | Global whisper block toggle (`ui/console.go:796`). |
| `CZ_CREATE_CHATROOM` | `SendCreateChatRoom` | `chat_packets.go:570` | Creating a chat room (`game/chat_room.go:46`). |
| `CZ_EXIT_ROOM` | `SendExitChatRoom` | `chat_packets.go:584` | Leaving a chat room (`game/chat_room.go:146`). |
| `CZ_REQ_ENTER_ROOM` | `SendEnterChatRoom` | `chat_packets.go:595` | Clicking a chat-room sign on the map (`game/chat_room.go:186`). |
| `CZ_REQ_EMOTION` | `SendEmotion` | `emotion_packets.go:39` | Picking an emote from the emote window (`ui/emote_window.go:212`) or `/e`-style console shortcuts (`ui/console.go:739`). |
| `CZ_DORIDORI` | `SendDoriDori` | `novice_packets.go:13` | The `/doridori` console command (`ui/console.go:850`), a novice-class head-shake that builds fame points over time. |

The chat-room lifecycle itself (create → board entry → enter with roster →
join/leave → change → role change) is documented from the *receiving*
side in `IN-PACKET-DETAILS.MD` §3; the sends above are what initiate steps
1, 3, and the leave step of that flow.

## 5. Friends

*In-game.*

| Packet | Method | `file:line` | Trigger |
|---|---|---|---|
| `CZ_ADD_FRIENDS` | `SendAddFriend` | `friend_packets.go:216` | Adding a friend by name (`game/friends.go:91`). |
| `CZ_DELETE_FRIENDS` | `SendDeleteFriend` | `friend_packets.go:227` | Removing a friend (`game/friends.go:106`). |
| `CZ_ACK_REQ_ADD_FRIENDS` | `SendFriendRequestAck` | `friend_packets.go:238` | Accepting/declining an incoming request (`game/friends.go:27,35`) — the outbound half of the `ZC_REQ_ADD_FRIENDS` → `ZC_ADD_FRIENDS_LIST` flow in `IN-PACKET-DETAILS.MD` §4. |

## 6. Party

*In-game.*

A create/invite/roster flow, structurally mirroring `IN-PACKET-DETAILS.MD`
§5:

<pre>
CZ_MAKE_GROUP / CZ_MAKE_GROUP2   → create a party
CZ_PARTY_JOIN_REQ                → invite a player by account ID
CZ_PARTY_JOIN_REQ_ACK            → invitee accepts/declines
CZ_REQ_EXPEL_GROUP_MEMBER        → kick a member
CZ_REQ_LEAVE_GROUP               → leave voluntarily
</pre>

| Packet | Method | `file:line` | Trigger |
|---|---|---|---|
| `CZ_MAKE_GROUP` | `SendMakeParty` | `party_packets.go:359` | `ui/console.go:591`, the `/party` console command's simple form. |
| `CZ_MAKE_GROUP2` | `SendMakeParty2` | `party_packets.go:370` | `game/party.go:63`, the party creation window (also sets item-pickup/division rules at creation time). |
| `CZ_PARTY_JOIN_REQ` | `SendPartyInvite` | `party_packets.go:381` | Inviting a specific player (`game/party.go:25`, `ui/console.go:697`). |
| `CZ_PARTY_JOIN_REQ_ACK` | `SendPartyInviteAck` | `party_packets.go:392` | Accepting/declining an incoming invite (`game/party.go:111,119`). |
| `CZ_REQ_LEAVE_GROUP` | `SendLeaveParty` | `party_packets.go:403` | Leaving the party (`game/world.go:1081`, `ui/console.go:672`). |
| `CZ_CHANGE_GROUPEXPOPTION` | `SendPartyOption` | `party_packets.go:414` | Toggling the party's exp-share rule (`ui/party_settings_window.go:100`). |
| `CZ_PARTY_CONFIG` | `SendPartyInviteConfig` | `party_packets.go:425` | Toggling "refuse invites" (`ui/party_settings_window.go:103`, `ui/console.go:712`). |
| `CZ_REQ_EXPEL_GROUP_MEMBER` | `SendExpelPartyMember` | `party_packets.go:436` | Kicking a party member (`game/party.go:95`). |
| `CZ_REQUEST_CHAT_PARTY` | `SendPartyMessage` | `party_packets.go:447` | The `%message` console prefix, via `client.SendChat` (see §4) — **not** called directly anywhere in `game`/`ui`. |

## 7. Guild

*In-game.* The largest section, mirroring `IN-PACKET-DETAILS.MD` §6's five
subdivisions.

### 7.1 Membership & lifecycle

| Packet | Method | `file:line` | Trigger |
|---|---|---|---|
| `CZ_REQ_MAKE_GUILD` | `SendCreateGuild` | `guild_packets.go:912` | Guild creation (`ui/console.go:615`). |
| `CZ_REQ_JOIN_GUILD` | `SendGuildInvite` | `guild_packets.go:923` | Inviting a player (`game/guild.go:157`). |
| `CZ_JOIN_GUILD` | `SendGuildInviteReply` | `guild_packets.go:934` | Accepting/declining an incoming invite (`game/guild.go:172,181`) — outbound half of the `ZC_REQ_JOIN_GUILD`/`ZC_ACK_REQ_JOIN_GUILD` flow in `IN-PACKET-DETAILS.MD` §6.4. |
| `CZ_REQ_LEAVE_GUILD` | `SendLeaveGuild` | `guild_packets.go:945` | Leaving voluntarily (`game/guild_lifecycle.go:75`). |
| `CZ_REQ_BAN_GUILD` | `SendExpelGuildMember` | `guild_packets.go:956` | Expelling a member (`game/guild_lifecycle.go:77`). |
| `CZ_REQ_DISORGANIZE_GUILD` | `SendDisbandGuild` | `guild_packets.go:967` | Disbanding the guild (`ui/console.go:632`). |

### 7.2 Positions & permissions

| Packet | Method | `file:line` | Trigger |
|---|---|---|---|
| `CZ_REQ_CHANGE_MEMBERPOS` | `SendGuildMemberPositions` | `guild_packets.go:1022` | Batch-reassigning member ranks (`game/guild.go:296`). |
| `CZ_REG_CHANGE_GUILD_POSITIONINFO` | `SendGuildPositions` | `guild_packets.go:1033` | Editing position definitions (name/permissions/pay) (`game/guild.go:321`). |
| `CZ_REQ_GUILD_MENUINTERFACE` | `SendGuildMenuInterfaceRequest` | `guild_packets.go:1055` | Opening the guild window, requesting the local player's permission bitmask (`game/world.go:1346`) — response is `ZC_ACK_GUILD_MENUINTERFACE`, `IN-PACKET-DETAILS.MD` §6.4. |
| `CZ_REQ_GUILD_MENU` | `SendGuildMenuRequest` | `guild_packets.go:1066` | Switching guild-window tabs (`game/world.go:1363`), `tab` selects which sub-view to (re)request. |

### 7.3 Relations & alliances

<pre>
CZ_REQ_ALLY_GUILD    → propose an alliance
CZ_ALLY_GUILD         → target accepts/declines
CZ_REQ_HOSTILE_GUILD → declare another guild hostile
CZ_REQ_DELETE_RELATED_GUILD → remove an existing alliance/hostility entry
</pre>

| Packet | Method | `file:line` | Trigger |
|---|---|---|---|
| `CZ_REQ_ALLY_GUILD` | `SendGuildAllianceRequest` | `guild_packets.go:978` | `game/guild.go:46`. |
| `CZ_ALLY_GUILD` | `SendGuildAllianceReply` | `guild_packets.go:989` | Accepting/declining an alliance proposal (`game/guild.go:74,81`). |
| `CZ_REQ_HOSTILE_GUILD` | `SendGuildHostilityRequest` | `guild_packets.go:1000` | `game/guild.go:59`. |
| `CZ_REQ_DELETE_RELATED_GUILD` | `SendDeleteGuildRelation` | `guild_packets.go:1011` | Removing an alliance/hostility entry (`game/guild.go:135`). |

### 7.4 Chat & notice

| Packet | Method | `file:line` | Trigger |
|---|---|---|---|
| `CZ_GUILD_NOTICE` | `SendGuildNotice` | `guild_packets.go:1077` | Editing the guild bulletin board (`game/guild.go:352`). |
| `CZ_REQUEST_CHAT_GUILD` | `SendGuildMessage` | `guild_packets.go:1088` | The `$message` console prefix, via `client.SendChat` (§4) — not called directly. |

### 7.5 Emblem

| Packet | Method | `file:line` | Trigger |
|---|---|---|---|
| `CZ_REQ_GUILD_EMBLEM_IMG` | `SendGuildEmblemRequest` | `guild_packets.go:1044` | Requesting a guild's emblem bitmap by version (`game/guild_emblem.go:50`) — the outbound half of `ZC_GUILD_EMBLEM_IMG`, `IN-PACKET-DETAILS.MD` §6.5. |
| `CZ_REGISTER_GUILD_EMBLEM_IMG` | `SendGuildEmblem` | `guild_packets.go:1102` | Uploading a new emblem bitmap (`game/guild.go:260`). `BuildRegisterGuildEmblemPacket` zlib-compresses the bitmap before sending and errors if the compressed packet would exceed the 16-bit packet-length field (`guild_packets.go:889-891`). |

## 8. Trade

*In-game.* Outbound half of the state machine documented in
`IN-PACKET-DETAILS.MD` §7 — read that section for the full
request→accept→add→conclude→execute sequence; this is simply the client's
side of each step.

| Packet | Method | `file:line` | Trigger |
|---|---|---|---|
| `CZ_REQ_EXCHANGE_ITEM` | `SendTradeRequest` | `trade_packets.go:216` | Right-click "Trade" on another player (`game/trade.go:20`). |
| `CZ_ACK_EXCHANGE_ITEM` | `SendTradeAck` | `trade_packets.go:227` | Accepting/declining an incoming trade request (`game/trade.go:41,51`). |
| `CZ_ADD_EXCHANGE_ITEM` | `SendTradeAddItem` | `trade_packets.go:238` | Dragging an item into the trade window (`ui/trade_window.go:107,314`); `amount == 0` with a nonzero index is used for adding zeny (checked by the window, not this method). |
| `CZ_CONCLUDE_EXCHANGE_ITEM` | `SendTradeConclude` | `trade_packets.go:249` | Clicking "Trade" to lock in (`ui/trade_window.go:323`). |
| `CZ_CANCEL_EXCHANGE_ITEM` | `SendTradeCancel` | `trade_packets.go:260` | Canceling the trade (`ui/trade_window.go:330`). |
| `CZ_EXEC_EXCHANGE_ITEM` | `SendTradeCommit` | `trade_packets.go:271` | Clicking "OK" after both sides locked in (`ui/trade_window.go:342`). |

## 9. Adoption

*In-game.*

| Packet | Method | `file:line` | Trigger |
|---|---|---|---|
| `CZ_REQ_JOIN_BABY` | `SendAdoptionRequest` | `adoption_packets.go:76` | Proposing adoption to a target player (`game/adoption.go:38`). |
| `CZ_JOIN_BABY` | `SendAdoptionReply` | `adoption_packets.go:87` | Accepting/declining an incoming proposal (`game/adoption.go:67`) — outbound half of `ZC_REQ_BABY`, `IN-PACKET-DETAILS.MD` §8. |

## 10. Vending & NPC shops

*In-game.* Two systems, per `IN-PACKET-DETAILS.MD` §9.

### Vending

<pre>
CZ_REQ_OPENSTORE2  → configure and open a personal store
  (cancel variant: same builder, empty item list)
CZ_REQ_CLOSESTORE  → close it
CZ_REQ_BUY_FROMMC  → request another player's store's item list
CZ_PC_PURCHASE_ITEMLIST_FROMMC → buy from it
</pre>

| Packet | Method | `file:line` | Trigger |
|---|---|---|---|
| `CZ_REQ_OPENSTORE2` | `SendOpenVendingStore` | `vending_packets.go:289` | Confirming the vending setup dialog (`ui/vending_window.go:770`). |
| `CZ_REQ_OPENSTORE2` (cancel) | `SendCancelVendingStoreOpen` | `vending_packets.go:300` | Canceling the setup dialog (`ui/vending_window.go:813`) — same opcode as open, sent with an empty payload; `network/vending_packets.go` builds it via a separate helper rather than overloading `SendOpenVendingStore` with a nil-items case. |
| `CZ_REQ_CLOSESTORE` | `SendCloseVendingStore` | `vending_packets.go:278` | Closing your own store (`ui/vending_window.go:803`). |
| `CZ_REQ_BUY_FROMMC` | `SendVendingListRequest` | `vending_packets.go:311` | Clicking another player's vending sign (`game/vending.go:65`). |
| `CZ_PC_PURCHASE_ITEMLIST_FROMMC` | `SendVendingPurchase` | `vending_packets.go:322` | Confirming a purchase from someone else's store (`ui/vending_window.go:792`). |

### NPC shops

| Packet | Method | `file:line` | Trigger |
|---|---|---|---|
| `CZ_ACK_SELECT_DEALTYPE` | `SendShopDealSelection` | `item_packets.go:1669` | Choosing buy or sell at an NPC shop prompt (`ui/shop_window.go:1126`). |
| `CZ_PC_SELL_ITEMLIST` | `SendShopSellItems` | `item_packets.go:1680` | Confirming a multi-item sell (`ui/shop_window.go:1035,1107`). |
| `CZ_PC_PURCHASE_ITEMLIST` | `SendShopBuyItems` | `item_packets.go:1691` | Confirming a multi-item buy (`ui/shop_window.go:1058,1103`). |

## 11. Inventory, equipment & minigames

*In-game.*

### Pickup, use, drop

| Packet | Method | `file:line` | Trigger |
|---|---|---|---|
| `CZ_ITEM_PICKUP` | `SendItemPickup` | `item_packets.go:1460` | Clicking a floor item (`game/items.go:257`, `WorldMode.sendPickupRequest`, gated by the player not being in Hide). |
| `CZ_USE_ITEM` | `SendUseInventoryItem` | `item_packets.go:1471` | Using/consuming an inventory item (`ui/inventory_actions.go:35`). |
| `CZ_ITEM_THROW` | `SendDropInventoryItem` | `item_packets.go:1482` | Dropping an item to the ground (`ui/inventory_actions.go:42`). |

### The four "list → choose → ack" minigames

Same shape as the inbound flows in `IN-PACKET-DETAILS.MD` §10 — these are
the requests that produce those list/ack packets.

| Flow | Packet | Method | `file:line` | Trigger |
|---|---|---|---|---|
| Identify | `CZ_REQ_ITEMIDENTIFY` | `SendItemIdentify` | `item_packets.go:1493` | Picking an item from the identify list (`ui/identify_window.go:221,234`). |
| Card composition | `CZ_REQ_ITEMCOMPOSITION_LIST` | `SendItemCompositionList` | `item_packets.go:1504` | Opening the "which equipment can this card go into" list (`ui/inventory_bag_window.go:441`). |
| Card composition | `CZ_REQ_ITEMCOMPOSITION` | `SendItemComposition` | `item_packets.go:1515` | Confirming the card+equipment pair (`ui/card_composition_window.go:142`). |
| Arrow crafting | `CZ_REQ_MAKINGARROW` | `SendMakingArrow` | `item_packets.go:1526` | Picking an arrow type to craft (`ui/making_arrow_window.go:172,182`); no separate ack packet on the receiving side either (`IN-PACKET-DETAILS.MD` §10 notes the same). |
| Item creation | `CZ_REQMAKINGITEM` | `SendMakingItem` | `item_packets.go:1537` | Picking a recipe + up to 3 material item IDs (`ui/making_item_window.go:183,193`). |
| Repair | `CZ_REQ_ITEMREPAIR` | `SendRepairItem` | `item_packets.go:1548` | Picking an item from the repair list (`ui/equipment_choice_window.go:279,289`). |
| Weapon refine | `CZ_REQ_WEAPONREFINE` | `SendWeaponRefine` | `item_packets.go:1559` | Picking an item from the refine list (`ui/equipment_choice_window.go:322,332`). |

### Equipping & viewing

| Packet | Method | `file:line` | Trigger |
|---|---|---|---|
| `CZ_REQ_WEAR_EQUIP` | `SendWearEquip` | `item_packets.go:1570` | Equipping an item to a slot (`ui/inventory_actions.go:57,67`). |
| `CZ_REQ_TAKEOFF_EQUIP` | `SendTakeoffEquip` | `item_packets.go:1581` | Unequipping (`ui/inventory_actions.go:51`, `ui/equipment_window.go:392`). |
| `CZ_REQ_CARTOFF` | `SendRemoveOption` | `equipment_packets.go:126` | "Remove cart" (despite the generic name, this specifically detaches the visible cart — `ui/equipment_window.go:401`). |
| `CZ_EQUIPWIN_MICROSCOPE` | `SendViewPlayerEquipment` | `equipment_packets.go:115` | Right-click "View Equipment" on another player (`game/equipment_view.go:16`) — requests `ZC_EQUIPWIN_MICROSCOPE`, `IN-PACKET-DETAILS.MD` §21. |
| `CZ_CONFIG` | `SendShowEquipConfig` | `equipment_packets.go:137` | Toggling "show my equipment to others" (`ui/equipment_window.go:221`); `BuildShowEquipConfigPacket` sends the generic `CZ_CONFIG` opcode with `ConfigOpenEquipmentWindow` as the config-type field — worth knowing if you ever need to add another `CZ_CONFIG`-based preference, since this is the only current user of that generic packet. |

## 12. Storage & cart

*In-game.* Structurally parallel to §11, per `IN-PACKET-DETAILS.MD` §11.

### Storage password

**`CZ_ACK_STORE_PASSWORD`** (`SendStoragePasswordReply`,
`storage_password_packets.go:87`) — replies to a `ZC_REQ_STORE_PASSWORD`
prompt (`IN-PACKET-DETAILS.MD` §11), sent from
`game/storage_password.go:89`. `replyType` distinguishes "verify existing
password" from "set a new one"; the 36-byte packet always carries both a
current-password field (bytes 4-19) and a new-password field (bytes
20-35), left blank when not applicable.

### Move & close

| Packet | Method | `file:line` | Trigger |
|---|---|---|---|
| `CZ_MOVE_ITEM_TO_STORAGE` | `SendMoveToStorage` | `item_packets.go:1592` | `ui/storage_window.go:167`. |
| `CZ_MOVE_ITEM_FROM_STORAGE` | `SendMoveFromStorage` | `item_packets.go:1603` | `ui/storage_window.go:362`. |
| `CZ_CLOSE_STORAGE` | `SendCloseStorage` | `item_packets.go:1614` | `ui/storage_window.go:341`, `game/storage_password.go:75`. |
| `CZ_MOVE_ITEM_TO_CART` | `SendMoveToCart` | `item_packets.go:1625` | `ui/cart_window.go:166`. |
| `CZ_MOVE_ITEM_FROM_CART` | `SendMoveFromCart` | `item_packets.go:1636` | `ui/cart_window.go:294`. |
| `CZ_MOVE_ITEM_FROM_STORAGE_TO_CART` | `SendMoveStorageToCart` | `item_packets.go:1647` | `ui/cart_window.go:187`. |
| `CZ_MOVE_ITEM_FROM_CART_TO_STORAGE` | `SendMoveCartToStorage` | `item_packets.go:1658` | `ui/storage_window.go:188`. |
| `CZ_REQ_CHANGECART` | `SendChangeCart` | `client.go:543` | Picking a cart appearance (`ui/change_cart_window.go:68`). |

## 13. Pets

*In-game.*

| Packet | Method | `file:line` | Trigger |
|---|---|---|---|
| `CZ_TRYCAPTURE_MONSTER` | `SendTryCaptureMonster` | `pet_packets.go:206` | Throwing a Pet Capture item at an eligible monster (`game/pet_slot_machine.go:48,56,92`) — corresponds to `ZC_TRYCAPTURE_MONSTER`, `IN-PACKET-DETAILS.MD` §12. |
| `CZ_COMMAND_PET` | `SendPetCommand` | `pet_packets.go:217` | Pet menu commands (feed/return to egg/etc., `game/pet.go:502`). |
| `CZ_PET_ACT` | `SendPetAct` | `pet_packets.go:228` | Triggering a pet "performance" action (`game/pet.go:307,324`). |
| `CZ_RENAME_PET` | `SendRenamePet` | `pet_packets.go:239` | Renaming a newly-hatched pet (`ui/pet_info_window.go:155`). |
| `CZ_SELECT_PETEGG` | `SendSelectPetEgg` | `pet_packets.go:250` | Picking which egg to hatch (`ui/pet_egg_window.go:127`) — response to `ZC_PETEGG_LIST`, `IN-PACKET-DETAILS.MD` §12. |

## 14. Homunculus & mercenary

*In-game.* Mirrored packet shapes for the two companion systems, per
`IN-PACKET-DETAILS.MD` §13.

| Packet | Method | `file:line` | Trigger |
|---|---|---|---|
| `CZ_COMMAND_MER` | `SendHomunculusCommand` | `client.go:452` | Homunculus menu commands (`game/homunculus.go:67`). |
| `CZ_COMMAND_MER` (feed) | `SendHomunculusFeed` | `client.go:463` | Feeding the homunculus (`game/homunculus.go:92`) — same opcode as the generic command, sent with `HomunculusCommandFeed` as the command byte. |
| `CZ_COMMAND_MER` (delete) | `SendHomunculusDelete` | `client.go:474` | Releasing/deleting the homunculus (`game/homunculus.go:103`) — same opcode again, `HomunculusCommandDelete`. |
| `CZ_RENAME_MER` | `SendHomunculusRename` | `client.go:485` | Renaming the homunculus (`game/homunculus.go:120`). |
| `CZ_REQUEST_MOVENPC` | `SendCompanionMove` | `client.go:496` | Commanding a homunculus/mercenary to move to a tile (`game/companion_ai.go:315`, `game/skill_use.go:632`). Despite "NPC" in the opcode name, this targets player-owned companions. |
| `CZ_REQUEST_ACTNPC` | `SendCompanionAttack` | `client.go:510` | Commanding a companion to attack (`game/companion_ai.go:328`). |
| `CZ_REQUEST_MOVETOOWNER` | `SendCompanionMoveToOwner` | `client.go:521` | Recalling a companion to the player's side (`game/companion_ai.go:301`). |
| `CZ_MER_COMMAND` | `SendMercenaryCommand` | `client.go:532` | Mercenary-specific menu commands (`game/mercenary.go:76`) — mercenaries have their own command opcode, unlike homunculi which reuse `CZ_COMMAND_MER` for feed/delete. |

## 15. Skills, casting & warp points

*In-game.*

- **`CZ_STATUS_CHANGE`** (`SendStatusIncrease`, `client.go:375`) —
  spending a stat point (`ui/stats_window.go:182`).
- **`CZ_UPGRADE_SKILLLEVEL`** (`SendSkillLevelUp`, `client.go:386`) —
  spending a skill point (`ui/skill_window.go:748`, `ui/homunculus_skill_window.go:499`,
  `game/guild.go:400` for guild skills — one opcode, three different skill
  trees).
- **`CZ_SELECTAUTOSPELL`** (`SendSelectAutoSpell`, `client.go:408`) —
  picking a sub-spell from a `ZC_AUTOSPELLLIST` choice (`game/auto_spell.go:32`),
  the outbound half of `IN-PACKET-DETAILS.MD` §16.
- **`CZ_USE_SKILL`** (`SendUseSkillToID`, `client.go:419`) — casting a
  targeted skill. **`CZ_USE_SKILL_TOGROUND`**
  (`SendUseSkillToGround`, `client.go:430`) and
  **`CZ_USE_SKILL_TOGROUND_WITHTALKBOX`**
  (`SendUseSkillToGroundWithText`, `client.go:441`, ground skills that also
  carry a text field, e.g. Mercenary/homunculus naming-style skills) —
  ground-targeted casts.

  These three are not called directly by UI/game code for normal
  skill-use — instead, `game/skill_use.go` defines a small
  `skillController` wrapper (`SendToID`/`SendToGround`/
  `SendToGroundWithText`, `game/skill_use.go:280-560`) that adds
  pre-flight checks (stealth-breaking, player-death, "is this skill
  actually learned") and post-send bookkeeping (remembering the last
  skill/target for `ContinuePendingTarget` re-tries, and — for level-1
  Teleport specifically — immediately following up with
  `SendSelectWarpPoint` since that skill's random-destination behavior
  doesn't wait for a `ZC_WARPLIST`) before calling the real
  `network.Client` methods above. If you're tracing "why didn't my skill
  cast send a packet," `skillController.SendToID`/`SendToGround` is where
  to look first, not `network/client.go`.
- **`CZ_REMEMBER_WARPPOINT`** (`SendRememberWarpPoint`, `client.go:397`)
  — saving the current map as a Warp Portal memo point (`ui/console.go:807`)
  — request for `ZC_ACK_REMEMBER_WARPPOINT`, `IN-PACKET-DETAILS.MD` §16.
- **`CZ_SELECT_WARPPOINT`** (`SendSelectWarpPoint`, `client.go:554`) —
  picking a destination from the warp list (`ui/teleport_modal.go:112,131`,
  `game/teleport_handlers.go:78`) or, as noted above, the level-1 Teleport
  auto-follow-up (`game/skill_use.go:321`).

## 16. Actor name lookup

*In-game.* **`CZ_REQNAME`** (`SendNameRequest`, `client.go:239`) — asks
the server to resolve a GID's display name; `WorldMode` calls this
(`game/actor.go:1260`) when it encounters an actor it doesn't have a
cached name for yet. `BuildNameRequestPacketForClientDate` can return
`ok=false` for packet-version profiles that don't support this request at
all, in which case `SendNameRequest` silently skips sending and logs a
warning rather than erroring — a name-lookup failure here is not
surfaced as an error to the caller.

## 17. Mail

*In-game.* The legacy 2008-era mail system's outbound half — see
`IN-PACKET-DETAILS.MD`'s note on the pre-filtered `handleMailPacket`
family for the inbound side (`network/mail_packets.go`,
`game/mail.go:183`).

| Packet | Method | `file:line` | Trigger |
|---|---|---|---|
| `CZ_MAIL_REFRESH` | `SendMailRefresh` | `mail_packets.go:256` | Opening/refreshing the mail list (`game/mail.go:74`). |
| `CZ_MAIL_READ` | `SendMailRead` | `mail_packets.go:257` | Opening a specific mail (`game/mail.go:84,309`). |
| `CZ_MAIL_DELETE` | `SendMailDelete` | `mail_packets.go:258` | Deleting a mail (`game/mail.go:104`). |
| `CZ_MAIL_GET_ATTACH` | `SendMailGetAttachment` | `mail_packets.go:259` | Claiming an attached item/zeny (`game/mail.go:97`). |
| `CZ_MAIL_RESET` | `SendMailReset` | `mail_packets.go:262` | Clearing the compose-window attachment state; called at several points in `game/mail.go` (`:62,67,72,123,156,335`) whenever the compose flow needs to reset before/after another action. `MailResetType` distinguishes which part is being cleared. |
| `CZ_MAIL_ADD_ATTACH` | `SendMailAddAttachment` | `mail_packets.go:263` | Attaching an inventory item to an outgoing mail (`game/mail.go:115,127`). |
| `CZ_MAIL_RETURN` | `SendMailReturn` | `mail_packets.go:266` | Returning a mail to its sender (`game/mail.go:106`). |
| `CZ_MAIL_SEND` | `SendMail` | `mail_packets.go:269` | Sending a composed mail (`game/mail.go:129,277`). `BuildMailSendPacket` validates recipient/subject/body length and UTF-8/NUL-byte content client-side before ever building the packet. |

## 18. Hotkeys

*In-game.* **`CZ_SHORTCUT_KEY_CHANGE`** (`SendHotkey`, `hotkey_packets.go:62`)
— setting a single shortcut-bar slot to an item or skill (`ui/shortcut_bar.go:736`).
Corresponds to one entry of the 27-slot `ZC_SHORTCUT_KEY_LIST` snapshot
documented in `IN-PACKET-DETAILS.MD` §1 (via `ARCHITECTURE.md`'s note on
`applyHotkeyList`) — each drag onto the shortcut bar sends one of these
rather than resending the whole list.

## 19. PvP info & fame ranking

*In-game.*

- **`CZ_REQUEST_PVPINFO`** (opcode constant `PacketCZRequestPvPInfo`;
  `SendPvPInfoRequest`, `pvp_packets.go:83`) — no `glog.Debugf` call on
  this one, so the opcode name above comes from the Go constant, not a log
  line. Sent from the `/pvpinfo` console command via `client.SendChat`
  (`client/chat.go:42`, see §4), requesting the local player's
  win/loss/point record — the response, `ZC_ACK_PVPPOINT`
  (`ParsePvPInfo`), *is* covered in `IN-PACKET-DETAILS.MD`, though that
  document doesn't cross-reference this send since it only catalogs
  `handleNetworkPacket`'s inbound chain.
- **`CZ_BLACKSMITH_RANK` / `CZ_ALCHEMIST_RANK` / `CZ_TAEKWON_RANK`**
  (`SendFameRankingRequest`, `ranking_packets.go:92`) — requesting one of
  the three fame leaderboards (`ui/console.go:818`, presumably a
  `/blacksmithrank`-style command); `BuildFameRankingRequestPacket` maps
  the requested `FameRankingKind` to the correct one of the three opcodes.

## 20. Star Gladiator (Taekwon)

*In-game.* **`CZ_AGREE_STARPLACE`** (`SendAgreeStarPlace`,
`taekwon_packets.go:61`) — confirming which of up to 3 offered "star
places" (feel/sign locations) a Star Gladiator player accepts
(`game/taekwon.go:30`); the method validates `place <= 2` client-side
before building the packet, mirroring the server-side bounds check
`IN-PACKET-DETAILS.MD` notes on the inbound `ZC_STARPLACE`.

## 21. Auto revive

*In-game.* **`SendAutoRevive`** (`revive_packets.go:13`, opcode constant
`PacketCZAutoRevive = 0x0292`, `revive_packets.go:5`) has no
`glog.Debugf` call either — the shortest file in the whole `network`
package (15 lines): build a bare 2-byte opcode packet, send it, no
parameters at all. Called from `client/revive.go:33` when the player
confirms an in-place revive (e.g. an Acolyte's Resurrection-adjacent
prompt) rather than returning to the save point.

## 22. Quests

*In-game.* **`CZ_QUEST_ACTIVE`** (`SendQuestActive`, `quest_packets.go:146`)
— marking a quest active/inactive in the quest journal (`game/quests.go:89`),
the one outbound packet for the otherwise inbound-only quest family
`IN-PACKET-DETAILS.MD` mentions is pre-filtered via `handleQuestPacket`.

## 23. Misc / UI preference & progress

*In-game.*

- **`CZ_LESSEFFECT`** (`SendLessEffect`, `effect_packets.go:85`) — the
  "reduce visual effects" toggle. Sent from three places: the settings
  window (`ui/settings_window.go:170`), the console command
  (`ui/console.go:545`), and automatically by
  `game/less_effects.go:12`/`sendLessEffectPreference` right after every
  `ZC_ACCEPT_ENTER` (map load) — see `IN-PACKET-DETAILS.MD` §1 and §21,
  since servers reset this preference per map-enter and the client
  proactively re-asserts it.
- **`CZ_PROGRESS`** (`SendProgressBarDone`, `server_info_packets.go:130`)
  — tells the server the client-side progress bar UI (`ZC_PROGRESS`,
  `IN-PACKET-DETAILS.MD` doesn't catalog this by name but it's the
  `ParseProgressBar`/`ParseProgressBarCancel` pair) finished playing,
  sent from `game/server_progress.go:69` when the local timer completes.

---

*For how to add a new outgoing packet, see `development-docs/ARCHITECTURE.md`'s
"How to..." section ("Add a new outgoing packet").*
