# SwiftlyS2 Plugin Development Guidelines

General best practices for developing SwiftlyS2 plugins for Counter-Strike 2. These patterns are framework-specific and apply to any SwiftlyS2 plugin project.

## Framework Overview

**SwiftlyS2** is a server modification framework for CS2 built on Metamod:Source, targeting .NET 10.0.

- **Official Documentation**: [SwiftlyS2 API Reference](https://swiftlys2.net/llms-full.txt)
- **GitHub**: [swiftly-solution/swiftlys2](https://github.com/swiftly-solution/swiftlys2)
- **Core Interface**: All plugins access framework via `ISwiftlyCore` (conventionally named `Core`)

## Plugin Structure

### Required Metadata
Every plugin must declare metadata using the `[PluginMetadata]` attribute:

```csharp
[PluginMetadata(
    Id = "YourPluginId",
    Version = "1.0.0",
    Name = "Your Plugin Name",
    Author = "Your Name",
    Description = "Plugin description"
)]
public class YourPlugin : BasePlugin
{
    public static new ISwiftlyCore Core { get; private set; } = null!;
    
    public YourPlugin(ISwiftlyCore core) : base(core)
    {
        Core = base.Core;
    }
}
```

### Lifecycle Methods
Override these methods from `BasePlugin`:

```csharp
public override void Load(bool hotReload)
{
    Core = base.Core;
    Core.Registrator.Register(this); // Required for attribute-based hooks
    
    // Initialize your plugin
    LoadConfiguration();
    RegisterCommands();
    RegisterEvents();
}

public override void Unload()
{
    // Clean up resources
    UnregisterCommands();
    logger?.LogInformation("Plugin unloading...");
}

public override void ConfigureSharedInterface(IInterfaceManager interfaceManager)
{
    // Optional: Expose API for other plugins
    interfaceManager.AddSharedInterface<IYourApi, YourService>("YourPlugin.API", yourService);
}
```

## Dependency Management

### SwiftlyS2.CS2 Package
**Critical:** Always use these settings for the SwiftlyS2.CS2 NuGet package:

```xml
<PackageReference Include="SwiftlyS2.CS2" Version="*" ExcludeAssets="runtime" PrivateAssets="all">
    <IncludeAssets>compile; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
</PackageReference>
```

- Use wildcard version `*` to always get latest compatible version
- `ExcludeAssets="runtime"` prevents bundling runtime assemblies (framework provides them)
- `PrivateAssets="all"` prevents transitive dependencies from being exposed

### Microsoft.Extensions.* Packages
Same pattern for all Microsoft.Extensions packages:

```xml
<PackageReference Include="Microsoft.Extensions.Logging" Version="10.0.0" ExcludeAssets="runtime" PrivateAssets="all" />
<PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="10.0.0" ExcludeAssets="runtime" PrivateAssets="all" />
```

**Why?** SwiftlyS2 runtime already provides these assemblies. Bundling them causes conflicts and bloat.

## Configuration Pattern

### JSON Configuration with Dependency Injection
```csharp
private void LoadConfig()
{
    const string fileName = "config.jsonc"; // JSONC (with comments) supported
    const string section = "YourPluginName";

    Core.Configuration
        .InitializeJsonWithModel<ConfigModel>(fileName, section)
        .Configure(builder =>
        {
            builder.AddJsonFile(
                Core.Configuration.GetConfigPath(fileName),
                optional: false,
                reloadOnChange: true
            );
        });

    ServiceCollection services = new();
    services
        .AddSwiftly(Core, addLogger: true, addConfiguration: true)
        .AddOptionsWithValidateOnStart<ConfigModel>()
        .BindConfiguration(section);

    var provider = services.BuildServiceProvider();
    
    logger = provider.GetRequiredService<ILogger<YourPlugin>>();
    var cfgOptions = provider.GetRequiredService<IOptions<ConfigModel>>();
    cfg = cfgOptions.Value;
}
```

**Config file location:** `addons/swiftlys2/plugins/{YourPlugin}/config.jsonc`

## Event Handling

### Two Registration Patterns

**1. Delegate Subscription (Lifecycle Events)**
```csharp
private void RegisterEvents()
{
    Core.Event.OnClientPutInServer += HandleClientJoin;
    Core.Event.OnClientDisconnected += HandleClientDisconnect;
    Core.Event.OnMapLoad += HandleMapLoad;
}

private void HandleClientJoin(IOnClientPutInServerEvent ev)
{
    var player = Core.PlayerManager.GetPlayer(ev.PlayerId);
    if (player == null || !player.IsValid) return;
    // Handle join
}
```

**2. Attribute-Based Hooks (Game Events)**
```csharp
// Must call Core.Registrator.Register(this) in Load() first

[GameEventHandler]
public HookResult HandleRoundEnd(EventRoundEnd @event)
{
    // Handle round end
    return HookResult.Continue; // MUST return HookResult
}

[ClientCommandHookHandler]
public HookResult HandleClientCommand(int playerId, string commandLine)
{
    if (!commandLine.StartsWith("jointeam")) 
        return HookResult.Continue;
    
    var player = Core.PlayerManager.GetPlayer(playerId);
    // Handle command
    return HookResult.Stop; // Stop propagation if handled
}
```

**Critical:** Game event handlers MUST return `HookResult.Continue` or `HookResult.Stop` to control event propagation.

## Command System

### Command Registration with Aliases
```csharp
private void RegisterCommands()
{
    var commandHandlers = new Dictionary<string, ICommandService.CommandListener>
    {
        { "mycommand", OnMyCommand }
    };

    foreach (var (commandName, handler) in commandHandlers)
    {
        if (!config.Commands.TryGetValue(commandName, out var commandInfo))
            continue;

        // Register main command with permission
        Core.Command.RegisterCommand(commandName, handler, true, commandInfo.Permission);

        // Register all aliases
        foreach (var alias in commandInfo.Aliases)
        {
            Core.Command.RegisterCommandAlias(commandName, alias);
        }
    }
}

private void OnMyCommand(ICommandEvent commandEvent)
{
    var player = commandEvent.Player;
    if (player == null || !player.IsValid) return;
    
    player.SendChat("Command executed!");
}
```

**Config structure:**
```json
{
  "Commands": {
    "mycommand": {
      "Permission": "admin",
      "Aliases": ["mycmd", "mc"]
    }
  }
}
```

## Player Management

### Safe Player Access
**Always validate players before use:**

```csharp
private void DoSomethingWithPlayer(IPlayer player)
{
    if (player == null || !player.IsValid || !player.IsConnected || player.IsBot)
        return;
    
    // Safe to use player
    player.SendChat("Hello!");
}
```

### Common Player Operations
```csharp
// Get all players
var players = Core.PlayerManager.GetPlayers().Where(p => p.IsValid && p.IsConnected);

// Get player by slot
var player = Core.PlayerManager.GetPlayer(playerId);

// Send chat message
player.SendChat("Message");
Core.PlayerManager.SendChat("Broadcast to all");

// Move player to team
player.SwitchTeam(Team.CT);

// Kick/ban
player.Kick("Reason");
// For bans, use Core.Engine.ExecuteCommand("sm_ban ...")
```

### Team Enum Values
```csharp
public enum Team
{
    None = 0,
    Spectator = 1,
    T = 2,
    CT = 3
}
```
**Never assume numeric values** - always use enum names for clarity.

## Scheduler/Timer System

### Available Methods
```csharp
// One-time delayed execution
var token = Core.Scheduler.DelayBySeconds(5, () => 
{
    DoSomething();
});

// Repeated execution
var repeatToken = Core.Scheduler.RepeatBySeconds(10, () => 
{
    DoPeriodicTask();
});

// Next game tick
Core.Scheduler.NextTick(() => DoOnNextTick());

// Next world update
Core.Scheduler.NextWorldUpdate(() => DoOnWorldUpdate());
```

### Auto-Cleanup on Map Change
```csharp
var token = Core.Scheduler.DelayBySeconds(30, () => DoSomething());
Core.Scheduler.StopOnMapChange(token); // Automatically cancels on map change
```

**Always store `CancellationTokenSource`** for manual cancellation when needed.

## Server Control

### Executing CS2 Commands
```csharp
// Execute server command
Core.Engine.ExecuteCommand("mp_warmuptime 300");
Core.Engine.ExecuteCommand("exec myconfig.cfg");

// Change map
Core.Engine.ExecuteCommand("map de_dust2");

// Workshop map
Core.Engine.ExecuteCommand("ds_workshop_changelevel de_cache");
Core.Engine.ExecuteCommand("host_workshop_map 3070212801");
```

### Custom CFG Files
Place config files in `csgo/cfg/yourplugin/` and load with:
```csharp
Core.Engine.ExecuteCommand("exec yourplugin/myconfig.cfg");
```

## Localization

### Translation Files
Create `resources/translations/{lang}.jsonc`:
```json
{
  "server_prefix": "[ [lime]MyPlugin [default]]",
  "message.welcome": "Welcome [lime]{0}[default]!",
  "error.invalid_player": "[red]Invalid player!"
}
```

### Using Translations
```csharp
// Simple key
var message = Core.Localizer["message.welcome"];

// With parameters
var formatted = string.Format(Core.Localizer["message.welcome"], playerName);

// Sending to player
player.SendChat(Core.Localizer["server_prefix"] + " " + Core.Localizer["message.welcome"]);
```

**Color codes:** `[default]`, `[red]`, `[lime]`, `[gold]`, `[lightblue]`, `[orange]`, `[darkred]`, etc.

## Logging

### Structured Logging
```csharp
private ILogger<YourPlugin> logger = null!;

// Set up in LoadConfig() via DI
logger = provider.GetRequiredService<ILogger<YourPlugin>>();

// Usage
logger.LogInformation("Player {PlayerName} joined", player.Controller.PlayerName);
logger.LogWarning("Invalid config value: {Value}", configValue);
logger.LogError(ex, "Failed to process command");
logger.LogDebug("Debug info: {Data}", debugData);
```

**Best practice:** Use structured logging with parameters, not string interpolation.

## Shared API Pattern

### Exposing API to Other Plugins

**1. Create Contract Project**
```csharp
// YourPlugin.Contract/IYourPlugin.cs
public interface IYourPlugin : IDisposable
{
    string GetPluginVersion();
    bool IsFeatureEnabled(string feature);
}
```

**2. Implement Service**
```csharp
// YourPlugin/YourPluginService.cs
public class YourPluginService : IYourPlugin
{
    public string GetPluginVersion() => "1.0.0";
    public bool IsFeatureEnabled(string feature) => features.Contains(feature);
    public void Dispose() { /* cleanup */ }
}
```

**3. Register in Plugin**
```csharp
public override void ConfigureSharedInterface(IInterfaceManager interfaceManager)
{
    interfaceManager.AddSharedInterface<IYourPlugin, YourPluginService>(
        "YourPlugin.API", 
        yourPluginService
    );
}
```

**4. Consume from Other Plugins**
```csharp
var yourPlugin = interfaceManager.GetSharedInterface<IYourPlugin>("YourPlugin.API");
if (yourPlugin != null)
{
    var version = yourPlugin.GetPluginVersion();
}
```

**Contract Project Settings:**
```xml
<ItemGroup>
    <ProjectReference Include="..\YourPlugin.Contract\YourPlugin.Contract.csproj">
        <Private>false</Private> <!-- Don't export .dll -->
        <CopyLocalSatelliteAssemblies>false</CopyLocalSatelliteAssemblies>
    </ProjectReference>
</ItemGroup>
```

## Common Pitfalls

1. **Forgetting `ExcludeAssets="runtime"`** on SwiftlyS2.CS2 → runtime conflicts
2. **Not checking player validity** → null reference exceptions
3. **Not returning `HookResult`** from game event handlers → compilation errors
4. **Not calling `Core.Registrator.Register(this)`** → attribute-based hooks don't work
5. **Bundling runtime DLLs** → plugin fails to load or conflicts with framework
6. **Assuming team enum numeric values** → breaks when framework updates
7. **Not storing scheduler tokens** → memory leaks from uncancelled timers
8. **Executing commands synchronously** → use `Core.Scheduler.NextTick()` for CS2 commands

## Build & Deployment

### Building
```powershell
# Debug build
dotnet build YourPlugin.sln

# Release build
dotnet publish YourPlugin/YourPlugin.csproj -c Release
```

### Deployment Structure
```
addons/
  swiftlys2/
    plugins/
      YourPlugin/
        YourPlugin.dll
        YourPlugin.deps.json
        config.jsonc
        resources/
          translations/
            en.jsonc
            pt-BR.jsonc
```

### .csproj Output Settings
```xml
<PropertyGroup>
    <OutputPath>$(MSBuildThisFileDirectory)build/</OutputPath>
    <PublishDir>$(OutputPath)publish/YourPlugin</PublishDir>
</PropertyGroup>

<ItemGroup>
    <Content Include="resources/**/*">
        <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
        <CopyToPublishDirectory>PreserveNewest</CopyToPublishDirectory>
    </Content>
</ItemGroup>
```

## Testing

### Test Mode Configuration
```json
{
  "TestMode": true,
  "DetailedLogging": true
}
```

- `TestMode`: Load test-specific configs/bots
- `DetailedLogging`: Enable verbose logging

### Bot Commands
```
bot_add_ct
bot_add_t
bot_kick
bot_quota 4
```

### Console Verification
```
sw             # Verify SwiftlyS2 loaded
sw plugins     # List loaded plugins
sw reload      # Reload plugins
```

## Performance Best Practices

1. **Cache frequently accessed values** - Don't repeatedly fetch config/players
2. **Use `Core.Scheduler.NextTick()`** for deferred operations
3. **Batch operations** - Collect data, then process in one scheduler callback
4. **Avoid synchronous blocking** - Never block game thread
5. **Clean up timers** - Always cancel schedulers when done
6. **Minimize event handler complexity** - Keep handlers fast, defer heavy work

## Resources

- **API Documentation**: https://swiftlys2.net/llms-full.txt
- **GitHub Repository**: https://github.com/swiftly-solution/swiftlys2
- **Installation Guide**: https://swiftlys2.net/docs/installation
- **Example Plugins**: Check official SwiftlyS2 repositories for reference implementations
