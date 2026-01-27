# MixScrims CS2 Plugin - AI Coding Agent Instructions

## Project Overview

MixScrims is a **SwiftlyS2 plugin** that implements FACEIT-style PUG matches with in-game management. It's a **state machine-driven plugin** that progresses through match phases: Warmup → MapVoting → MapChosen → PickingTeam → KnifeRound → PickingStartingSide → Match.

- **Plugin Framework**: [SwiftlyS2](https://swiftlys2.net) - CS2 server modification framework for .NET 10.0
- **Architecture**: State-based partial classes with shared service layer
- **Key Components**: Main plugin (`MixScrims`), Contract API (`MixScrims.Contract`), state handlers, shared services

**Documentation:**
- [Project Wiki](https://github.com/shmitzas/MixScrims-SwiftlyS2/wiki) - Comprehensive guides for installation, configuration, features, and contributing
- [SwiftlyS2 API Reference](https://swiftlys2.net/llms-full.txt) - Framework API documentation

## Match Flow Overview

**Full Sequence:** Warmup → Map Voting → Map Chosen → Team Picking → Knife Round → Side Selection → Match → Ended

**Key Transitions:**
- **Warmup**: Players `!ready` until `MinimumReadyPlayers` met → advances to Map Voting (or Map Chosen if `SkipMapVoting`)
- **Map Voting**: 30s vote window, highest votes wins → map change → Map Chosen state
- **Map Chosen**: Players `!ready` again after map load → Captains selected (random/volunteer/manual) → Team Picking
- **Team Picking**: Captains alternate picking players via menu (random captain picks first) → Knife Round when complete
- **Knife Round**: Knife-only round, winning team captain gets menu → Side Selection
- **Side Selection**: Winner chooses `!stay` or `!switch` sides → Match starts with `match_start.cfg`
- **Match**: Standard MR12 competitive, team timeouts (vote-based, 3 per team), surrender voting → Ended → resets to Warmup

**Config Shortcuts:** `SkipMapVoting` (uses current map), `SkipTeamPicking` (random teams), `DisableCaptains` (team voting for side selection)

## Critical Architecture Patterns

### Partial Class Structure
The main `MixScrims` class is split across **multiple partial class files** organized by concern:
- `src/Main.cs` - Core plugin lifecycle, DI setup, command registration
- `src/StateAgnostic/Main.cs` - Cross-state logic (ready checks, team management)
- `src/States/{StateName}/Main.cs` - State-specific logic (e.g., `KnifeRound/Main.cs`)
- `src/States/{StateName}/Events.cs` - State-specific event handlers

When modifying functionality, **always check if related code exists in other partial files** before adding new methods.

### State Machine Management
State transitions are managed through `MixScrimsService.SetMatchState(MatchState state)`:
```csharp
// States defined in MixScrims.Contract/MatchState.cs
mixScrimsService.SetMatchState(MatchState.MapVoting);
```
**Never modify state directly** - always use the service. State changes trigger phase transitions that execute CFG files and update game rules.

### Command Registration Pattern
Commands are **config-driven** with dynamic aliases:
```csharp
// Config defines: command name → permissions + aliases
{ "ready", new() { Permission = "", Aliases = ["r"] } }

// Registration uses config lookup
Core.Command.RegisterCommand(commandName, handler, true, commandInfo.Permission);
foreach (var alias in commandInfo.Aliases)
    Core.Command.RegisterCommandAlias(commandName, alias);
```
When adding commands: (1) Add handler to `commandHandlers` dict in `RegisterCommands()`, (2) Add config entry to `Config.Commands`.

### Event Handling Pattern
Events use **two registration patterns**:
```csharp
// 1. Delegate subscription (lifecycle events)
Core.Event.OnClientPutInServer += HandleClientPutInServer;
Core.Event.OnMapLoad += WarmupHandleOnMapStart;

// 2. Attribute-based hooks (game events)
[ClientCommandHookHandler]
public HookResult OnClientCommand(int playerId, string commandLine) { ... }

[GameEventHandler]
public HookResult HandleRoundEnd(EventRoundEnd @event) { ... }
```
Game event handlers **must return `HookResult.Continue`** to allow event propagation. Register listeners in `RegisterXListeners()` methods per state.

### Configuration & Localization
- **Config**: JSON-based with JSONC support (`config.jsonc`), bound via `IOptions<Config>`
- **Translations**: Localized strings in `resources/translations/{lang}.jsonc`, accessed via `Core.Localizer["key"]`
- Config reloads on change but **requires plugin reload** for structural changes (maps, commands)

### CS2 Server Control
Match phases execute **CFG files to configure game rules**, loaded from `csgo_configs/`:
```csharp
Core.Engine.ExecuteCommand("exec mixscrims/warmup.cfg");      // mp_warmuptime unlimited
Core.Engine.ExecuteCommand("exec mixscrims/knife_round.cfg");  // mp_give_player_c4 0, mp_ct_default_grenades "weapon_knife"
```
CFG files **must exist on CS2 server** at `csgo/cfg/mixscrims/`. Environment-specific overrides use `staging_overrides.cfg` or `production_overrides.cfg` (controlled by `cfg.TestMode`).

## Development Workflows

### Building
```powershell
# Standard build
dotnet build MixScrims-SwiftlyS2.sln

# Release build for deployment
dotnet publish MixScrims/MixScrims.csproj -c Release
```
Output: `MixScrims/build/publish/MixScrims/` contains deployable plugin files.

### Project Structure for New Features
1. **State-specific logic**: Add to `src/States/{StateName}/Main.cs` or `Events.cs`
2. **Cross-state logic**: Add to `src/StateAgnostic/Main.cs`
3. **Shared utilities**: Add to `src/Shared/Helpers.cs` or create new service
4. **API contracts**: Extend `MixScrims.Contract/IMixScrims.cs` (shared with other plugins)

### Dependency Notes
- **SwiftlyS2.CS2**: NuGet package providing framework API (use wildcard version `*`)
  - **Must use** `ExcludeAssets="runtime" PrivateAssets="all"` to avoid bundling
  - Framework provides `ISwiftlyCore` accessed via `Core` property in plugins
- **MixScrims.Contract**: Separate project for plugin API exposure
  - Set `<Private>false</Private>` in ProjectReference to prevent .dll export
- Microsoft.Extensions.* packages: All require `ExcludeAssets="runtime"` pattern

## Project-Specific Conventions

### Player Management
Players are tracked via `IPlayer` from SwiftlyS2. **Three lifecycle stages**:
1. `readyPlayers` - Warmup ready check (`!ready` typed)
2. `pickedCtPlayers`/`pickedTPlayers` - Captain selections during team picking
3. `playingCtPlayers`/`playingTPlayers` - Active match rosters (knife → match end)

Lists clear at phase transitions (e.g., `readyPlayers.Clear()` when knife starts). **Always check player validity**:
```csharp
if (!player.IsValid || !player.IsConnected || player.IsBot) return;
```

**Captain Selection:** Volunteers (`!volunteer_captain t/ct` if enabled) take priority, else random from ready players. Team names set to captain names unless `DisableCaptains` is true.

### Timer/Scheduler Pattern
Use `Core.Scheduler` (ISchedulerService) for async operations:
```csharp
var token = Core.Scheduler.DelayBySeconds(5, () => StartMatch());
Core.Scheduler.StopOnMapChange(token); // Auto-cleanup on map change
```
Available methods: `DelayBySeconds()`, `RepeatBySeconds()`, `NextTick()`, `NextWorldUpdate()`. **Always store CancellationTokenSource** for manual cleanup.

### Logging Strategy
- `cfg.DetailedLogging` flag enables verbose logs
- Use structured logging: `logger.LogInformation("Message {Param}", value)`
- State transitions should always log

### Team Enum Gotcha
`Team` enum: `None = 0, Spectator = 1, T = 2, CT = 3`
**Never assume numeric values** - always use enum names.

### Config Features Explained
- **FaceitLikeDamageControl**: Reflects friendly fire damage back to shooter
- **MoveOverflowPlayersToSpec**: Auto-moves players beyond 10 to spectator during team picking
- **RequireAllConnectedPlayersToBeReady**: If true, ALL connected players must ready (not just `MinimumReadyPlayers`)
- **DisallowVotePreviousMaps**: Excludes N recently played maps from voting pool
- **TestMode**: Loads `staging_overrides.cfg` (adds bots, lower requirements) instead of `production_overrides.cfg`

## Key Files Reference

- [MixScrims/src/Main.cs](MixScrims/src/Main.cs) - Plugin entry point, DI, lifecycle
- [MixScrims/src/StateAgnostic/Main.cs](MixScrims/src/StateAgnostic/Main.cs) - Ready system, team logic
- [MixScrims/src/Shared/Config.cs](MixScrims/src/Shared/Config.cs) - Configuration model
- [MixScrims.Contract/MatchState.cs](MixScrims.Contract/MatchState.cs) - State enum
- [MixScrims/resources/translations/en.jsonc](MixScrims/resources/translations/en.jsonc) - Localization keys

## Integration Points

### Shared API Pattern
This plugin exposes `IMixScrims` for other plugins via Swiftly's shared interface system:
```csharp
// Provider (this plugin)
interfaceManager.AddSharedInterface<IMixScrims, MixScrimsService>("MixScrims.API", mixScrimsService);

// Consumer (other plugins)
var mixScrims = interfaceManager.GetSharedInterface<IMixScrims>("MixScrims.API");
var state = mixScrims.GetCurrentMatchState();
```

### Discord Webhooks
Optional Discord notifications via `DiscordWebHook.cs` - sends formatted messages on player invite commands. Configured in `Config.DiscordInviteWebhooks`.

## Testing & Debugging

**Quick Test Setup:**
```csharp
// In config.jsonc
"MinimumReadyPlayers": 4,  // Lower for bot testing
"DetailedLogging": true,    // Verbose state transitions
"TestMode": true            // Loads staging_overrides.cfg with bots
```

**Console Commands:**
```
bot_add_ct; bot_add_ct; bot_add_t; bot_add_t  // Add 4 bots for testing
```

**Key Test Scenarios:** Default flow (all features), `SkipMapVoting`, `SkipTeamPicking`, `DisableCaptains`, edge cases (player disconnects during picking, odd player counts). See [Testing Guide](https://github.com/shmitzas/MixScrims-SwiftlyS2/wiki/%F0%9F%A7%AA-Testing-Guide) for comprehensive scenarios.

## Common Pitfalls

1. **Forgetting partial class scope** - Methods may already exist in other partial files
2. **Direct state modification** - Always use `mixScrimsService.SetMatchState()`
3. **Missing config entries** - New commands need both code + config entries
4. **CFG file deployment** - Changes to `.cfg` files require manual server deployment
5. **Player validity checks** - Always verify `IsValid` and `IsConnected` before operations
6. **ExcludeAssets confusion** - SwiftlyS2 packages must not bundle runtime assemblies
7. **Side selection modes** - Captain mode (`DisableCaptains: false`) shows menu to winner captain; team mode (`DisableCaptains: true`) requires majority vote from winning team
