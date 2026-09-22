# Packet Interaction Flows

`IN-PACKET-DETAILS.MD` catalogs every packet the server sends; `OUT-PACKET-DETAILS.MD`
catalogs every packet the client sends. Neither document tells you the story
of a single player action end-to-end — what goes out, what plausibly happens
on the server, what comes back, and what the client's code actually does
with the reply (including the parts a packet table can't show: local
prediction, queued/pending state, retries, and failure handling). That story
is this document's job.

This is a **synthesis** document, not a third catalog. It assumes you've
skimmed `ARCHITECTURE.md`'s "Networking" section (packet framing, the
per-frame `Client.Pump`/`DrainPackets` cycle, the `handleNetworkPacket`
dispatch chain) and doesn't re-explain that machinery. Every flow below
cross-references `IN-PACKET-DETAILS.MD`/`OUT-PACKET-DETAILS.MD` by section
instead of re-pasting their tables, and every claim about client-side
behavior carries a `file:line` reference. **This codebase contains no
server source** — only the client — so anywhere this document describes what
the server does or why, that is an inference from the client's reaction to
its packets, not a fact read out of server code. Those spots are called out
explicitly.

## Table of contents

1. [Movement](#1-movement) — the flagship example
2. [Melee/ranged attack](#2-meleeranged-attack)
3. [Skill cast](#3-skill-cast)
4. [Item pickup](#4-item-pickup)
5. [NPC dialog & shop purchase](#5-npc-dialog--shop-purchase)
6. [Trade](#6-trade)
7. [Party invite and guild invite (three-party flows)](#7-party-invite-and-guild-invite-three-party-flows)
8. [Storage password gate](#8-storage-password-gate)
9. [Pet capture](#9-pet-capture)
10. [Vending](#10-vending)
11. [Item minigames (identify, and the rest)](#11-item-minigames-identify-and-the-rest)
12. [Map transition / warp](#12-map-transition--warp)
13. [Login → character select → map enter](#13-login--character-select--map-enter)

---

## 1. Movement

Clicking the ground is the single most common action in the game, and it's
the best flow to understand first because the attack, pickup, and skill
flows below all reuse its exact "walk there, then do the thing" pattern.

### The wire round-trip

```
client → server   CZ_REQUEST_MOVE        SendWalkToXY(x, y)
server → client   ZC_NOTIFY_PLAYERMOVE   ParseSelfMoveAck (From/To + server tick)
```

`WorldMode.requestWalk` (`game/movement.go:153`) builds and sends
`CZ_REQUEST_MOVE` — see `OUT-PACKET-DETAILS.MD` §2 — after checking the
player isn't dead, isn't hidden (unless they know Tunnel Drive), and the
target tile is in map bounds; `walkRequestCooldown` (200ms,
`game/movement.go:17`) throttles repeat requests so a mouse held down or a
spammed click doesn't flood the connection. On success it starts a cooldown
timer; on failure (e.g. `ctx.Network` briefly unavailable) it starts a
*longer* cooldown (`walkErrorCooldown`, 500ms) before the player can try
again.

### There is no optimistic movement — the client waits for the ack

This is the detail worth internalizing before anything else: **`requestWalk`
does not touch `world.Player`'s position at all.** It only calls
`SendWalkToXY` and sets a cooldown. The player sprite does not start moving
the instant you click. Movement only begins once `ZC_NOTIFY_PLAYERMOVE`
comes back and `applySelfMoveAck` (`game/movement.go:327`) calls
`setPlayerMovementAt`, which is what actually populates
`world.Actor.FromX/FromY/ToX/ToY/MoveStarted/MoveDuration/MovePath` — the
interpolation fields `ARCHITECTURE.md` and `GRAPHICS.md`'s camera/sprite
sections describe. So "client-side prediction" here doesn't mean "start
moving immediately and correct later" (the way some action games do it); it
means "the client doesn't render movement until the server confirms it,"
which for a ~single-digit-millisecond LAN round-trip to a local rAthena is
imperceptible, and for a slow connection just means a visible pause before
the character actually steps off.

`applySelfMoveAck` also reads `ack.ServerTick` and calls
`ctx.Session.ElapsedSinceServerTick` (`game/movement.go:331`) to compute a
`fastForward` duration — if the ack arrived late enough that real time has
already passed since the server started the move, `setPlayerMovementPathAt`
(`game/movement.go:349`) uses that duration to start the interpolation
partway through the path rather than at the beginning, so a laggy ack
doesn't visually "rewind" the character to the start of a walk that the
server considers partially complete.

Other actors' movement (`upsertActorAt`, `game/movement.go:388`, triggered by
`ActorEntry`/`ParseActorMoveEntryModern` variants — see
`IN-PACKET-DETAILS.MD` §22) works the same way: the server always supplies
the from/to pair and (usually) a start tick; the client never predicts
another actor's walk either.

### `ZC_STOPMOVE` and `ZC_HIGHJUMP`

`ZC_STOPMOVE` (`ParseActorSetPosition`, `IN-PACKET-DETAILS.MD` §2) is the
server unilaterally placing an actor at an exact cell, overriding any
interpolation currently in flight — `applyActorSetPosition`
(`game/movement.go:443`) calls `ctx.World.SetPlayerPosition` directly rather
than starting a new interpolated move, so the local player visually snaps.
`ZC_HIGHJUMP` (`ParseActorJumpPosition`, `applyActorJumpPosition`,
`game/movement.go:457`) is structurally identical — same snap-to-position
behavior — used for skills/scripts that teleport rather than walk an actor.

### The out-of-range pattern: walk-then-retry

This is the mechanism that makes "click to attack/pick-up/cast something
that's currently out of range" work, and it's the same three-step shape in
all three cases:

1. **Queue an intent.** `requestAttack` (`game/battle.go:256`),
   `requestPickup` (`game/items.go:159`), and
   `skillController.chaseSkillTargetIfNeeded` (`game/skill_use.go:566`) all
   do the same thing when the target is out of range: compute an approach
   cell within range of the target, store the intent
   (`m.pendingAttack = attackIntent{targetID, expires, ...}`,
   `game/battle.go:288`; `m.pendingPickup = pickupIntent{itemID, expires}`,
   `game/items.go:167`; `c.mode.pendingSkill = pending`,
   `game/skill_use.go:609`), and call `requestWalk`/`requestSkillCasterMove`
   toward that approach cell. Each intent carries an 8-second expiry so a
   target that never becomes reachable doesn't leave the client silently
   "stuck" trying forever.
2. **Wait for a movement-related ack.** `ZC_NOTIFY_PLAYERMOVE`
   (`applySelfMoveAck`) and `ZC_STOPMOVE` (`applyActorSetPosition`) both, for
   the local player specifically, call the same three resume functions in
   sequence right after applying the position
   (`game/world_packets.go:313-315` and `:326-328`):
   ```
   m.continuePendingAttack(ctx, "walk ack" | "position fix")
   m.continuePendingPickup(ctx, "walk ack" | "position fix")
   m.skills().ContinuePendingTarget(ctx, "walk ack" | "position fix")
   ```
   All three are no-ops if there's no matching pending intent
   (`m.pendingAttack.targetID == 0` etc.), so this costs nothing on the
   common case of an ack for a walk that wasn't chasing anything.
3. **Re-check range, then either fire or chase again.** `updatePendingAttack`
   (`game/battle.go:339`) re-measures the distance to the target's *current*
   cell (the target may itself have moved while the player was walking); if
   now in range it schedules the actual attack via
   `pendingAttackReadyAt` + `processPendingAttack` (`game/battle.go:382`,
   driven every `Update` tick, not from inside the packet handler itself —
   see below); if still out of range it calls `requestAttack` again, which
   re-computes a fresh approach cell and re-queues, throttled by
   `attackRetryDue`/`attackRetryInterval` (1.2s, `game/battle.go:226`) so a
   fast-moving target doesn't trigger a walk request every single frame.
   `updatePendingPickup`/`chaseSkillTargetIfNeeded` mirror this exactly for
   pickup and skill targets respectively.

Note the two-phase design for attacks/skills: the packet handler only
*schedules* readiness (`pendingAttack.readyAt`), and a separate per-frame
`processPendingAttack`/`processPendingPickup` call (driven from
`WorldMode.Update`, not shown in the packet-dispatch chain at all) is what
actually fires `sendAttackAction`/`sendPickupRequest` once `readyAt` has
passed. This exists so the action doesn't fire mid-frame from inside packet
processing, and so a small "weapon-ready" delay
(`pendingAttackReadyAt`/`pendingPickupReadyAt`, `+60ms`, extended to cover
the tail of an in-flight walk) is respected uniformly regardless of whether
the action became ready because of a walk-ack or was already in range from
the start.

### `ZC_ATTACK_FAILURE_FOR_DISTANCE` — the server's own out-of-range rejection

Even after all of the above, the server can still reject an attack the
client believed was in range (its own position tracking may be a tick or two
ahead of what the server has committed). `applyAttackFailureForDistance`
(`game/battle.go:1406`, see `IN-PACKET-DETAILS.MD` §2) hard-snaps *both* the
local player and the target to the server-authoritative positions the
failure packet carries (`ctx.World.SetPlayerPosition`, direct actor field
writes — not an interpolated move), then, if the failed attack was the one
currently locked/pending, clears `m.pendingAttack` and — if the
server-authoritative positions are *still* out of range —
calls `m.requestAttack` again to restart the chase from the corrected
position. This is the same walk-then-retry loop from above, just re-entered
from a rejection instead of from an ack.

---

## 2. Melee/ranged attack

```
client → server   CZ_REQUEST_ACT (action=ActionAttack)   SendActionRequest
server → client   ZC_NOTIFY_ACT / ZC_NOTIFY_SKILL         ParseActorActionNotify  (hit, on success)
              or  ZC_ATTACK_FAILURE_FOR_DISTANCE          ParseAttackFailureForDistance  (rejected)
```

`sendAttackAction` (`game/battle.go:478`) is the actual `SendActionRequest`
call site; it's reached either directly from `requestAttack` when the target
is already in range, or from `processPendingAttack`/`processLockedAttack`
once a chased target becomes reachable and ready (§1). On success it starts
`walkRequestCooldown` and records `lastAttackAt` for the locked-attack
retry-rate limiter (`game/battle.go:226`); a `SendActionRequest` transport
error (not a game-level rejection — an actual send failure) sets the longer
`walkErrorCooldown` instead.

On success, the server's `ZC_NOTIFY_ACT`/`ZC_NOTIFY_SKILL` family
(`ParseActorActionNotify`, `IN-PACKET-DETAILS.MD` §14) is what actually
carries the damage number and drives the swing animation —
`applyActorActionNotify` (`game/battle.go:492`) reads `action.SourceSpeed`
to time the attacker's animation against the server's authoritative attack
speed (not the client's own guess), so the animation and the moment the
damage number appears stay synced with what the server considers to have
happened, even under variable latency.

On rejection, `ZC_ATTACK_FAILURE_FOR_DISTANCE` triggers the correction/retry
loop described in §1 — that failure path is shared between the initial
attack and any chase-and-retry attack, so it's documented once there rather
than twice.

There is no separate "attack was rejected for a reason other than distance"
packet in this codebase's catalog (no ammo/target-invalid/etc. ack distinct
from the distance-failure one) — a rejected attack that isn't a range
problem apparently just produces no `ZC_NOTIFY_ACT` at all, and the client
has no dedicated handling for that silence beyond the attack simply not
animating. This is inferred from the absence of another incoming packet in
`IN-PACKET-DETAILS.MD`'s catalog, not confirmed against server source.

---

## 3. Skill cast

Skill casting has two distinct shapes — target-a-unit and target-a-ground-cell
— and both funnel through a small wrapper, `skillController`
(`game/skill_use.go`), rather than calling `network.Client`'s
`SendUseSkillToID`/`SendUseSkillToGround` methods directly. If you're
tracing "why didn't my skill send a packet," start at `skillController`, not
`network/client.go` — `OUT-PACKET-DETAILS.MD` §15 notes this same thing.

### Unit-targeted skill

```
client → server   CZ_USE_SKILL              skillController.SendToID → SendUseSkillToID
server → client   ZC_USESKILL_ACK            ParseSkillCastNotify   (cast-time began)
                    ... DelayTime elapses client-side ...
server → client   ZC_NOTIFY_ACT / ZC_USE_SKILL / ZC_SKILL_ENTRY   (outcome, once cast completes server-side)
              or  ZC_ACK_TOUSESKILL (Result != 0)                  ParseSkillFailAck   (rejected up front)
```

`skillController.UseTarget` (`game/skill_use.go:469`) is reached from a
click while a skill is pending target selection
(`skillController.HandleClick`, `game/skill_use.go:431`). Before sending
anything it runs client-side pre-flight checks that mirror what the server
will also enforce: the caster isn't dead, the skill is actually learned
(`skill.Level > 0`), it isn't a ground/self-target skill routed here by
mistake, and — via `chaseTargetIfNeeded` (`game/skill_use.go:561`) — whether
the target is within the skill's range at all. If not, it re-enters the same
walk-then-retry pattern from §1 (`chaseSkillTargetIfNeeded`,
`game/skill_use.go:566`, sets `pendingSkill` and calls `requestWalk` or, for
homunculus/mercenary-cast skills, `SendCompanionMove`,
`game/skill_use.go:615`); only once in range does it actually call
`SendToID` → `network.Client.SendUseSkillToID`.

`SendToID` (`game/skill_use.go:294`) itself does one more thing worth
knowing: for **level-1 Teleport specifically**, it immediately follows the
skill-use send with `SendSelectWarpPoint(skill.ID, TeleportRandomMap)`
(`game/skill_use.go:321`) — that skill's original-client behavior is an
instant random-destination warp with no `ZC_WARPLIST` round-trip, so the
client fires both packets back-to-back rather than waiting for a list it
knows won't arrive for this specific skill. See §12 for the general warp
flow this short-circuits.

**Success**: `ZC_USESKILL_ACK` (`ParseSkillCastNotify`,
`IN-PACKET-DETAILS.MD` §16) arrives — `applySkillCastNotify`
(`game/effects.go:997`) starts the caster's swing/cast animation and, unless
the specific skill hides it (`skillHidesCastBar`), a visible cast bar
(`startActorCastBar`, `game/effects.go:1012`) sized to `notify.DelayTime`.
**The client does not send anything when the cast bar finishes** — cast-time
completion is purely a local animation timer; the actual skill *resolution*
is a separate, later incoming packet the server sends on its own schedule
once it decides the cast completed (and wasn't interrupted): a damage skill
arrives as `ZC_NOTIFY_ACT`/`ZC_NOTIFY_SKILL` (same packet family as a normal
attack, §2, via `applyActorActionNotify`); a non-damaging buff/heal arrives
as `ZC_USE_SKILL`/`ZC_USE_SKILL2` (`ParseSkillNoDamageNotify`,
`applySkillNoDamageNotify`, `game/effects.go:960`, which is also what
actually spawns the skill's visual effect table entries — see
`GRAPHICS.md` §7 for how `worldEffectSpec`/`worldEffectComponent` render
those); a persistent ground effect (traps, walls) arrives as
`ZC_SKILL_ENTRY` (`ParseSkillUnitEntry`, `IN-PACKET-DETAILS.MD` §16's note
on this opcode's dual use).

**Rejection**: `ZC_ACK_TOUSESKILL` with a nonzero `Result`
(`ParseSkillFailAck`, `applySkillFailAck`, `game/effects.go:1253`) — the
client does **not** retry a rejected skill use; it just prints the mapped
failure reason (`skillFailMessage`) as a console error and clears any
pending sense-target bookkeeping (`m.clearSenseRequest`). This is different
from the attack-distance-failure path in §1/§2, which *does* retry — a
skill failure (insufficient SP, wrong weapon, on cooldown, etc.) is
generally not something walking closer would fix, so there's no retry loop
here.

### Ground-targeted skill

```
client → server   CZ_USE_SKILL_TOGROUND (or _WITHTALKBOX)   skillController.SendToGround(WithText)
server → client   ZC_USESKILL_ACK                            ParseSkillCastNotify (X/Y instead of TargetID)
                    ... same cast-time/outcome pattern as above ...
```

Structurally identical to the unit-targeted case — same pre-flight checks,
same `chaseSkillTargetIfNeeded` distance handling (using the clicked
ground cell instead of an actor's cell), same `ZC_USESKILL_ACK` cast-bar
trigger. The only difference is the outbound opcode and payload shape
(`SendUseSkillToGround`/`SendUseSkillToGroundWithText`,
`OUT-PACKET-DETAILS.MD` §15) and that `skillNeedsGroundText` skills (e.g.
Talkie Box, Graffiti) open a text-input prompt
(`openSkillTextPrompt`, `game/skill_use.go:443`) before sending, so the
player supplies the ground text client-side first.

---

## 4. Item pickup

```
client → server   CZ_ITEM_PICKUP        SendItemPickup(itemID)
server → client   ZC_ITEM_PICKUP_ACK    ParseItemPickupAck  (success or failure)
```

Reuses the exact walk-then-retry shape from §1: `requestPickup`
(`game/items.go:159`) checks the player isn't in Hide and isn't dead, then
either calls `sendPickupRequest` directly (in range) or computes an approach
cell, sets `m.pendingPickup = pickupIntent{itemID, expires}`
(`game/items.go:167`), and walks there — resumed by
`continuePendingPickup` from the same `ZC_NOTIFY_PLAYERMOVE`/`ZC_STOPMOVE`
acks as §1, and fired from the per-frame `processPendingPickup`
(`game/items.go:222`) once `pendingPickupReadyAt` passes, exactly mirroring
`processPendingAttack`.

One difference from attack/skill: there is no server-side "you're too far
away" rejection packet for pickup in this catalog — pickup success/failure
is reported as a single `ZC_ITEM_PICKUP_ACK` with a result code, not split
into a separate distance-failure packet. `applyItemPickupAck`
(`game/items.go:114`) distinguishes success (`itemPickupResultSuccess` or
`itemPickupResultReceive`, the latter meaning "merged into an existing
stack" — it computes `gained` as the delta against the previous stack
amount rather than assuming the whole `Amount` field is newly gained,
`game/items.go:126-133`) from failure by result code alone; on failure the
world_packets.go handler (`IN-PACKET-DETAILS.MD` §10) posts a red console
error naming the item and result code, with no retry — if the pickup failed
because another player grabbed the item first, the preceding
`ZC_ITEM_DISAPPEAR` (`applyFloorItemDisappear`, `game/items.go:97`) has
already cleared `m.pendingPickup` for that item ID anyway, so there's
nothing left to retry against.

Also removes the floor item's world-state entry as a side effect of a
disappear notice specifically, not of the ack — see `IN-PACKET-DETAILS.MD`
§10 for `ZC_ITEM_DISAPPEAR` vs. `ZC_ITEM_PICKUP_ACK`, two different incoming
packets that can both end an item's on-screen lifetime.

---

## 5. NPC dialog & shop purchase

NPC interaction is the longest incoming/outgoing alternation of any flow in
this document, because RO's NPC dialog system is a small stateful
mini-protocol in its own right — a sequence of narrow request/response pairs
rather than one request producing one response.

### Talking to an NPC

```
client → server   CZ_CONTACTNPC          SendNPCContact(npcID)      — click the NPC
server → client   ZC_SAY_DIALOG          ParseNPCDialog (text line)
client → server   CZ_REQ_NEXT_SCRIPT     SendNPCNext(npcID)          — click "next"
server → client   ZC_WAIT_DIALOG / ZC_SAY_DIALOG / ZC_MENU_LIST / ZC_CLOSE_DIALOG / ...
   ... repeats: next line, a menu choice, a number/text prompt, or a close ...
```

`requestNPCTalk` (`game/world.go:1543`) sends `CZ_CONTACTNPC` when the
player clicks an NPC actor (after clearing any locked attack — you can't be
mid-combat-lock and start a conversation). Every subsequent step is driven
by `ui/npc_dialog.go` reacting to the *currently displayed* `NPCDialogKind`
(`IN-PACKET-DETAILS.MD` §19) and sending the matching outbound packet
(`OUT-PACKET-DETAILS.MD` §3) once the player interacts with that specific
dialog widget:

| Server shows (`ZC_SAY_DIALOG` sub-kind) | Player does | Client sends |
|---|---|---|
| a text line, `ZC_WAIT_DIALOG` | clicks anywhere/"next" | `CZ_REQ_NEXT_SCRIPT` (`ui/npc_dialog.go:253`) |
| `ZC_MENU_LIST` | picks an option | `CZ_CHOOSE_MENU` (`ui/npc_dialog.go:347`) |
| `ZC_OPEN_EDITDLG` | types a number, confirms | `CZ_INPUT_EDITDLG` (`ui/npc_dialog.go:295`) |
| `ZC_OPEN_EDITDLGSTR` | types text, confirms | `CZ_INPUT_EDITDLGSTR` (`ui/npc_dialog.go:297`) |
| `ZC_CLOSE_DIALOG`/window close button | closes the window | `CZ_CLOSE_DIALOG` (`ui/npc_dialog.go:322`) |

There's no client-side timeout or retry anywhere in this loop — the dialog
window just sits open until the player acts or the server sends
`ZC_CLEAR_DIALOG`/`ZC_CLOSE_DIALOG` unprompted (e.g. an NPC script ending
itself). This is a pure ping-pong: the client only ever sends in direct
response to the dialog state the server most recently put it in, never
proactively.

### Buying/selling at a shop

Continuing from the dialog loop above once the NPC's script reaches its shop
step:

```
server → client   ZC_SELECT_DEALTYPE        ParseShopDealSelection   — offers buy/sell choice
client → server   CZ_ACK_SELECT_DEALTYPE    SendShopDealSelection    — player picks one
server → client   ZC_PC_SELL_ITEMLIST / ZC_PC_PURCHASE_ITEMLIST      — the matching item list
client → server   CZ_PC_SELL_ITEMLIST / CZ_PC_PURCHASE_ITEMLIST      — confirmed cart of items+amounts
server → client   ZC_PC_SELL_RESULT / ZC_PC_PURCHASE_RESULT          ParseShopResult
```

`ShopWindow.OpenDeal` (`ui/shop_window.go:92`) opens the buy/sell choice
prompt on `ZC_SELECT_DEALTYPE`; the player's choice sends
`SendShopDealSelection` (`ui/shop_window.go:1126`). The resulting
`ZC_PC_SELL_ITEMLIST`/`ZC_PC_PURCHASE_ITEMLIST` opens the corresponding cart
UI (`OpenSell`/`OpenBuy`, `ui/shop_window.go:97,120`); confirming that cart
sends the whole selected item+amount list in one packet
(`SendShopSellItems`/`SendShopBuyItems`, `ui/shop_window.go:1035,1058`) —
note this is a **batch** send, not one packet per item, unlike trade (§6)
which adds items one at a time. `applyShopWindowResult`-equivalent handling
of `ZC_PC_PURCHASE_RESULT`/`ZC_PC_SELL_RESULT` (`IN-PACKET-DETAILS.MD` §9)
closes the loop; a successful sell additionally posts a blue "deal
completed" console message. `ui/shop_window.go:1103,1107` also shows the
window can send an *empty* buy/sell list to explicitly cancel the deal
rather than confirming it — same opcodes, zero-length payload.

---

## 6. Trade

`IN-PACKET-DETAILS.MD` §7 and `OUT-PACKET-DETAILS.MD` §8 already table out
every packet in this state machine; this section is the causal narrative
that ties the two sides together into one sequence, since trade genuinely
is a back-and-forth between two live clients relayed through the server (see
§7 for why that matters).

```
A → server   CZ_REQ_EXCHANGE_ITEM        SendTradeRequest          — A right-clicks B, "Trade"
server → B   ZC_REQ_EXCHANGE_ITEM        ParseTradeRequest         — opens B's incoming-request dialog
B → server   CZ_ACK_EXCHANGE_ITEM        SendTradeAck              — B accepts or declines
server → A   ZC_ACK_EXCHANGE_ITEM        ParseTradeResponse        — A's request result
  (if accepted, both windows are now open)
A → server   CZ_ADD_EXCHANGE_ITEM        SendTradeAddItem  ─┐
server → A   ZC_ACK_ADD_EXCHANGE_ITEM    ParseTradeAddItemAck │ repeats once per item
server → B   ZC_ADD_EXCHANGE_ITEM        ParseTradeItem      ─┘ either side adds, any number of times
A → server   CZ_CONCLUDE_EXCHANGE_ITEM   SendTradeConclude         — A clicks "Trade" (locks in)
server → both ZC_CONCLUDE_EXCHANGE_ITEM  ParseTradeConclude        — both windows show who locked in
  (once BOTH sides have concluded)
A/B → server CZ_EXEC_EXCHANGE_ITEM       SendTradeCommit           — either side clicks "OK"
server → both ZC_EXEC_EXCHANGE_ITEM      ParseTradeExec            — items actually change hands
```

`game/trade.go:20` sends the initial request on right-click; `:41,51` send
the accept/decline reply from the incoming-request dialog. Every item drag
into the trade window sends one `CZ_ADD_EXCHANGE_ITEM`
(`ui/trade_window.go:107,314`) — the server's per-add ack
(`ZC_ACK_ADD_EXCHANGE_ITEM`, confirming *your own* addition landed) and the
other party's mirrored `ZC_ADD_EXCHANGE_ITEM` (their addition, as seen by
you) are two different incoming packets, which is why
`IN-PACKET-DETAILS.MD` §7 lists both `AddOwnItemAck` and `AddReceivedItem`
as separate handlers — the trade window UI needs to distinguish "an item I
just added really landed" from "the other person added something" even
though both ultimately just add a row to a list.

`SetConcluded(ctx, tradeConclude.Other)` (`ui/trade_window.go`,
`IN-PACKET-DETAILS.MD` §7) is the detail worth internalizing: **conclude is
not a two-way handshake on its own** — either side can conclude
independently, and the `Other` boolean on the incoming ack is what tells
each client's UI whether it was *their own* conclude being echoed back or
the *other party's*. The actual trade only executes once both sides have
independently concluded and one of them additionally sends
`CZ_EXEC_EXCHANGE_ITEM` — conclude alone does not execute the trade. Either
side can cancel (`CZ_CANCEL_EXCHANGE_ITEM`/`ZC_CANCEL_EXCHANGE_ITEM`) or
undo a not-yet-executed conclude
(`CZ_EXCHANGE_ITEM_UNDO`/`ZC_EXCHANGE_ITEM_UNDO`) at any point before
`ZC_EXEC_EXCHANGE_ITEM` actually lands.

---

## 7. Party invite and guild invite (three-party flows)

Both of these are structurally identical, and both are **not** a simple
client↔server request/response — they're a three-hop relay through a second
player's own client:

```
A → server        CZ_PARTY_JOIN_REQ         SendPartyInvite       — A invites B by name/ID
server → B         ZC_REQ_JOIN_GROUP         ParsePartyInviteRequest — B's own client shows a prompt
B → server         CZ_PARTY_JOIN_REQ_ACK     SendPartyInviteAck    — B's own client sends accept/decline
server → A         ZC_PARTY_JOIN_REQ_ACK     ParsePartyInviteAnswer  — A finds out the result
server → both      ZC_ADD_MEMBER_TO_GROUP / ZC_GROUP_LIST           — roster sync, once accepted
```

The middle two hops happen entirely on player B's machine, running the exact
same client binary A is running, reacting to a packet the server relayed
from A's original request. `openPartyInviteRequest` (`game/party.go:107`)
is what runs on B's client — it opens the incoming-invite dialog and wires
`SendPartyInviteAck(request.RequestID, true/false)` to the accept/decline
buttons; `handlePartyInviteAnswer` (`game/party.go:144`) is what runs back
on A's client once B's answer round-trips through the server, mapping the
numeric `Answer` code to one of nine possible console messages (already
joined a party, denied, joined successfully, party full, same-account
character already in party, blocked invitations, not online, map
restrictions on either end). None of this involves A's and B's clients
talking to each other directly — the server is the only thing either client
ever has a socket open to; "three-party" describes the *logical* shape of
who's involved, not a peer-to-peer connection.

Guild invites (`game/guild.go`) are the same shape with different opcodes:
`SendGuildInvite` (`:157`) → relayed as `ZC_REQ_JOIN_GUILD` → the invitee's
`openGuildInviteRequest` (`game/guild.go:165`) sends
`SendGuildInviteReply` (`:172,181`) → relayed back to the inviter as
`ZC_ACK_REQ_JOIN_GUILD` → `handleGuildInviteAck` (`game/guild.go:215`) maps
the result to one of four messages (already in a guild, invitation
rejected, accepted, guild full). See `IN-PACKET-DETAILS.MD` §6.1/§6.4 and
`OUT-PACKET-DETAILS.MD` §7.1 for the full packet names on both sides.

Friend requests (`IN-PACKET-DETAILS.MD` §4 / `OUT-PACKET-DETAILS.MD` §5) and
guild alliance proposals (§6.3/§7.3) follow this identical three-hop shape
too — the pattern generalizes to essentially every "ask another specific
player something" feature in the game, not just party/guild.

---

## 8. Storage password gate

```
client → server (implicit)  — opening storage (e.g. via an NPC, or an item)
server → client   ZC_REQ_STORE_PASSWORD     ParseStoragePasswordPrompt  (State: not-set / set / locked)
client → server   CZ_ACK_STORE_PASSWORD     SendStoragePasswordReply
server → client   ZC_RESULT_STORE_PASSWORD  ParseStoragePasswordResult
  (on success) →   storage window opens; ZC_STORE_NORMAL_ITEMLIST/etc. follow (§ storage/cart, IN-PACKET-DETAILS.MD §11)
```

This flow is entirely a small, self-contained state machine in
`game/storage_password.go` — genuinely one of the cleanest examples in the
codebase of an incoming-packet handler and its outgoing reply living
side-by-side. `handleStoragePasswordPrompt` (`:14`) sets
`ctx.Session.Storage.PasswordPending = true` and, depending on
`prompt.State`, opens one of three prompt UI variants (set / enter / locked)
— `network.StoragePasswordNotSet`/`Set`/`Locked`. The player's submission
goes through `sendStoragePasswordReply` (`:84`), which is a thin wrapper
around `SendStoragePasswordReply(replyType, password, newPassword)` —
`replyType` distinguishes "verifying an existing password" from "setting a
new one," matching the two different prompt UI modes.

`handleStoragePasswordResult` (`:35`) is guarded by
`ctx.Session.Storage.PasswordPending` — if that flag isn't set, an incoming
result is logged and ignored (`glog.Debugf("ignored storage password
result...")`) rather than acted on, since it can't correspond to a request
this client actually made. On `StoragePasswordCheckSucceeded` it clears the
pending flag, sets `Storage.Open = true`, and opens the storage window
directly — the **next** incoming `ZC_STORE_NORMAL_ITEMLIST`/etc. (which the
server sends unprompted once storage access is granted) is what actually
populates it. On `StoragePasswordCheckFailed` it re-prompts with a
remaining-attempts count computed from `result.ErrorCount`
(`storagePasswordMaximumFailures = 3`); on
`StoragePasswordTooManyFailures` it switches to the locked-state prompt.
There is no automatic retry here — every failure requires the player to
re-type a password.

---

## 9. Pet capture

```
client → server   CZ_USE_ITEM               SendUseInventoryItem   — use a Pet Capture item
server → client    ZC_START_CAPTURE          ParsePetCaptureStart   — capture minigame begins
  (client shows the "select a monster" prompt, then the slot-machine UI once a target is clicked)
client → server   CZ_TRYCAPTURE_MONSTER     SendTryCaptureMonster
server → client    ZC_TRYCAPTURE_MONSTER     ParsePetCaptureResult  — success or failure
```

The capture item itself is used through the ordinary item-use flow (not
covered separately in this document — see `IN-PACKET-DETAILS.MD` §10's
`ZC_USE_ITEM_ACK`); what's specific to pet capture is what the server's
`ZC_START_CAPTURE` reply triggers. `startPetCapture`
(`game/pet.go:99`) doesn't send anything itself — it just clears any other
pending skill/pickup target and sets `pendingPetCapture.active = true` with
a console prompt ("Select a monster to capture"). The player's subsequent
click on an eligible monster is what actually reaches
`openPetSlotMachine` (`game/pet_slot_machine.go:47`), which — if a slot
machine visual asset can't be loaded, or UI is disabled
(`ctx.Config.Render.NoUI`) — sends `CZ_TRYCAPTURE_MONSTER` immediately;
otherwise it opens the slot-machine minigame animation and waits for the
player's own click on the spinning display
(`updatePetSlotMachine`, `game/pet_slot_machine.go:69`) before sending. So
the *visual* capture attempt (spinning slots) is purely a local animation
gate in front of the real network request — the actual capture odds are
determined server-side by whatever `ZC_TRYCAPTURE_MONSTER`'s result carries,
not by anything client-visible about where the slots land.

`applyPetCaptureResult` (`game/pet.go:151`) branches on whether the
slot-machine UI is active: if so, it feeds the result into the slot machine
(`m.petSlotMachine.setResult(result.Success)`) so the spinning animation
resolves to match the real outcome; if the slot machine wasn't shown (the
no-UI/asset-missing path above), it just posts a plain success/failure
console message directly.

---

## 10. Vending

Two independent flows share the vending packet family: setting up your own
store, and buying from someone else's.

### Selling (setting up a store)

```
client → server   CZ_REQ_OPENSTORE2          SendOpenVendingStore   — confirm the setup dialog
server → other clients  ZC_STORE_ENTRY        ParseVendingBoard      — your sign appears on their screens
  ... other players browse and buy; each purchase is invisible to you except: ...
server → client   ZC_DELETEITEM_FROM_MCSTORE  ParseVendingSoldItem   — one of your items just sold
client → server   CZ_REQ_CLOSESTORE          SendCloseVendingStore  — you close the store
```

`ZC_OPENSTORE` (`IN-PACKET-DETAILS.MD` §9) is the server's initial prompt
that opens the setup dialog in the first place (this is presumably itself
triggered by using a Vending shop-license item, following the same
item-use-triggers-a-server-prompt shape as pet capture in §9, though this
codebase's catalog doesn't show that specific item-use link explicitly).
Confirming the dialog sends the item list + store name
(`ui/vending_window.go:770`); `SendCancelVendingStoreOpen`
(`ui/vending_window.go:813`) cancels it — same opcode, empty payload, per
`OUT-PACKET-DETAILS.MD` §10's note that these are built by two separate
helpers rather than one method overloaded on a nil-items case. Once open,
`applyVendingBoardToWorld` (`game/vending.go:15`) marks
`ctx.World.Player.Vending = true` so the local player's own sprite/nameplate
reflects the vending state; each sale streams back as
`ZC_DELETEITEM_FROM_MCSTORE` with no outbound trigger at all — it's a pure
server-initiated notification of something another player did.

### Buying (browsing someone else's store)

```
client → server   CZ_REQ_BUY_FROMMC                  SendVendingListRequest(sellerID)  — click their sign
server → client    ZC_PC_PURCHASE_ITEMLIST_FROMMC     ParseVendingItemList (Own: false)
client → server   CZ_PC_PURCHASE_ITEMLIST_FROMMC     SendVendingPurchase              — confirm your cart
server → client    ZC_PC_PURCHASE_RESULT_FROMMC       ParseVendingPurchaseResult
```

`requestVendingList` (`game/vending.go:59`) sends the list request when the
player clicks another player's vending sign; `ui/vending_window.go:792`
sends the actual purchase once the buyer confirms their selection —
this, like the NPC shop buy/sell flow in §5, is a single batch send of the
whole chosen cart, not one packet per item, unlike trade's one-item-per-send
shape in §6.

---

## 11. Item minigames (identify, and the rest)

All five of `IN-PACKET-DETAILS.MD` §10's "list → choose → ack" minigames
(identify, card composition, repair, weapon refine, item creation/arrow
crafting) share one shape. Identify is the simplest, so it's the
fully-worked example:

```
server → client   ZC_ITEMIDENTIFY_LIST    ParseItemIdentifyList   — list of identifiable items opens
client → server   CZ_REQ_ITEMIDENTIFY     SendItemIdentify(index) — player picks one
server → client   ZC_ACK_ITEMIDENTIFY     ParseItemIdentifyAck    — success/failure for that item
```

`ui/identify_window.go:221,234` sends the choice once the player clicks an
item in the list (`OUT-PACKET-DETAILS.MD` §11); the ack
(`IN-PACKET-DETAILS.MD` §10) both applies the identified flag to the
session's inventory (`applyItemIdentifyAck`) and re-clamps the inventory
bag's scroll offset (`ClampScroll`) since the item's display name/icon
changing can change the bag's effective row count. A rejected identify
(`identifyAck.Success == false`) just logs a warning — there's no retry, the
list stays open (or is re-sent by the server) for the player to try a
different item.

The other four minigames differ only in cosmetic detail, all documented
side-by-side in `IN-PACKET-DETAILS.MD` §10's table and
`OUT-PACKET-DETAILS.MD` §11:

- **Card composition** is the only one that's a *pair* of outbound sends
  rather than one: `CZ_REQ_ITEMCOMPOSITION_LIST` first requests "which
  equipment slots can this specific card go into" (opened from the
  inventory bag's right-click menu, `ui/inventory_bag_window.go:441`), and
  only after that list comes back does `CZ_REQ_ITEMCOMPOSITION` send the
  actual card+equipment pairing (`ui/card_composition_window.go:142`).
- **Repair** and **weapon refine** both use the same
  `ui/equipment_choice_window.go` list widget for two different underlying
  minigames (`:279,289` for repair, `:322,332` for refine) — same UI
  component, different `Send*` call at the confirm step.
- **Item creation** (potions etc.) is the only one where the outbound packet
  carries more than a single index — `SendMakingItem`
  (`ui/making_item_window.go:183,193`) sends a recipe ID *plus* up to three
  material item IDs the player selected to substitute in, not just "which
  list row."
- **Arrow crafting** has no separate ack packet on the receiving side at
  all (`IN-PACKET-DETAILS.MD` §10 notes this) — the result is presumably
  folded into the ordinary inventory-update packets rather than a dedicated
  ack, which this document can't confirm further without server source.

---

## 12. Map transition / warp

This is the flow with the most branches, because "the player ends up on a
different map" can be triggered by the player (a remembered warp point, a
teleport skill), by a monster, or purely by the server (a script-driven
warp, a portal walked into) — but every one of those causes converges on
the same two incoming packets and the same client-side state machine.

### The incoming side: fade → reconnect-if-needed → confirm

```
server → client   ZC_MAPMOVE / ZC_CHANGEMAPSERVER   ParseMapChange
  (client starts a fade-to-black; this does NOT immediately process the change)
  ... several frames of fade animation ...
  (fade completes; NOW the client processes the map change)
[same map server]      → client rebuilds/reuses the map, no reconnect
[different map server] → client reconnects TCP, re-sends CZ_ENTER2, waits
server → client   ZC_ACCEPT_ENTER / ZC_ACCEPT_ENTER3   ParseMapAcceptEnter
  (client fades back in; if it was a cross-server warp, the new WorldMode is built HERE, not earlier)
```

`ZC_MAPMOVE`/`ZC_CHANGEMAPSERVER` (`ParseMapChange`,
`IN-PACKET-DETAILS.MD` §1) doesn't apply the map change synchronously —
it calls `m.startMapFadeOut(change, time.Now())` (`game/map_fade.go:11`)
and returns `(nil, true)`, which stops packet processing for the rest of
that network read (with any remaining packets in the same batch deferred to
the next `WorldMode`/frame, per `handleNetworkPackets`,
`game/world_packets.go:17-48` — this is why a mail ack arriving in the same
read as a map change isn't silently dropped). The actual map-change logic,
`handleMapChange` (`game/world.go:1387`), only runs once the fade-out
animation has visually completed and held for a frame
(`WorldMode.Update`, `game/world.go:646-661`):

- **Same map, reload avoided** (`reuseLoadedMap`,
  `!change.ServerMove && sameLoadedMap(...)`, `game/world.go:1409,1416`) —
  the client resets the player position and immediately sends
  `CZ_NOTIFY_ACTORINIT` (`SendLoadEndAck`) without reconnecting or reloading
  scenery at all, since the GND/GAT/RSW/textures are already resident. This
  is the fast path used for e.g. server-side "reset player to a fixed spot
  on this same map" scripts.
- **Different map, same server** — the function falls through to
  `return m.nextWorldMode()` (`game/world.go:1448`), building a fresh
  `WorldMode` immediately (no reconnect needed), which then drives the
  normal scene-loading path and sends `CZ_NOTIFY_ACTORINIT` once loading
  finishes (`game/world.go:562`, per `OUT-PACKET-DETAILS.MD` §2).
- **Different map server** (`change.ServerMove`, `game/world.go:1426`) —
  the client tears down mail/deferred-packet state, updates
  `ctx.Session.Zone` to the new address, opens a fresh TCP connection
  (`ctx.Network.Connect`, 5-second timeout), and re-sends `CZ_ENTER2`
  (`SendMapServerEnter`) on that new connection. **It does not build a new
  `WorldMode` yet** — it sets `m.pendingWarp = true` and returns `nil`,
  which per `WorldMode.Update`'s fade-in guard (`if !m.pendingWarp {
  m.startMapFadeIn(now) }`, `game/world.go:657`) also means the screen
  **stays black** rather than fading back in prematurely. A reconnect
  failure at this point opens the "connection failed"/disconnect dialog
  instead of silently hanging (`game/world.go:1435-1437,1440-1443`).

`ZC_ACCEPT_ENTER`/`ZC_ACCEPT_ENTER3` (`ParseMapAcceptEnter`,
`IN-PACKET-DETAILS.MD` §1) is what finally clears `m.pendingWarp` and — only
in the cross-server-warp case — builds the actual new `WorldMode` via
`m.nextWorldMode()` at that point (`game/world_packets.go:247-249`); for the
same-server and same-map cases, `WorldMode` was already built or reused
earlier, so this ack for those paths just applies spawn position/tick and
re-asserts the less-effects preference (`sendLessEffectPreference`) without
building anything new.

### What triggers the outbound side

None of the above is initiated by a single outbound packet — `ZC_MAPMOVE`
can be the server's response to several different unrelated triggers:

- **Remembered warp point**: `SendSelectWarpPoint`
  (`ui/teleport_modal.go:112,131`, `game/teleport_handlers.go:78`) picks a
  destination from a list the server sent via `ZC_WARPLIST`
  (`IN-PACKET-DETAILS.MD` §16), itself granted by casting a Warp
  Portal-family skill (§3's skill-cast flow) or the equivalent NPC/console
  flow.
- **Level-1 Teleport**: as noted in §3, `SendSelectWarpPoint` is sent
  automatically, immediately after the skill-use packet, with no
  `ZC_WARPLIST` round-trip at all — this codebase's one instance of an
  outbound packet firing in direct response to *another outbound send*
  rather than an incoming packet.
- **NPC-scripted warp**: reached entirely through the NPC dialog loop in §5
  — from the client's perspective this looks identical to any other
  NPC-driven server action; the client sends nothing warp-specific at all,
  it's simply what a `CZ_REQ_NEXT_SCRIPT`/menu-choice send happens to
  trigger server-side.
- **Server/monster-initiated** (a script-forced warp, being knocked into a
  portal, etc.) — the client sends nothing preceding `ZC_MAPMOVE` at all in
  these cases; this is inferred from there being no outbound packet in
  `OUT-PACKET-DETAILS.MD`'s catalog that obviously precedes every possible
  map-move trigger, not confirmed against server logic.

---

## 13. Login → character select → map enter

`OUT-PACKET-DETAILS.MD` §1 already walks this handshake step-by-step from
the outbound side; this section adds the piece that document is scoped to
skip — the **incoming** replies interleaved with each step, and every
failure branch. All of it happens in `game/login.go`, in one big
`for _, pkt := range ctx.Network.DrainPackets()` loop
(`game/login.go:215`) that, unlike `WorldMode.handleNetworkPacket`, is not
split into a separate file/function per concern.

```
1.  client → login server   connect (TCP)
2.  A → server    CA_LOGIN               SendAccountLogin
    success:  server → A   AC_ACCEPT_LOGIN (0x0069)   ParseAccountAcceptLogin → char-server list
    failure:  server → A   AC_REFUSE_LOGIN (0x006A)   ParseAccountRefuseLogin → login refused, socket closed

3.  client → char server     connect (TCP)
4.  A → server    CA_ENTER               SendCharServerEnter
    success:  server → A   HC_ACCEPT_ENTER (0x006B)   ParseCharList → character list
    (server can also reject a later selection without disconnecting — see step 5)

5a. A → server    CH_SELECT_CHAR         SendSelectCharacter(slot)
    reject:   server → A   HC_REFUSE_ENTER (0x006C)   — selection refused, char list stays open
5b. A → server    CH_MAKE_CHAR           SendMakeCharacter
    success:  server → A   HC_ACCEPT_MAKECHAR (0x006D)  ParseMakeCharacterAccept
    failure:  server → A   HC_REFUSE_MAKECHAR (0x006E)  ParseMakeCharacterRefuse
5c. A → server    CH_DELETE_CHAR         SendDeleteCharacter
    success:  server → A   HC_ACCEPT_DELETECHAR (0x006F) ParseDeleteCharacterAccept
    failure:  server → A   HC_REFUSE_DELETECHAR (0x0070) ParseDeleteCharacterRefuse

    (successful character selection continues to:)
    server → A   HC_NOTIFY_ZONESVR (0x0071)   ParseZoneServerNotify → map server address

6.  client → map server      connect (TCP)
7.  A → server    CZ_ENTER2              SendMapServerEnter  (starts map keepalive, ARCHITECTURE.md "Networking")
8.  ... normal in-game packet flow begins; see IN/OUT-PACKET-DETAILS.MD from here ...
```

### Failure branches, by incoming packet

- **`AC_REFUSE_LOGIN` (0x006A)** — `applyAccountRefuseLogin`
  (`game/login_refuse.go:46`) closes the socket outright
  (`ctx.Network.Close()`) and opens a modal alert; the specific message is
  looked up from `refusal.ErrorCode` against a table of 20 known codes
  (`game/login_refuse.go:12-35`) covering everything from wrong
  password/unregistered ID through several ban/lock variants to "server is
  jammed due to overpopulation" (code 7) — this is the client's only signal
  for a full server (there's no separate "server full" packet, it's folded
  into the generic login-refusal code space). Codes 6 and 100 additionally
  interpolate a `%s` placeholder with `refusal.UnblockTime` from the packet.
- **`HC_REFUSE_ENTER` (0x006C)** — handled inline at
  `game/login.go:222-226` with no dedicated function: just clears
  `m.charSelectPending` and continues the loop. Notably this does **not**
  close the connection or show a dialog — the character list stays open and
  the player can pick a different slot, distinguishing it from the
  login-level refusal above which is fatal to the whole session.
- **`HC_REFUSE_MAKECHAR` (0x006E)** — `describeMakeCharacterRefuse` maps the
  code to a status-line message (not a modal); the character-creation form
  stays open for another attempt.
- **`HC_REFUSE_DELETECHAR` (0x0070)** — `applyDeleteCharacterRefuse`
  (`game/login.go:369`) similarly surfaces the failure without navigating
  away from character select.

### What isn't shown above

`SendLoginServerKeepalive`/`SendPing` (`OUT-PACKET-DETAILS.MD` §1) run on
independent timers alongside this whole sequence while the client is
sitting at the login/character screens, not as part of the request/response
chain itself — they exist purely to stop the server from timing out an idle
connection during however long a human takes to type credentials or browse
characters, and have no corresponding "pong" this document's catalogs list
(no incoming ack for either).

---

*See `ARCHITECTURE.md` for the general dispatch machinery these flows all
build on, and `IN-PACKET-DETAILS.MD`/`OUT-PACKET-DETAILS.MD` for the
complete packet-by-packet reference each flow above only partially quotes.*
