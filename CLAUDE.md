# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MaxDps is a World of Warcraft addon (Lua, WoW API) that highlights optimal spells on action bars as a rotation helper. It is a **framework** — class-specific rotation logic lives in separate addon modules (e.g., `MaxDps_Warrior`, `MaxDps_Mage`) that are installed independently.

## Linting

```bash
# Requires luacheck installed (brew install luacheck / luarocks install luacheck)
luacheck . -q
```

Configuration is in `.luacheckrc` (Lua 5.1 standard, `Libs/` excluded, WoW API globals declared).

## Build & Release

Packaging is handled by [BigWigsMods/packager](https://github.com/BigWigsMods/packager) via GitHub Actions. Tag pushes trigger builds for 6 WoW versions (Retail, Classic, BCC, Wrath, Cata, Mists). External libraries are pulled via `.pkgmeta` — do not vendor Ace3 or other libs manually.

There are no local build or test commands; the addon is loaded directly by the WoW client from the `Interface/AddOns/` directory.

## Architecture

### Core Loop

1. **Init** (`Core.lua`): Addon registers via Ace3. On combat enter (or loading screen, configurable), it calls `InitRotations()` which loads either a custom user rotation or a class module.
2. **Rotation Timer**: A repeating timer (default 0.15s) calls `InvokeNextSpell()` → `PrepareFrameData()` → class module's `NextSpell(self)` → `GlowNextSpell()` to highlight the button.
3. **Disable**: On combat exit, the timer stops and glows are cleared.

### Key Files

| File | Role |
|---|---|
| `Core.lua` | Addon lifecycle, rotation enable/disable, event handling, `InvokeNextSpell` loop |
| `Buttons.lua` | Scans 11+ action bar addons (ElvUI, Bartender4, Dominos, Blizzard, etc.) for spell buttons, manages glow overlays |
| `Helper.lua` | Rotation utility API: `Cooldown()`, `Aura()`, `TargetAura()`, `HasTalent()`, `SpellAvailable()`, `AttackHaste()`, `TargetPercentHealth()`, `GlobalCooldown()` |
| `SpellData.lua` | Spell ID mappings by class/spec (Retail only). Format: `MaxDps.classSpellData[class][spec][SpellName] = spellID` |
| `Cooldowns.lua` | Offensive/defensive cooldown definitions per class/spec |
| `Options.lua` | Settings UI via StdUi library |
| `Tier.lua` | Raid tier set piece detection |
| `TimeToDie.lua` | Enemy time-to-die estimation |
| `SpellFrame.lua` | Large overlay frame showing the current suggested spell |
| `Modules/Custom.lua` | Custom user-written rotation system |
| `Modules/Window.lua` | Settings/editor window UI |
| `Modules/Profiler.lua` | Performance profiling |

### Plugin Architecture (Class Modules)

Class modules are **separate addons** (e.g., `MaxDps_Warrior`) that register via Ace3's `NewModule()`. They only need to implement a `NextSpell(self)` function that:
- Reads `self.FrameData` (cooldowns, buffs, debuffs, talents, spell history, time-to-die, etc.)
- Uses Helper.lua functions for state checks
- Returns a spell ID (number) to highlight, or 0 for none

### FrameData Structure

Populated each tick in `PrepareFrameData()`:
```lua
MaxDps.FrameData = {
    cooldown, activeDot, gcd, buff, debuff, talents,
    azerite, essences, covenant, runeforge, spellHistory, timeToDie, ACSpells
}
```

### Multi-Version Support

The addon supports Retail, Classic, BCC, Wrath, Cata, and Mists via interface version checks (`IsRetailWow()`, `IsClassicWow()`, etc.) and conditional `.toc` blocks. `SwingTimer.lua` is Classic-only.

### WeakAuras Integration

Fires `MAXDPS_SPELL_UPDATE` event with the suggested spell ID, allowing WeakAuras to display suggestions independently.

## Conventions

- All spell references use numeric spell IDs, not names.
- The addon uses Ace3 extensively: AceAddon, AceDB, AceEvent, AceTimer, AceHook, AceConsole, AceBucket.
- SavedVariables are stored in `MaxDpsOptions`.
- Button detection iterates known frame name patterns per action bar addon.
- Frame overlays use an object pool (`FramePool`) for performance.
