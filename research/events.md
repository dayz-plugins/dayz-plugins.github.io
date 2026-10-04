---
name: dayz-engine-events
description: >-
  DayZ 1.29.163709 event system: EventType is a C++ RTTI pointer, how events are raised and dispatched into script, the full event table with payloads, how chat messages arrive and are sent, how mod RPCs travel, and the session natives for connect, disconnect and exit.
game_build: DayZ 1.29.163709 (DayZ_x64.exe, PE timestamp 0x6A72FC58)
created: 2026-10-04T20:05+0200
last_edited: 2026-10-04T20:15+0200
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
it yields the event's name as text, with no address table and no RTTI parsing, which means a
hook can name events it has never heard of — including the ones script cannot see at all
(`ReloadShadersEvent`, `SetPausedEvent`, `CrashLogEvent` and the rest of the list the shipped
scripts enumerate in a comment but do not declare).

#### Better: read the name, do not call for it

Every one of these getters compiles to the same fourteen bytes, because every one of them is
the same one-line function:

```asm
48 8D 05 <rel32>   lea rax, [rip+disp]   ; the string literal
48 89 02           mov [rdx], rax
48 8B C2           mov rax, rdx
C3                 ret
```

So a hook does not have to **call** an unknown function pointer out of a vtable in the
middle of a frame. It can read those fourteen bytes, check they are that exact shape, decode
the `rel32`, and read the literal — all plain memory reads, nothing in the game executed.

The shape check is also the validation, which is the point. Of the **168** classes in this
build whose name contains "Event", **106** have the shape at slot 2. The 62 that do not are
not broadcaster events at all: handlers, functors, weak-pointer trackers, `AISlotEvent*`,
`NetworkMessageShotEvent`, the `Statistics::StatEvent*` hierarchy. Restricting to the real
`game::*Event` and `enf::*Event` classes, **76 of 77 match**, and the one that does not —
`enf::AISlotEvent`, whose slot 2 is `mov al, 1; ret` — is a predicate on a class that is not
raised through the event manager. An object that is not an event is therefore *rejected*
rather than misread, and the loader reports it by vtable address instead.

Verified at runtime: the loader's `game_events` module does exactly this, and a session that
raised `StartedEvent`, `StartupEvent`, `ScriptLogEvent` and `WorldCleaupEvent` named **5 of 5
classes with 0 rejected**.

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
| **The event manager itself** | `0xFEBBE0` | decompiled; the global the getter returns |
| Raise an event | `0x1D6260` | **verified** — hooked, `(manager, &event)` |
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

#### `0x1D6260` is a plain broadcaster

Decompiled, it is a dozen lines: two listener arrays on the manager, walked in order.

```c
for (i = 0; i < *(int *)(manager + 0x214); i++) {              // first array
    listener = *(longlong **)(*(longlong *)(manager + 0x208) + i * 8);
    (**(code **)(*listener + 8))(listener, event);              // vtable slot 1
}
for (i = 0; i < *(int *)(manager + 0x1F4); i++) {              // second array
    listener = *(longlong **)(*(longlong *)(manager + 0x1E8) + i * 8);
    (**(code **)(*listener + 0x78))(listener, event);           // vtable slot 15
}
```

| Offset on the manager | What |
| --- | --- |
| `+0x208` / `+0x214` | Listener array and count, dispatched on vtable slot 1 |
| `+0x1E8` / `+0x1F4` | Listener array and count, dispatched on vtable slot 15 |

Two consequences. A plugin could in principle **register itself as a listener** by appending
to one of those arrays, which needs no code patching at all — but it also needs a fabricated
C++ object with a sixteen-slot vtable and a grown engine array, so the loader detours the
function instead. And a detour here is the one place a five-byte patch is genuinely risky,
because the engine calls it constantly: the loader therefore installs during its own
initialisation, on the game's first DXGI call, while the renderer is still being built.

#### Which hook point to use

- **`0x1D6260`** sees *every* event the engine raises, including those that never reach
  script, and its `this` is the global at `0xFEBBE0`. Best for a listener, and what the
  loader uses.
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

The three string arguments are `RefString **` — pointers to Enfusion refcounted string
holders, which the function **moves out of**: it takes each holder, nulls the caller's
pointer, and owns the reference from then on. The event it builds holds its vtable, those
three pointers and the channel; the exact field offsets are not given here, because the
decompiler's stack-local names are not field offsets and reading them as such is how the first
draft of this file got them wrong. They are also not needed — the arguments carry the same
four values before the event exists.

#### The string holder layout

`0x5DC10` builds a holder from a C string and settles it, because the allocation size and the
three writes agree:

```c
puVar4 = FUN_14033a260(len + 0x18);      // allocate header + characters
*puVar4 = 0;                              // +0x00  u32 refcount
*(ulonglong *)(puVar4 + 2) = len;         // +0x08  u64 length, excluding the terminator
memmove(puVar4 + 4, src, len + 1);        // +0x10  the characters, NUL terminated
```

| Offset | Field |
| --- | --- |
| `+0x00` | `u32` refcount — the engine touches it with `LOCK INC` / `LOCK DEC` and frees at one |
| `+0x08` | `u64` length |
| `+0x10` | the characters, NUL terminated |

An **empty string is a null holder**, not a holder of length zero: `0x5DC10` returns null for
`""`. So a null holder has to read as `""`, and code that treats null as absent will lose
every empty field.

#### What swallowing costs

Because `0x719B90` takes ownership of the three holders, not calling it leaves them unreleased:
a swallowed line leaks its own text, about `0x18` bytes plus its length. The loader accepts
that and says so. Releasing them from a hook would mean reimplementing the engine's reference
counting against a pointer whose other owners are unknown, and getting that wrong is a double
free in the middle of a frame.

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

Incoming calls arrive at **`0x5BB620`** (**verified** — hooked), which is five lines and
forwards straight to script's `OnRPC`:

```c
FUN_1405bb620(CGame *game, sender, target, int kind, params)
    -> FUN_140317060(game, out, id_of("OnRPC"), sender, target, kind, params)
```

The four values map onto script's `OnRPC(PlayerIdentity sender, Object target, int rpc_type,
ParamsReadContext ctx)` in that order. The loader passes all four through without
interpreting them: `kind` is the mod's own number, and `params` is a serialised stream whose
shape belongs to whoever sent it — see open question 3.

There is no separate "mod command" channel for outgoing calls. A mod sends its own messages with the same four
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

`0x319210` is `FindMethod(scriptClass, name)` and is decompiled in full: it hashes the name
with `h = h * 0x25 + c`, looks the hash up in a table at `+0x78` against a count at `+0x88`,
walks the base-class chain at `+0x28` when it misses, and returns a method index from the
table at `+0x90`. `0x5BB620` and `0x5CACD0` both use it to find `"OnRPC"` and `"OnEvent"`
once and cache the index in a global. Together with `0x317060` that is a general **call any
script method by name** primitive, which is a larger capability than anything on this page
and is not yet used by anything.

## Open questions

1. **The `CGame` instance pointer.** Still needed before any native above can be called, and
   still the thing blocking the console's `connect` and `disconnect`. It is no longer hard to
   reach, though: **`0x5BB620`'s first argument is it**, and that function is now hooked, so
   capturing a live `CGame` is a line of code in the RPC hook — once an RPC has arrived.
   `0x719B90` also takes it, but does not use it, so the argument there is not worth trusting
   without a check. Either candidate can be validated for free by calling the side-effect-free
   predicates `IsClient` (`0x5B69F0`) or `IsMultiplayer` (`0x5B6A60`) on it. The tidy answer
   is still a global, and `0xFEBBE0` is *not* it — that is the event manager.
2. **`RefString` construction** from native code is now answered for reading, and nearly for
   writing: the layout is above and `0x5DC10` builds one from a C string. What is untested is
   handing a holder *we* allocated to an engine native that will release it.
3. **`ParamsReadContext`**, without which RPC payloads can be counted but not read. This is
   now the biggest remaining gap: the loader reports a remote call's number and its three
   pointers, and nothing can be said about its contents.
4. Whether `0x1D6260` is reached by every event or only by the manager's own queue. Partly
   answered by running it: a menu session raised `StartedEvent`, `StartupEvent`,
   `ScriptLogEvent` and `WorldCleaupEvent` through it, so it carries engine lifecycle events
   and not just one subsystem's. Whether anything bypasses it is still unknown.
5. **How an event's payload becomes script's `Param`.** The dispatcher passes exactly two
   values — the `EventType` and the event's second qword — so there is no generic per-class
   marshaller on that path, and `0x5B8560` is not one either. The payload must therefore live
   in the event object's own fields, which means **reading an event's contents is a per-class
   job**: find that class's layout, record the offsets in `dayz-data`, decode it in the
   loader. `ChatMessageEvent` does not need it, because `0x719B90` hands over the same four
   values one level earlier.
6. Runtime verification. No longer "none of this": `0x1D6260`, `0x719B90` and `0x5BB620` are
   hooked and the event path is **verified** — the name decoding, the event manager global and
   the dispatch all work in a live session. What remains decompiled-only is everything about
   *sending*: the chat natives, the four RPC natives, and the session natives.
