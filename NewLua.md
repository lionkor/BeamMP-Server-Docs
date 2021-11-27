**To BeamMP Staff: Do __NOT__ edit this wiki. It's still heavily WIP.**

# Introduction

The Server's Plugin system uses [Lua 5.3](https://www.lua.org/manual/5.3/). This section details how to get started writing plugins, teaches some basic concepts and gets you started with your first plugin. **It is recommended you read this section even if you know the pre-v2.4.0 system, as a few things changed drastically**.

For a migration guide from pre-v2.4.0 lua, go to the section ["Migrating from old Lua"](#migrating-from-old-lua).

## How to Start Writing a Plugin

### Directory Structure

Server plugins, unlike mods, are situated (by default) in `Resources/Server`, while mods, which are written for BeamNG.drive and are sent to the clients are in `Resources/Client`. Each plugin must have it's own subfolder in `Resources/Server`, for example for a plugin called "MyPlugin", the structure would be:

```
Resources
└── Server
    ├── MyPlugin
    │   └── main.lua
    └── SomeOtherPlugin
        └── ...
```

Here we also display another plugin called "SomeOtherPlugin", to illustrate how your `Resources/Server` folder can have multiple different plugin folders. We will keep using this directory structure as an example throughout this guide.

You also notice the `main.lua`. You can have as many Lua `.lua` files as you like. All Lua files in your plugin's main directory are loaded in *alphabetical order* (so `aaa.lua` is run before `bbb.lua`).


### Lua Files

Each Lua `.lua` file in the plugin's folder is loaded on server startup. This means that statements outside of functions are evaluated ("run") immediately.

Lua files in subfolders are ignored, but can be `require()`-ed.

For example, our `main.lua` looks like this:

```lua
function PrintMyName()
	print("I'm 'My Plugin'!")
end

print("What's up!")
```

When the server starts and the `main.lua` is loaded, it will run `print("What's up!")` *immediately*, but will **NOT** *call* the `PrintMyName` function yet (because it wasn't called)!

### Events

An event is something like "a player is joining", "a player sent a chat message", "a player spawned a vehicle".

You can cancel events (if they are cancellable) by returning `1` from the handler.

In Lua, you usually want to react to some of these. For this, you can register a "Handler". This is a function which is called when an event happens, and gets passed some arguments.

Example:

```lua
function MyChatMessageHandler(sender_id, sender_name, message)
	-- censoring only the exact message 'darn'
	if message == "darn" then
		-- cancel the event by returning 1
		return 1
	else
		return 0
	end
end

MP.RegisterEvent("onChatMessage", "MyChatMessageHandler")
```

This will effectively make sure that any message that is exactly equal to "darn" will not be sent and won't show in chat. Cancelling an event causes it to not happen, for example a chat message not to be shown to anyone else, a vehicle not to be spawned, etc.

### Custom Events

You can register to any event you like, for example:

```lua
MP.RegisterEvent("MyCoolCustomEvent", "MyHandler")
```

You can then trigger those custom events:

```lua
-- call all event handlers to this in ALL plugins
MP.TriggerGlobalEvent("MyCoolCustomEvent")
-- call all event handlers to this in THIS plugin
MP.TriggerLocalEvent("MyCoolCustomEvent")
```

You can do a lot more with events, but those possibilities will be covered in detail below in the API reference.

### Event Timers ("Threads")

Pre-v2.4.0 Lua had a concept of "threads" which run X times per second. This naming was slightly misleading, as they were synchronous.

Post-v2.4.0 Lua instead has "Event Timers". These are timers which run inside the server, and once they run out, they trigger an event (globally). This is also synchronous. Please be aware that the second argument is an interval in milliseconds.

Example:

```lua
local seconds = 0

function CountSeconds()
	seconds = seconds + 1
end

-- create a custom event called 'EverySecond'
-- and register the handler function 'CountSeconds' to it
MP.RegisterEvent("EverySecond", "CountSeconds")

-- create a timer for this event, which will fire every 1000ms (1s)
MP.CreateEventTimer("EverySecond", 1000)
```

This will cause "CountSeconds" to be called every second. You can also cancel event timers with `MP.CancelEventTimer` (see API reference)

# API Reference

Documentation format: `function_name(arg_name: arg_type, arg_name: arg_type) -> return_types`

# Builtin Functions

## `print(...)`, `printRaw(...)`

Prints the message to the server console, prefixed with `[DATE TIME] [LUA]`. If you don't want this prefix, you can use `printRaw(...)`.

Example:

```lua
local name = "John Doe"
print("Hello, I'm", name, "and I'm", 32)
```

It can take as many arguments of arbitrary types as you like. It will also happily dump tables!

This behaves like the lua interpreter's `print`, so it will put tabs between arguments.

## `exit()`

Shuts down the server gracefully. Causes the `onShutdown` event to be triggered.

# MP Functions

## `MP.CreateTimer() -> Timer`

Creates a timer object, which can be used to keep track of how long something took / how much time elapsed. It starts once created, and can be reset/restarted with `mytimer:Start()`.

You can get the current elapsed time in seconds with `mytimer:GetCurrent()`.

Example:

```lua
local mytimer = MP.CreateTimer()
-- do stuff here that needs to be timed
print(mytimer:GetCurrent()) -- print how much time elapsed
```

Timers do not need to be stopped (and can't be stopped), they have no overhead.

## `MP.GetOSName() -> string`

Returns the name of the current OS, either `Windows`, `Linux` or `Other`.

## `MP.GetServerVersion() -> number,number,number`

Returns the current server version in major, minor, patch format. For example, the v2.4.0 version would return `2, 4, 0`.

Example:

```lua
local major, minor, patch = MP.GetServerVersion()
print(major, minor, patch)
```
Output:
```
2	4	0
```

## `MP.RegisterEvent(event_name: string, function_name: string)`

Remembers the function with name `Function Name` as an event handler to event with name `Event Name`.

You can register as many handlers to an event as you like.

For a list of events the server provides, see here (TODO link).

If the event with that name doesn't exist, it's created, and thus RegisterEvent cannot fail. This can be used to create custom events.

## `MP.CreateEventTimer(event_name: string, interval_ms: number)`

Starts a timer inside the server which triggers the event `event_name` every `interval_ms` milliseconds.

Event timers can be cancelled with `MP.CancelEventTimer`.

Intervals <25 ms are not encouraged, as multiple such intervals will likely not be served in time reliably. While multiple timers can be started on the same event, it's encouraged to create as few event timers as possible. For example, if you need one event that runs every half second, and one which runs every second, consider just making the half-second one and running the every-second-functiosecond trigger.

You may also use `MP.CreateTimer` to make a timer and measure time passed since the last event call, in order to minimize event timers, though this is not necessarily recommended as it increases the code complexity significantly.

## `MP.CancelEventTimer(event_name: string)`

Cancels all timers on the event with the name `event_name` On some occasions, the timer might go off one more time before being cancelled, due to the nature of asynchronous programming.

## `MP.TriggerLocalEvent(event_name: string, ...) -> table`

Plugin-local synchronous event trigger.

Triggers an event locally, which causes all handlers to that event *in the current lua state* (usually the current plugin, unless state was shared via PluginConfig.toml) to be called.

You can pass arguments to this function (`...`) which are copied and sent to all handlers as function arguments.

This call is synchronous and will return once all event handlers finished.

The returned value is a table of all results. If a handler returned a value, it will be in this table, unannotated and unnamed. This can be used to "collect" things, or register sub-handlers for events that can be cancelled. This is practically an array.

Example:

```lua
local Results = MP.TriggerLocalEvent("MyEvent")
print(Results)
```

## `MP.TriggerGlobalEvent(event_name: string, ...) -> table`

Global asynchronous event trigger.

Triggers an event globally, which causes all handlers to that event *in all plugins* (including *this* plugin) to be called.

You can pass arguments to this function (`...`) which are copied and sent to all handlers as function arguments.

This call is asynchronous and returns a future-like object. Local handlers (handlers in the same plugin as the caller) run synchronously and immediately. 

The table returned has two functions:

- `IsDone() -> boolean` tells you whether all handlers have finished. You can wait until this is true by checking it and `MP.Sleep`-ing for a little bit in a loop.
- `GetResults() -> table` returns an unannotated unnamed table with all return values of all handlers. This is practically an array.

Make sure to call these with `Obj:Function()` syntax (`:`, NOT `.`).

Example:

```lua
local Future = MP.TriggerGlobalEvent("MyEvent")
-- wait until handlers finished
while not Future:IsDone() do
	MP.Sleep(100) -- sleep 100 ms
end
local Results = Future:GetResults()
print(Results)
```

Be aware that a handler registering to "MyEvent" here and never returning could lock up your plugin. You likely want to keep track of how long you have waited and stop waiting after a few seconds.

## `MP.Sleep(time_ms: number)`

Waits for an amount of time, specified in milliseconds.

This does not yield the execution of the lua state and nothing will execute in the state while asleep. 

WARNING: Do NOT sleep for >500 ms if you have event handlers registered, unless you know *exactly* what you are doing. This is intended to be used to sleep for 1-100 ms, in order to wait for results or similar. A locked up (sleeping) lua state can slow the entire server down drastically if not careful.

## `MP.SendChatMessage(player_id: number, message: string)`

Sends a chat message that only the specified player can see (or everyone if the ID is `-1`).
In the game, this will not appear as a directed message.

You can use this, for example, to tell a player *why* you cancelled their vehicle spawn, chat message, or similar, or to display some information about your server.

## `MP.TriggerClientEvent(player_id: number, event_name: string, data: string)`

// TODO Documentation incomplete

Will call that event with the given data on the specified client (-1 for broadcast).

## `MP.GetPlayerCount() -> number`

Returns the amount of players currently in the server.

## `MP.IsPlayerConnected(player_id: number) -> boolean`

// TODO Documentation incomplete

Whether the player is connected.

## `MP.GetPlayerName(player_id: number) -> string`

Gets the display-name of the player.

## `MP.RemoveVehicle(player_id: number, vehicle_id: number)`

Removes the specified vehicle for the specified player.

## `MP.GetPlayerVehicles(player_id: number) -> table`

Returns a table of all vehicles the player currently has. Each entry in the table is a mapping from vehicle ID to vehicle data (which is currently a raw json string).

## `MP.GetPlayers() -> table`

Returns a table of all connected players. This table maps IDs to Names, like so:  
```json
{
	0: "LionKor",
	1: "JohnDoe"
}
```

## `MP.IsPlayerGuest(player_id: number) -> boolean`

Whether the player is a guest. A guest is someone who didn't log in, and instead chose to play as a guest. Their name is usually `guest` followed by a long number.

As guests aren't logged in, you might want to disallow them from joining, for example when running a serious racing server or similar.

## `MP.DropPlayer(player_id: number, [reason: string])`

Kicks the player with the specified ID. The reason parameter is optional.

## `MP.GetStateMemoryUsage() -> number`

Returns the memory usage of the current Lua state in bytes.

## `MP.GetLuaMemoryUsage() -> number` 

Returns the memory usage of all lua states combined, in bytes.

## `MP.GetPlayerIdentifiers(player_id: number) -> table`

Returns a table with information about the player, such as beammp forum ID and IP address.

Example:

```json
{
	ip: "1.2.3.4",
	beammp: "1234"
}
```

## `MP.Set(setting: number, ...)`

Sets a ServerConfig setting temporarily. For this, the `MP.Settings` table is useful.

Example:

Turning on Debug mode
```lua
MP.Set(MP.Settings.Debug, true)
```

### `MP.Settings`

You can see an up-to-date list of these by printing them, like so:
```lua
print(MP.Settings)
```

# Events

## Explanation

- Arguments: List of arguments given to handlers of this event
- Cancellable: Whether the event can be cancelled. If it can be cancelled, a handler can do so by returning `1`, like `return 1`.

## Summary of events

A player join triggers the following events in the given order:

1. `onPlayerAuth`
2. `onPlayerConnecting`
3. `onPlayerJoining`
4. `onPlayerJoin`

## System Events

### `onInit`

Arguments: NONE
Cancellable: NO

Triggered right after all files in the plugin were initialized.

### `onShutdown`

Arguments: NONE
Cancellable: NO

Triggered when the server shuts down. Currently happens after all players were kicked.

## Game-Related Events

### `onPlayerAuth`

Arguments: `player_name: string`, `player_role: string`, `is_guest: bool`
Cancellable: YES

First event that gets triggered when a player wants to join.

### `onPlayerConnecting`

Arguments: `player_id: number`
Cancellable: NO

Triggered when a player first starts connecting, after `onPlayerAuth`.

### `onPlayerJoining`

Arguments: `player_id: number`
Cancellable: NO

Triggered when a player has finished loading all mods, after `onPlayerConnecting`.

### `onPlayerDisconnect`

Arguments: `player_id: number`
Cancellable: NO

Triggered when a player disconnects.

### `onChatMessage`

Arguments: `player_id: number`, `player_name: string`, `message: string`
Cancellable: YES

Triggered when a player sends a chat message. When cancelled, it will not show the chat message to anyone, not even the player who sent it.

### `onVehicleSpawn`

Arguments: `player_id: number`, `vehicle_id: number`, `data: string`
Cancellable: YES

Triggered when a player spawns a new vehicle. The `data` argument contains the car's config as json. When cancelled, the car is not spawned.

### `onVehicleEdited`

Arguments: `player_id: number`, `vehicle_id: number`, `data: string`
Cancellable: YES

Triggered when a player edits their vehicle and applies the edit. The `data` argument contains the car's change config as json. When cancelled, the edit is not applied.

### `onVehicleDeleted`

Arguments: `player_id: number`, `vehicle_id: number`
Cancellable: NO

Triggered when a player deletes their vehicle.

### `onVehicleReset`

Arguments: `player_id: number`, `vehicle_id: number`, `data: string`
Cancellable: NO

Triggered when a player resets their vehicle. `data` is the car's data as json.

# Migrating from old Lua

This is a short run-down of the basic steps to take to migrate from old to new lua.

## Understand how the new lua works

For this, please read through the section ["Introduction"](#how-to-start-writing-a-plugin) and all its subsections carefully.
It's necessary to do the next steps properly.

## Search & Replace

First, you should search and replace all MP functions. The substitution should add an `MP.` infront of all MP functions, except `print()`.

Example:

```lua
local players = GetPlayers()
print(#players)
```
becomes

```lua
local players = MP.GetPlayers()
print(#players) -- note how print() doesn't change
```

## Goodbye Threads, Hello Event Timers!

As discussed in the introduction, threads are event timers. For any calls to `CreateThread`, replace it with a call to `CreateEventTimer`. Carefully inspect the timing your old CreateThread had (the number was X per second), and think about what the event timer timeout value is for this (which is in milliseconds). Also keep in mind that instead of a function name, it takes an event name, so you will have to register an event as well.

Example:

```lua
CreateThread("myFunction", 2) -- calls "myFunction" twice per second
```
becomes

```lua
MP.RegisterEvent("myEvent", "myFunction") -- registering our event for the timer
MP.CreateEventTimer("myEvent", 500) -- 500 milliseconds = 2 times per second
```

If you have many event timers, it makes sense to see if you can combine them, e.g. by creating a "every minute" event and registering multiple functions to it which need to be called every minute, instead of having multiple event timers. Each event timer costs the server a little bit of time to trigger.

## No more implicit event calling

You need to register all your events. You cannot rely on function names. In the old lua, this was unclear, but in the new lua this is usually enforced. A good pattern is: 

```lua
MP.RegisterEvent("onChatMessage", "chatMessageHandler")
-- or 
MP.RegisterEvent("onChatMessage", "handleChatMessage")
```

This is a better pattern than calling the handler the same as the event, which is misleading and confusing.

