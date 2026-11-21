# RXPGuides

This is Jeol's personal fork of RestedXP Guides for World of Warcraft. The focus is UI appearance, layout, visibility, and in-game settings. Keep work local unless Jeol asks otherwise. The inherited `.github/workflows/package.yml` runs the BigWigs packager on pushes to `main` and on tags. Leave guide routes, gameplay automation, and packaging alone for UI-only changes unless the requested feature needs them.

## Start with the load order

- `RXPGuides.toc`, `RXPGuides_Vanilla.toc`, and `RXPGuides_TBC.toc` declare load order and SavedVariables. Check the manifest for the client being tested. The shared TOC uses `AllowLoadGameType` to choose library embeds, database files, guides, and feature modules. Forever uses `camelot` there and `addon.game == "FOREVER"` in Lua. `RXPGuides.lua` sets `addon.game` from the client interface version.
- Addon files share the namespace passed as `local addonName, addon = ...`. Library embeds and locale tables load first, then `UI/includes.xml` and `UI/V2/manifest.xml`, then `Locale.lua` creates the AceAddon instance and exposes `addon.locale.Get`. The V2 files load early and defer work that needs the locale helper or profile until initialization.
- `RXPGuides.lua` is the lifecycle entry point. `OnInitialize` initializes the database and options, registers the V2 widgets, loads themes, and sets up the frames and modules. `OnEnable` restores saved frame positions and reapplies anchors. `PLAYER_LOGOUT` saves frame positions before WoW writes SavedVariables, including on `/reload`.

## Find the UI owner

| Surface | Start here |
| --- | --- |
| Guide header, step list, active steps, and guide menus | `GuideWindow.lua` owns legacy rendering and shared guide state. Start with `SetupGuideWindow`, `RenderFrame`, `SetStep`, and `RXPFrame:GenerateMenuTable`. For the V2 shell and list, see `RegisterRXPV2GuideWindow`, `BuildGuideStepsSnapshot`, and `RegisterRXPV2GuideSteps` in `UI/V2/Core.lua`. |
| V2 active steps and party steps | `GuideWindow.lua` manages anchoring and row updates through `SetActiveStepsFrameAnchor`, `UpdateActiveStepsFrame`, and `UpdateActivePartyStepsFrame`. `UI/V2/Core.lua` registers the AceGUI frames, cards, and rows. `RXPGuides.lua` wires up the `addon.v2.events` refresh messages. |
| Theme colors, fonts, borders, and textures | `Themes.lua` owns theme selection, `LoadActiveTheme`, `ReloadTheme`, and `addon.v2:ConvertThemes` / `GetTheme`. Legacy uses `addon.activeTheme`, `addon.colors`, and `addon.GetV1Texture`. V2 uses its own theme fields and `Textures/v2/` for shared assets. Legacy assets live under `Textures/` and `Textures/DarkMode/`. |
| Arrow, world-map and minimap pins, and map buttons | `map.lua` owns the arrow (`SetupArrow`, `DrawArrow`, and `arrowFrame:UpdateVisuals`) and pin rendering (`UpdateMap`). `SettingsPanel.lua` owns the minimap launcher and world-map shortcut (`UpdateMinimapButton`, `SetupMapButton`). |
| Active items, targets, tips, splits, and timers | `ActiveItemFrame.lua`, `Targeting.lua`, `Tips.lua`, `LevelingTracker.lua`, and `Timers.lua`. Item and target frames have V1 and V2 visual update paths. Preserve the secure click behavior and existing combat-lockdown handling when changing item or target buttons. |
| Guide configurator, importer, and auction house | `UI/V2/GuideConfigurator.lua`, `UI/V2/GuideImporter.lua`, and `UI/V2/AuctionHouse.lua` own the visible windows. Guide import and parsing live in `GuideLoader.lua`. Auction house behavior lives in `ItemUpgrades.lua`. |

`addon.v2:IsEnabled()` requires both `enableBetaFeatures` and `enableV2Interface`. The `Enable V2 Interface` option triggers a reload. Even when V2 is active, `addon.RXPFrame` still provides shared state and hooks. For code that positions elements against the displayed guide window, check `addon.v2:GetGuideWindowAnchorFrame()` and the existing fallback to `addon.RXPFrame`.

Guide definitions are in `Guides/`, translated guides in `lang/`, and client-specific game data in `DB/`. `GuideLoader.lua` registers, loads, and parses guides, while `functions.lua` implements many guide directives. `libs/` contains bundled dependencies. Start in addon-owned files for UI work.

## Change in-game settings

- `/rxp`, `/rxpg`, and `/rxpguides` open settings when invoked without arguments. `SettingsPanel.lua` builds the AceConfig tabs in `CreateAceOptionsPanel`. Most appearance controls live in `lookAndFeel`. V2 controls live in `generalSettings` and are gated by `Enable Beta Features` in `advancedSettings`. Use `addon.locale.Get` (`L`) for new labels and tooltips where neighboring controls use it.
- Trace each option key through `settingsDBDefaults.profile`, its option descriptor, and the runtime consumer. The default AceConfig getter and setter use `addon.settings.profile[info[#info]]`. Follow nearby setters for immediate redraw, layout, or reload behavior. For example, `windowScale` applies the new scale and triggers the V2 guide layout refresh.
- `RXPSettings` is the account-wide AceDB settings store. `addon.settings.profile` is the active profile, with defaults selected in `InitializeDatabase` from `RXPData.defaultProfile`, `RXPCData.localDB`, or `settingsDBDefaults`. Forever also fills missing values in `UpdateLocaleProfileDefaults`. Existing saved values can take precedence over a code default. Check profile change, copy, and reset handlers when changing how a setting is applied. `RXPCSettings` is a legacy migration source, `RXPCData` holds per-character state including guide progress, and `RXPDB` stores imported guides and reports.
- For layout, trace `addon.enabledFrames`, `SaveFramePositions`, `LoadFramePosition`, and `LoadScales`. Saved `framePositions` and `frameSizes` can override constructor defaults. Check for managed anchors and frame-specific size restoration before changing `SetPoint` or `SetSize` calls. Keep existing profile data and character progress intact.

## Verify changes

- Match nearby Lua conventions and the four-space indentation in `.editorconfig`. Use `luac -p <file>` with a working Lua 5.1 compiler for changed Lua files. The repository has no automated addon test suite, and syntax checks cannot exercise WoW's frame API.
- To verify behavior, load the local `RXPGuides` folder from the intended client's `Interface/AddOns` directory and use `/reload`. Test the affected control through `/rxp` and check the visible result. Use `/fstack` to identify a frame, and `/rxp preview` to bring up the arrow, active items, and target frame for positioning. For persisted layout or settings, check after `/reload` and with an existing profile. Exercise both UI modes when the change affects shared frames or settings.
