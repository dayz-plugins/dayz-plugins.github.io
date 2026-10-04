---
name: dayz-engine-events
description: >-
  DayZ 1.29.163709 event system: EventType is a C++ RTTI pointer, how events are raised and dispatched into script, the full event table with payloads, how chat messages arrive and are sent, how mod RPCs travel, and the session natives for connect, disconnect and exit.
game_build: DayZ 1.29.163709 (DayZ_x64.exe, PE timestamp 0x6A72FC58)
created: 2026-10-04T20:05+0200
last_edited: 2026-10-04T20:05+0200
---

# Events, chat and RPC (DayZ 1.29.163709)

Three questions that turn out to have one answer: how a mod hears about **what the game is
doing** (loaded, connecting, kicked, spawned), how it hears about **chat**, and how it
exchanges **its own messages with a server mod**. All three run through the engine's event
system, and the parts a native plugin would hook are a handful of functions.

Evidence classes used below: **decompiled** (Ghidra output only), **verified** (observed at
runtime), and **from the shipped scripts** — `dta/scripts.pbo`, which is the game's own
Enforce source and is therefore stronger than a decompile but says nothing about addresses.

## The event system

### `EventType` is not a number

Script declares every event id as `const EventType XEventTypeID;` with no value, and the
engine fills them in at registration (`0x5B77C0`, decompiled), which is also where the
`CT_*`, `PROGRESS_*` and chat-channel constants are set. Each one is assigned **the address
of the event class's MSVC RTTI type descriptor**:

```c
puVar2 = FUN_14031b690(ctx,"ChatMessageEventTypeID",0x40);
*puVar2 = &game::ChatMessageEvent::RTTI_Type_Descriptor;
```

So `EventType` is a pointer to a C++ type, not an enum, and comparing event ids in script is
comparing type identities. This matters for native code in two ways: the ids are **not stable
numbers to hardcode** (they are addresses, so they move with every build), and an event object
can be identified from its own vtable without any table of ids at all — see below.

### An event object knows its own type and name

Every event class has the same six-slot vtable (`game::ChatMessageEvent::vftable` at
`0xCE73F0`, `game::MPConnectionCloseEvent::vftable` at `0xCE72F0`, decompiled):

| Slot | Offset | What it is |
| --- | --- | --- |
| 0 | `+0x00` | Destructor |
| 1 | `+0x08` | Returns the event's `EventType` — the RTTI descriptor pointer above |
| 2 | `+0x10` | Returns the class name as a plain string, e.g. `"ChatMessageEvent"` |
| 3, 4 | `+0x18`, `+0x20` | Null (`_guard_check_icall`) on both classes inspected |
| 5 | `+0x28` | `enf::BaseItem` teardown |

Slot 1 and slot 2 are each 14 bytes and do nothing but write a constant into the out-parameter
(`0x71B4D0` and `0x71B5A0` for `ChatMessageEvent`). **Slot 2 is the useful one for a plugin**:
calling it on any event object yields the event's name as text, with no address table and no
RTTI parsing, which means a hook can name events it has never heard of — including the ones
script cannot see at all (`ReloadShadersEvent`, `SetPausedEvent`, `CrashLogEvent` and the rest
of the list the shipped scripts enumerate in a comment but do not declare).

### Raising and dispatching

```
    raiser (e.g. 0x719B90 for chat)
      builds the event on its stack, vftable + fields
        -> FUN_1401D6260(FUN_1401D2BD0(), &event)        raise on the event manager
             -> ... -> FUN_1405CACD0(game, &event)       dispatch into script
                         slot 1 -> EventType
                         event[1] -> passed through as script's `Param` (see below)
                         -> script CGame.OnEvent(type, params)
                         if script returned non-zero: stop
                         else FUN_1405CACB0(game)        the engine's own handling
```

| What | RVA | Evidence |
| --- | --- | --- |
| Event manager getter | `0x1D2BD0` | decompiled, called by both raisers inspected |
| Raise an event | `0x1D6260` | decompiled, `(manager, &event)` |
| Dispatch into script `OnEvent` | `0x5CACD0` | decompiled |
| The engine's handling when script did not take it | `0x5CACB0` | decompiled |
| `EventType` registration (all ids) | `0x5B77C0` | decompiled |

`0x5CACD0` is small enough to quote in full, and it is the whole contract:

```c
if (DAT_140f226a0 == -1)
    DAT_140f226a0 = FUN_140319210(*(undefined8 *)(param_1 + -0x20),"OnEvent");
lVar2  = param_2[1];                                  // second qword -> script arg 2
puVar3 = (undefined8 *)(**(code **)(*param_2 + 8))(param_2);   // vtable slot 1: EventType
puVar3 = FUN_140317060(param_1 + -0x28,out,DAT_140f226a0,*puVar3,lVar2,0,0);  // call script
if (*(int *)*puVar3 != 0) return 1;                   // script handled it
return FUN_1405cacb0(param_1,param_2);                // engine default
```

The `EventType` half is settled: slot 1 of the event's own vtable, and nothing else. The
payload half is **not**. This dispatcher passes the event's second qword straight through as
script's `Param params`, but the chat raiser below stores a refcounted pointer there rather
than an obvious `Param` object, so either the event classes share a base that holds a `Param`
at `+0x08` or chat reaches script by another route. The third reference to the `"OnEvent"`
string is a 4936-byte function at `0x5B8560` that has not been read yet and is the likeliest
place for a per-event-class marshaller. Until that is traced, **a plugin hooking `0x5CACD0`
can name an event reliably and read its payload only speculatively.**

Two hook points follow, with different reach:

- **`0x1D6260`** sees *every* event the engine raises, including those that never reach
  script. Best for a listener.
- **`0x5CACD0`** sees only events on their way to script, and **its return value decides
  whether the engine still handles the event**. Returning 1 without calling the original is
  how a plugin swallows one.

### The events script can see

From the shipped scripts (`3_Game/gameplay.c`), with their payload types. The ones a plugin
would want for a rich lifecycle are marked.

| Event | Payload | Notes |
| --- | --- | --- |
| `StartupEventTypeID` | — | ★ engine up |
| `WorldCleaupEventTypeID` | — | |
| `ConnectingStartEventTypeID` | — | ★ a connection attempt begins |
| `ConnectingAbortEventTypeID` | — | ★ the player cancelled it |
| `MPSessionStartEventTypeID` | — | ★ session accepted |
| `MPSessionPlayerReadyEventTypeID` | — | ★ **in game**: script sets `DayZGameState.IN_GAME` here |
| `MPSessionEndEventTypeID` | — | ★ back to the main menu |
| `MPSessionFailEventTypeID` | — | ★ the connection failed |
| `MPConnectionLostEventTypeID` | `Param1<int>` duration | ★ |
| `MPConnectionCloseEventTypeID` | `Param2<int, string>` — `EClientKicked`, extra text | ★ **this is "we were kicked", with the reason** |
| `ChatMessageEventTypeID` | `Param4<int, string, string, string>` — channel, from, text, colour class | ★ see below |
| `ChatChannelEventTypeID` | `Param1<int>` channel | |
| `ClientConnectedEventTypeID` | `Param2<string, string>` name, uid | server side |
| `ClientPrepareEventTypeID` | `Param5<PlayerIdentity, bool, vector, float, int>` | server side; the `vector` is the spawn position |
| `ClientNewEventTypeID` | `Param3<PlayerIdentity, vector, Serializer>` | server side |
| `ClientNewReadyEventTypeID` / `ClientReadyEventTypeID` / `ClientRespawnEventTypeID` / `ClientReconnectEventTypeID` | `Param2<PlayerIdentity, Man>` | server side |
| `ClientDisconnectedEventTypeID` | `Param4<PlayerIdentity, Man, int, bool>` | server side; last two are logout time and auth-failed |
| `ClientRemovedEventTypeID` | — | server side |
| `PlayerDeathEventTypeID` | `Param2<DayZPlayer, Object>` | ★ the second is not reliably the killer on a client |
| `RespawnEventTypeID` | `Param1<int>` time | ★ |
| `PreloadEventTypeID` | `Param1<vector>` position | ★ **where we are about to spawn** |
| `LoginTimeEventTypeID` / `LogoutEventTypeID` / `LogoutCancelEventTypeID` | `Param1<int>` / `Param1<Man>` | ★ the login and logout queues |
| `LoginStatusEventTypeID` | `Param2<string, string>` two message lines | ★ what the loading screen says |
| `ProgressEventTypeID` | `Param3<int, float, string>` state, progress, title | ★ loading progress |
| `ConnectivityStatsUpdatedEventTypeID` | `Param1<PlayerIdentity>` | |
| `ServerFpsStatsUpdatedEventTypeID` | `Param4<float, float, int, int>` | ★ server fps and skipped steps |
| `NetworkInputBufferEventTypeID` | `Param1<bool>` isFull | |
| `VONStateEventTypeID` | `Param2<bool, bool>` listening, toggled | |
| `VONStartSpeakingEventTypeID` / `VONStopSpeakingEventTypeID` | `Param2<string, string>` name, id | ★ who is talking |
| `VONUserStartedTransmittingAudioEventTypeID` / `…Stopped…` | — | |
| `NetworkManagerClientEventTypeID` / `NetworkManagerServerEventTypeID` | — | |
| `DialogQueuedEventTypeID` | — | |
| `ScriptLogEventTypeID` | `Param1<string>` | ★ script `Print` output |
| `SelectedUserChangedEventTypeID` | — | console platforms |
| `PartyChatStatusChangedEventTypeID` | — | console platforms |
| `DLCOwnerShipFailedEventTypeID` | `Param1<string>` world | |
| `SetFreeCameraEventTypeID` | `Param1<FreeDebugCamera>` | |
| `WindowsResizeEventTypeID` | `Param3<int, int, bool>` | |

`EClientKicked`, the first member of the kick payload, is a plain enum in the shipped scripts
(`3_Game/Global/ErrorModuleHandler/ClientKickedModule.c`): `UNKNOWN = -1`, `OK = 0`, then
`SERVER_EXIT`, `KICK_ALL_ADMIN`, `KICK_ALL_SERVER`, `TIMEOUT`, `LOGOUT`, `KICK`, `BAN`,
`PING`, `MODIFIED_DATA`, `UNSTABLE_NETWORK`, `SERVER_SHUTDOWN`, `NOT_WHITELISTED`,
`NO_IDENTITY`, `NO_INPUT_INTERFACE`, `INVALID_UID`, `BANK_COUNT`, `ADMIN_KICK`, `INVALID_ID`,
`INPUT_HACK`, `QUIT`, `LEAVE`, …

The server a client is connected to does not arrive with the kick event; it is asked for
separately with `GetHostAddress`/`GetHostName`/`GetHostData` (see the native table below),
which a plugin can call at any time and should sample when `ConnectingStartEventTypeID`
fires, because by the time the kick arrives the session is already going away.

## Chat

### Incoming

Every chat line the client displays is raised by **`0x719B90`** (decompiled), which builds a
`ChatMessageEvent` on its stack and raises it through the usual path:

```c
FUN_140719b90(game, int channel, RefString *from, RefString *text, RefString *colourClass)
```

The three string arguments are Enfusion refcounted blocks — a pointer whose first `int` is the
refcount, which the function increments into the event and decrements on the way out. The
event it builds holds its vtable, those three pointers and the channel; the exact offsets are
not given here, because the decompiler's stack-local names are not field offsets and reading
them as such is how the first draft of this file got them wrong.

A hook here sees every message from every source — other players, the server, admin messages,
BattlEye, and the client's own `Chat`/`ChatPlayer` calls — and **returning without calling the
original swallows the line entirely**: it never becomes an event, so neither the chat widget
nor any script mod sees it. That is a stronger swallow than the `0x5CACD0` one, which only
stops the script half.

Channels, from the shipped scripts (`5_Mission/GUI/Chat/Chat.c`), are a bit field:

| Value | Channel |
| --- | --- |
| 1 | `CCSystem` |
| 2 | `CCAdmin` |
| 4 | `CCDirect` |
| 8 | `CCMegaphone` |
| 16 | `CCTransmitter` |
| 32 | `CCPublicAddressSystem` |
| 64 | `CCBattlEye` |

### Outgoing

| Script method | RVA | What it does |
| --- | --- | --- |
| `Chat(string text, string colorClass)` | `0x5AD300` | Local only: puts a line in this client's chat. What item notifications use. |
| `ChatMP(Man recipient, string text, string colorClass)` | `0x5AD3D0` | Server side: send to one player. |
| `ChatPlayer(string text)` | `0x5AD4B0` | **Says it as the player**, on the current channel, to the server. |

All three are decompiled as ordinary member functions — see *Calling a native* below.

The shipped scripts show the send path end to end: `ChatInputMenu.OnChange` takes the edit
box text and calls `g_Game.ChatPlayer(text)`; in single player, where there is no server to
echo it back, the script additionally constructs a `ChatMessageEventParams(CCDirect, name,
text, "")` and hands it to `MissionGameplay.m_Chat.Add` itself. So in multiplayer the
round trip through the server is what makes your own message appear, which is why a swallow
hook at `0x719B90` catches your own messages too.

## RPC — how a mod talks to its server mod

There is no separate "mod command" channel. A mod sends its own messages with the same four
natives every vanilla subsystem uses, over the game's own connection:

| Script method | RVA |
| --- | --- |
| `RPC(Object target, int rpcType, array<ref Param> params, bool guaranteed, PlayerIdentity recipient = null)` | `0x5BCF70` |
| `RPCSingleParam(Object target, int rpc_type, Param param, bool guaranteed, PlayerIdentity recipient = null)` | `0x5BD000` |
| `RPCSelf(Object target, int rpcType, array<ref Param> params)` | `0x5BCFA0` |
| `RPCSelfSingleParam(Object target, int rpcType, Param param)` | `0x5BCFD0` |

Semantics, from the shipped scripts (`3_Game/Global/Game.c`): called on a client the RPC is
evaluated **on the server**; called on the server it goes to **all clients**, or to one when
`recipient` is given. A null `target` means the RPC is global and `CGame` itself handles it;
a non-null one routes it to that entity's `OnRPC`. `RPCSelf*` is not a network call at all —
it delivers to this process only.

Incoming RPCs reach script through **`0x5BB620`** (decompiled), which has exactly the shape of
the event dispatcher:

```c
if (DAT_140f221a0 == -1)
    DAT_140f221a0 = FUN_140319210(*(undefined8 *)(param_1 + 8),"OnRPC");
FUN_140317060(param_1,out,DAT_140f221a0,sender,target,rpc_type,ctx);
```

so a plugin hooking it sees `(PlayerIdentity sender, Object target, int rpc_type,
ParamsReadContext ctx)` for every RPC the client receives.

`rpc_type` is a plain `int` that both ends simply agree on. Vanilla's are the `ERPCs` enum
(`3_Game/Enums/ERPCs.c`), which starts at `RPC_SYNC_ITEM_VAR = 0` and runs upwards, with
negative values `-1…-4` reserved for the persistence bank. **A mod picks its own numbers and
must stay clear of that range**; the convention in the DayZ modding ecosystem is a large
constant offset. Nothing in the engine registers or validates an id, so an id collision
between two mods is silent and looks like corrupted parameters.

The payload is a `ParamsReadContext` — the engine's serializer — not raw bytes, so a native
plugin that wants to read or write RPC parameters needs that serializer's interface. That
part is **not yet researched**; what is settled is the transport, the dispatch point and the
id space.

## Session control natives

Straight out of the `CGame` registrator at `0x5B9D80` (decompiled), with the signatures the
shipped scripts give:

| Script method | RVA |
| --- | --- |
| `Connect(UIScriptedMenu parent, string IpAddress, int port, string password)` | `0x5AFFA0` |
| `ConnectLastSession(UIScriptedMenu parent, int selectedCharacter = -1)` | `0x5AFB20` |
| `DisconnectSession()` | `0x5B1D10` |
| `DisconnectSessionForce()` | `0x5B1D40` |
| `RequestExit(int code)` | `0x5BCAC0` |
| `RequestRestart(int code)` | `0x5BCB10` |
| `GetHostAddress(out string address, out int port)` | `0x5B3080` |
| `GetHostName(out string name)` | `0x5B3340` |
| `GetHostData()` | `0x5B3260` |
| `GetMainMenuWorld()` | `0x5B35B0` |
| `IsMultiplayer()` | `0x5B6A60` |
| `IsClient()` | `0x5B69F0` |

`RequestExit` does not exit. Its whole body is:

```c
*(undefined4 *)(param_1 + 0xd8) = code;   // remember the exit code
DAT_14426f290 = 1;                        // ask the main loop to stop
```

(after a check that bails to `FUN_140809dc0(5)` on some condition). So the exit is cooperative
and happens at the top of the next frame, which is exactly what a console `quit` wants:
the game shuts down the way it would from its own menu rather than being killed.

`DisconnectSession` is equally thin — it hands the current world name
(`GetMainMenuWorld`, `0x5B35B0`) to `FUN_1406A8A50(this + 0x278, world, 0)`.

## Calling a native

The important structural finding: **`proto native` methods are bound as ordinary member
function pointers**, not as script-stack thunks. `RequestExit` is
`void(CGame *this, int code)`, `Chat` is `void(CGame *this, RefString *text, RefString
*colour)`, `DisconnectSession` is `void(CGame *this)`. The script VM marshals arguments into
the normal x64 convention before calling. Native code can therefore call any of them directly,
given two things:

1. **The `CGame` instance.** Not yet located as a global. The dispatchers above receive it
   (`0x5CACD0` is called with the game object; its script class sits at `this-0x20`), so the
   cheapest route is to capture it from a hook on a function the game calls every frame rather
   than to hunt for the global.
2. **An Enfusion `RefString`** for any string argument, which is a refcounted block the callee
   increments. Constructing one from native code needs that type's allocator; `0x5DC10` is the
   function `ChatPlayer` and `Chat` both call on their string arguments and is the obvious
   place to start.

Both are prerequisites for a loader-side `chat`, `connect` or `quit` command, and neither is
settled yet.

For completeness, the engine's own way into script — what a plugin would use to call a
*script* method rather than a native — is the pair the dispatchers use:

| What | RVA |
| --- | --- |
| Resolve a method name on a script class to an id | `0x319210` |
| Call a script method by id, up to four arguments | `0x317060` |

## Open questions

1. **The `CGame` instance pointer.** Needed before any native above can be called. Capturing
   it from `0x5CACD0` is the cheap route; finding the global is the tidy one.
2. **`RefString` construction** from native code, for every native that takes a string.
3. **`ParamsReadContext`**, without which RPC payloads can be counted but not read.
4. Whether `0x1D6260` is reached by every event or only by the manager's own queue — only two
   raisers have been inspected, and both go through it.
5. **How an event's payload becomes script's `Param`** — see the note under the dispatcher.
   `0x5B8560` is the function to read next, and it blocks reading any event's contents from a
   hook, which is most of what a logging plugin wants.
6. None of this has been **verified at runtime** yet. Every address above is decompiled, and
   every script fact is from the shipped scripts; the first hook installed on any of them is
   what turns them into verified ones.
