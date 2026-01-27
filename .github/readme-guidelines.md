# SwiftlyS2 Plugin README Guidelines

Best practices for creating clear, professional README files for SwiftlyS2 plugins. These guidelines are framework-agnostic and apply to any SwiftlyS2 plugin documentation.

## Essential README Structure

### Header Section
```markdown
# Plugin Name

Brief one-line description of what the plugin does.

---

## Features

- 🎯 Feature 1 - Brief description
- ⚡ Feature 2 - Brief description
- 🔧 Feature 3 - Brief description
- 📊 Feature 4 - Brief description

Use emojis sparingly to improve visual scanning.
```

---

## Commands Reference Format

### Table Structure
Use clear, scannable tables with these columns:

```markdown
## Commands

### Player Commands

| Command | Aliases | Permission | Description |
|---------|---------|------------|-------------|
| `!info` | `!i` | None | Show plugin information |
| `!help <command>` | `!h` | None | Show help for specific command |
| `!vote <option>` | `!v` | None | Cast vote for option |
| `!stats [player]` | `!st` | None | Show stats (yours or specified player) |

### Admin Commands

| Command | Aliases | Permission | Description |
|---------|---------|------------|-------------|
| `!reload_config` | `!rcfg` | `admin` | Reload plugin configuration |
| `!toggle_feature` | `!tf` | `admin` | Enable/disable plugin feature |
| `!reset_data` | - | `admin.data` | Reset plugin data |

**Permission Levels:**
- `admin` - Basic admin access
- `admin.stats` - Statistics management
- `admin.config` - Configuration management
```

### Best Practices for Command Tables
- **Alphabetical order** within each category
- **Optional parameters** in `[brackets]`
- **Required parameters** in `<angles>`
- **Multiple aliases** separated by commas
- **"None"** for no permission required
- **"-"** for no aliases

---

## Configuration Reference Format

### Overview Table
Provide high-level config summary first:

```markdown
## Configuration

Config file location: `addons/swiftlys2/plugins/YourPlugin/config.jsonc`

### Quick Reference

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `Enabled` | bool | `true` | Enable/disable plugin |
| `MinimumPlayers` | int | `4` | Minimum players required |
| `EventDuration` | int | `300` | Event duration in seconds |
| `AllowSpectators` | bool | `false` | Allow spectators to vote |
| `MessagePrefix` | string | `"[Plugin]"` | Chat message prefix |
```

### Detailed Configuration Sections

For complex configs, break into logical sections:

```markdown
### Core Settings

| Setting | Type | Default | Description | Valid Values |
|---------|------|---------|-------------|--------------|
| `PluginMode` | string | `"standard"` | Operating mode | `standard`, `advanced`, `custom` |
| `AutoStart` | bool | `true` | Start automatically when conditions met | `true`, `false` |
| `DebugMode` | bool | `false` | Enable verbose logging | `true`, `false` |

### Player Settings

| Setting | Type | Default | Description | Notes |
|---------|------|---------|-------------|-------|
| `MinimumPlayers` | int | `4` | Min players to activate | - |
| `MaximumPlayers` | int | `16` | Max players allowed | 0 = unlimited |
| `TrackPlayerData` | bool | `true` | Enable player data tracking | - |

### Timer Configuration

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `EventDuration` | int | `300` | Event duration in seconds |
| `CooldownDuration` | int | `60` | Cooldown between events |
| `TimerWarningThreshold` | int | `30` | Seconds before showing warning |

### Command Customization

| Command | Aliases | Permission | Configurable |
|---------|---------|------------|--------------|
| `info` | `["i", "about"]` | `""` | ✅ Yes |
| `start` | `["go"]` | `"admin"` | ✅ Yes |

**Note:** Edit command names/aliases in `Commands` section of config.
```

### Configuration Example

Always provide a complete example:

```markdown
### Example Configuration

**Minimal setup (standard mode):**
```json
{
  "YourPlugin": {
    "PluginMode": "standard",
    "MinimumPlayers": 4,
    "AutoStart": true
  }
}
```

**Custom advanced setup:**
```json
{
  "YourPlugin": {
    "PluginMode": "advanced",
    "MinimumPlayers": 8,
    "MaximumPlayers": 16,
    "TrackPlayerData": true,
    "EventDuration": 600,
    "Commands": {
      "info": {
        "Permission": "",
        "Aliases": ["i", "about"]
      }
    }
  }
}
```
```

---

## How It Works Section

### Structure
Explain plugin mechanics in 2-4 short paragraphs:

```markdown
## How It Works

**Overview:**
Brief 1-2 sentence summary of core functionality.

**Phase 1 - Initialization:**
Explain what happens when plugin loads or match starts. Keep to 2-3 sentences.

**Phase 2 - Main Operation:**
Explain primary plugin behavior during active use. Mention key triggers and state changes.

**Phase 3 - Completion:**
Explain how events conclude and cleanup occurs.

**State Machine:** (if applicable)
```
Idle → WaitingForPlayers → Active → Cooldown → Idle
```

**Key Mechanics:**
- **Mechanism 1:** Brief explanation of how it works
- **Mechanism 2:** Brief explanation of technical detail
- **Mechanism 3:** Brief explanation of user-facing behavior
```

### Example - Simple Plugin
```markdown
## How It Works

**Deathmatch Spawning**

This plugin provides instant respawn for deathmatch servers. When a player dies, they are teleported back to a random spawn point after a configurable delay.

The plugin monitors `player_death` events and maintains a queue of dead players. Every game tick, it checks if the respawn delay has elapsed for each queued player. When the delay completes, the player is moved to spectator briefly (to trigger respawn logic), then assigned a random spawn point from the map's spawn list and teleported.

**Spawn Selection Logic:**
- Randomly selects from available spawns
- Checks distance from active players (configurable minimum)
- Rotates through spawn types (CT/T) if `BalancedSpawns` enabled
- Falls back to default spawns if no valid location found
```

### Example - Complex Plugin
```markdown
## How It Works

**Tournament System Overview**

This plugin manages tournaments with bracket progression, match scheduling, and automatic server configuration.

**Bracket Management:**
Tournaments are created with a bracket type (single/double elimination) and team count. Teams are seeded based on registration order or imported rankings. The plugin generates the bracket structure automatically, calculating match pairings for each round.

**Match Execution:**
When a match is scheduled, the plugin:
1. Loads the designated map and match config
2. Moves registered players to their teams
3. Activates match recording (demo + stats)
4. Monitors match completion via `cs_win_panel_match` event
5. Records winner, advances bracket, and queues next match

**State Transitions:**
```
Registration → Bracket Generation → Match Queue → Active Match → Match End → [Next Match | Tournament End]
```

**Data Persistence:**
All tournament data (brackets, scores, player stats) is stored in JSON files in the plugin data directory. Matches can be resumed after server restart by detecting incomplete bracket states.
```

---

## Additional README Sections

### Troubleshooting Table
```markdown
## Troubleshooting

| Problem | Possible Cause | Solution |
|---------|----------------|----------|
| Plugin not loading | SwiftlyS2 not installed | Install SwiftlyS2 first |
| Commands not working | Missing permissions | Check admin permissions |
| Config not applying | Invalid JSON syntax | Validate JSON with linter |
| Players can't join teams | Team size limits | Check `MaximumPlayers` setting |
```

### FAQ Section
```markdown
## Frequently Asked Questions

**Q: Can I use this with other plugins?**  
A: Yes, this plugin is compatible with most others. Known conflicts: PluginX (use API integration).

**Q: How do I reset statistics?**  
A: Delete `data/stats.json` or use `!reset_stats` command (requires admin).

**Q: Does this work on all server types?**  
A: This is designed for community servers. Official matchmaking servers are not supported.
```



## Styling Best Practices

### ✅ Do's
- Use tables for structured data (commands, config, troubleshooting)
- Include visual hierarchy (##, ###, **bold**, `code`)
- Provide complete config examples
- Keep sentences short and scannable
- Use bullet points for lists
- Include code blocks with language hints (```csharp, ```json)
- Add emoji sparingly for visual breaks (🎯 ⚡ 🔧 📊)

### ❌ Don'ts
- Don't write walls of text
- Don't use complex nested tables
- Don't leave config settings unexplained
- Don't forget default values
- Don't use technical jargon without context
- Don't skip the "How It Works" section for complex plugins

---

## Complete Template Example

```markdown
# AwesomePlugin

Advanced player skill tracking system for CS2 with automatic ranking and performance analysis.

---

## Features

- 🎯 Automatic skill rating calculation based on performance
- ⚡ Real-time statistics tracking and leaderboards  
- 🔧 Customizable rating algorithms and metrics
- 📊 Performance trend analysis and reports
- 🌐 Multi-language support (EN, PT-BR, ES)

## Commands

### Player Commands

| Command | Aliases | Permission | Description |
|---------|---------|------------|-------------|
| `!rank` | `!r` | None | Show your current rank |
| `!stats` | `!st` | None | Show your statistics |
| `!top10` | `!leaderboard` | None | Show top 10 players |

### Admin Commands

| Command | Aliases | Permission | Description |
|---------|---------|------------|-------------|
| `!recalculate` | `!recalc` | `admin` | Recalculate all rankings |
| `!reset_stats` | - | `admin.stats` | Reset all statistics |

## Configuration

Config: `addons/swiftlys2/plugins/AwesomePlugin/config.jsonc`

### Core Settings

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `MinimumRounds` | int | `10` | Minimum rounds for ranking |
| `EnableAutoRanking` | bool | `true` | Auto-calculate rankings |
| `TrackStatistics` | bool | `true` | Enable stats tracking |

### Example Configuration

```json
{
  "AwesomePlugin": {
    "MinimumRounds": 5,
    "EnableAutoRanking": true,
    "TrackStatistics": true
  }
}
```

## How It Works

This plugin tracks player performance and calculates skill ratings using a custom algorithm.

Player actions (kills, deaths, assists, objectives) are recorded in real-time. The plugin evaluates performance metrics and assigns a skill rating. Rankings are updated after each round, with higher performance leading to faster progression. All data is persisted to a SQLite database.

**Rating Flow:**
```
Player Action → Metric Calculation → Rating Update → Leaderboard Update → Data Storage
```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Rankings not updating | Ensure minimum rounds threshold is met |
| Stats not saving | Check write permissions on plugin directory |
```

---

## README Checklist

Verify your README includes the essential sections:

- [ ] Clear plugin description (1-2 sentences)
- [ ] Feature list (with optional emojis for visual breaks)
- [ ] Complete commands table (player + admin sections)
- [ ] Configuration reference with defaults and types
- [ ] Example configurations for common use cases
- [ ] "How It Works" explanation (2-4 paragraphs)
- [ ] Troubleshooting section (optional but recommended)

**Note:** Add installation, build, contribution, and support sections based on your project's visibility (public/private) and distribution method.

---

## Tools

**Markdown Validators:**
- [MarkdownLint](https://github.com/DavidAnson/markdownlint)
- [Grammarly](https://grammarly.com) for proofreading

**Table Generators:**
- [Tables Generator](https://www.tablesgenerator.com/markdown_tables)
- [Markdown Tables](https://jakebathman.github.io/Markdown-Table-Generator/)

**Badge Generators:**
- [Shields.io](https://shields.io)
- [Badgen](https://badgen.net)
