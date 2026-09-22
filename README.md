# PaddleSlots

Xbox Elite rear-paddle action panels for **WoW: Forever**.

## 0.7.1 — Panels nested in the crossbar

The default layout now puts each paddle panel inside the native crossbar, next to the bar that uses the same trigger combination, and the panels are anchored to `GamepadMainActionBarFrame` so they follow it if the crossbar is moved or scaled:

- **BASE** sits above the top bar, centred between its d-pad and face-button groups.
- **LT** and **RT** sit above the left and right bars, in the notch between the top d-pad slot and the top face-button slot of each bar.
- **LT + RT** sits in the free box in the middle of the cross, between the BASE bar and the LT + RT bar.

The gap between a native bar's inner d-pad slot and inner face-button slot is only 18 units wide, so a 2 x 2 paddle grid cannot sit inside it; the notch above the centre of each bar is the closest spot with enough room (the offsets are chosen so the expanded, focused grid still clears the expanded native slots). LT + RT is the tight one: its expanded grid overlaps the neighbouring native rings by a few units while both triggers are held.

- **Trigger prompts are off by default.** The crossbar already draws the LT, RT, and LT + RT prompts under its own bars, right next to the new panel spots, so the addon's copies are now the opt-in **Show LT / RT modifier icons** setting.
- Panels that were still at the 0.6 / 0.7 default spots move to the new layout automatically. Panels you moved yourself stay where they are; `/paddles reset` adopts the new layout.
- **Range feedback is polled** with `IsActionInRange` every 0.25 s. The previous push-based approach (`C_ActionBar.EnableActionRangeCheck`) trips a client assert on build 69913 for gamepad storage slots that no native button owns, which crashed the game on startup once a paddle slot held an action.
- **Assign by pressing** works on the first click. Creating the prompt window used to clear the capture state, which left the first prompt blank and unresponsive until it was opened a second time.

## 0.7.0 — Native crossbar look

The four paddle panels are now built from the same art and behaviour as Forever's native gamepad crossbar (`Blizzard_GamepadActionBars`):

- **Slots** use the native circle-slot atlases: `gamepad-actionbar-circleslot-border-normal` / `-pressed` / `-hover`, the circle drop shadow, the `CircleMask` icon mask, and the native circular cooldown swipe.
- **Focus** works like the native bars. The panel for the held trigger combination is *expanded* (40 px slots, `gamepad-actionbar-focus-bg-circ` focus shadow that settles to 40 % opacity, `gamepad-actionbar-focus-bg-section` highlight behind it). The other three panels are *collapsed* (32 px slots with the normal drop shadow). Nothing is dimmed by default any more, exactly like the native bars.
- **Pressed state** shrinks the slot by 4 px and nudges it down, then swaps to the pressed border, matching `ActionBarStyles.lua`. Paddle actions now fire on press (`useOnKeyDown`), like native gamepad buttons.
- **Modifier prompts**: the LT, RT, and LT + RT panels show Blizzard's own `InputIconTextureFrameTemplate` / `InputPromptTwoIconTemplate` trigger glyphs beneath the slots. They follow the connected controller's glyph style and switch to the focused variant when their panel is active.
- **Paddle glyphs**: empty slots show Blizzard's `Gamepad_Gen_Paddle1-4` glyphs (the same art the key-binding UI uses for paddles). Assigned actions on the focused panel get a small paddle prompt in the corner, like the native button prompts.
- **Usable / range feedback**: icons tint blue when out of resources and grey when unusable, and native-storage slots show the native range dot via `ACTION_RANGE_CHECK_UPDATE`.
- **Native settings are respected**: the game's `GamepadShowActionBarScaling`, `GamepadShowActionBarHighlight`, and `GamepadShowActionBarButtonPrompts` CVars control scaling, the focus highlight, and the paddle prompts, in addition to the addon's own options.
- **Focus detection** now asks the client the same question the native crossbar asks (`GamepadMode.IsLeftModifierDown` / `IsRightModifierDown`, falling back to the `GAMEPADLEFTMOD` / `GAMEPADRIGHTMOD` bindings) and listens to the native crossbar modifier callback. The mapped-state reader remains as a last resort.

Every native atlas has a bundled fallback, so the addon still renders on builds that rename art. `/paddles diag` reports the detection method and any missing atlases.

The settings now live on a single page. Selecting a settings category that has subcategories makes the client rebuild its category list, which crashes Blizzard's gamepad smart-navigation cursor (`ScrollUtil.lua: attempt to call a nil value` in `IsSelected`). Diagnostics moved to a **Print Diagnostics** button on that page.

Default panel positions in 0.7.0 flanked the native crossbar (BASE and LT on the left, RT and LT + RT on the right); 0.7.1 replaces that layout, see above.

## 0.6.0 — Independent panels, Edit Mode integration, LT/RT fix

### Independent HUD pieces

The four controller layers are now four separate frames:

- **BASE**
- **LT**
- **RT**
- **LT + RT**

Each frame can be positioned independently. Their positions are stored separately in `PaddleSlotsDB.panelPositions`.

### WoW Edit Mode

PaddleSlots listens to Blizzard's native `EditMode.Enter` / `EditMode.Exit` lifecycle. When normal WoW Edit Mode opens, all four PaddleSlots panels become independently draggable and get an edit overlay. Leaving Edit Mode locks them again and stores their positions.

Blizzard does not expose a supported public API for third-party addons to become first-class Edit Mode systems, so PaddleSlots deliberately does **not** call the internal `EditModeManagerFrame:RegisterSystemFrame()` machinery. That avoids tainting Blizzard's Edit Mode while still making the panels movable when you use the normal Edit Mode UI.

You can also enable **Unlock panels outside Edit Mode** in PaddleSlots settings if you want to move them without opening Blizzard Edit Mode.

### LT / RT panel detection

The previous version had two problems:

1. Forever routes both LT and RT through the same native `InputFunctionBindingButton_PADTRIGGER` click target, so wrapping that frame cannot identify which physical trigger caused the click.
2. The old mapped-state reader assumed the same index base for `C_GamePad.ButtonBindingToIndex()` and the Lua `GamePadMappedState.buttons` table. On your client that made LT resolve to the wrong state entry.

0.6.0 removes the shared native-click wrapper and detects the mapped-state table's index base at runtime before reading LT/RT. Visual state now distinguishes:

- no trigger → **BASE**
- LT → **LT**
- RT → **RT**
- LT + RT → **LT + RT**

For combat-safe paddle rebinding, PaddleSlots also detects whether Forever maps LT and RT to distinct native modifier CVars (`GamePadEmulateShift`, `GamePadEmulateCtrl`, or `GamePadEmulateAlt`). When it does, PaddleSlots uses a secure state driver so the paddle bindings can switch layers during combat without replacing Blizzard's trigger bindings.

If the client does not expose both triggers as distinct secure modifiers, the addon still follows the mapped controller state and switches correctly outside combat. `/paddles diag` reports exactly which mode is active.

### Native gamepad action storage

The storage scanner now handles Forever's separate pet-action range correctly. In current builds the pet range can begin below the normal gamepad storage range (for example pet storage near 11 and general gamepad storage around 181); the old scanner incorrectly treated that as an upper boundary and immediately fell back to SavedVariables.

When a safe block of unused native gamepad action-storage slots is available, PaddleSlots uses it for new/empty profiles. If you already have actions stored in the SavedVariables fallback, the addon deliberately keeps that mode so an update never makes existing paddle assignments disappear. Otherwise it falls back whenever the native pool is unavailable or unsafe.

## Settings

Open:

**Settings → AddOns → PaddleSlots**

or use:

`/paddles`

Options include:

- Unlock panels outside Edit Mode
- Only show in gamepad mode (panels hide in mouse-and-keyboard mode, stay visible while unlocked or in Edit Mode)
- Paddle inputs (one row per paddle, click to assign by pressing), Assign P1-P4, Setup guide
- HUD scale
- Inactive panel opacity (1.00 = native, no fading)
- Focus highlight and strength
- LT / RT modifier icons (off by default; the native crossbar shows the same prompts)
- Paddle prompts on the focused panel
- Diagnostics

## Paddle inputs

On Windows the Xbox Elite Series 2 does not report its paddles to games. WoW only sees whatever the Xbox Accessories app (or Steam Input / reWASD) maps a paddle to, so PaddleSlots listens for a configurable input per paddle instead of assuming the native `PADPADDLE1-4` keys.

PaddleSlots never takes over a button the native gamepad UI uses. **Settings -> AddOns -> PaddleSlots -> Paddle inputs** shows one row per paddle whose button reads the current input; click it and press the paddle to assign a new one. Accepted inputs are only ones WoW does not use:

- physical keyboard keys with no WoW binding (F9-F12 on a compact keyboard; also F6-F8, Page Up/Down, Scroll Lock, Pause, or Numpad keys), for the Xbox Accessories app's keyboard key mapping,
- the Share button (unused by the native UI, if Xbox Accessories offers it for your controller),
- keyboard F13-F24, for Steam Input or reWASD sending keys that are not on the keyboard,
- the native paddle keys, for controllers that actually report paddles.

**Assign by pressing** opens a prompt and assigns each paddle to whatever the controller sends when you press it. If that is A, B, X, Y, a stick click, or any other native button, the prompt refuses it, explains that the controller profile is mirroring the paddle, and keeps waiting. Keyboard keys that already have a binding are refused too. **Setup guide** opens an in-game window with the step-by-step instructions for both routes and a live "last input detected" line.

Two routes:

1. **Xbox Accessories keyboard mapping**: map each paddle to a real key WoW leaves unbound (F9-F12 on a compact keyboard; also F6-F8, Page Up/Down, Scroll Lock, Pause, or Numpad keys), then use Assign by pressing. Share also works for one paddle if the app offers it.
2. **Steam Input or reWASD**: install Steam's Xbox Extended Feature Support driver (or use reWASD), map the paddles to F13-F16, leave them unassigned in Xbox Accessories, and select Keyboard F13-F16 in the addon. All four paddles work and the native layout stays intact. If a paddle press flips the UI to mouse-and-keyboard mode, pin the interface style to gamepad in the game's controls settings.

The client's own default config for the Elite Series 2 (vendor 1118, product 767) maps raw buttons 16-19 to PADPADDLE1-4, but the controller never sends them on Windows. `/paddles test` shows raw presses; `/paddles learn` is only useful for controllers whose paddles arrive on other raw indices.

## Usage

Drag a spell, item, macro, or supported action-bar action onto any paddle slot. Press P1-P4 to activate the corresponding action in the currently active controller layer.

## Slash commands

- `/paddles` — open native PaddleSlots settings.
- `/paddles unlock` — move panels outside WoW Edit Mode.
- `/paddles lock` — lock panels outside WoW Edit Mode.
- `/paddles reset` — reset all four panel positions to the nested crossbar layout.
- `/paddles reset <base|lt|rt|both>` — reset one panel.
- `/paddles clear <base|lt|rt|both> <1-4>` — clear one paddle action.
- `/paddles guide` — open the setup guide window.
- `/paddles assign [1-4]` — assign one paddle (or all four in order) by pressing it.
- `/paddles keys [P1 P2 P3 P4 | reset]` — show or set the paddle inputs, e.g. `/paddles keys F13 F14 F15 F16`.
- `/paddles diag` — print gamepad integration diagnostics, including the paddle inputs and which raw buttons carry PADPADDLE1-4.
- `/paddles test` — for 30 seconds, print the raw controller button index of anything you press and what the client maps it to. Use it to confirm the paddles reach WoW at all.
- `/paddles learn` — press P1, P2, P3, P4 in order. The addon writes a device config (vendor/product specific) through `C_GamePad.SetConfig` so those raw buttons become PADPADDLE1-4. The client stores it in `WTF/GamePadConfig_AddOns.json`. Learning refuses raw buttons the client already uses (A/B/X/Y, D-pad, ...), because a paddle arriving as one of those means the controller profile mirrors it, and rebinding it would take the button away from the native UI. `/paddles learn force` overrides that check. `/paddles learn cancel` aborts; `/paddles learn clear` deletes the addon's device config again (follow with `/reload`).
- `/paddles help` — print command help.

## Diagnostics

After installing 0.6.0, `/paddles diag` reports:

- native storage status and slots
- LT / RT bindings
- modifier-emulation CVars
- LT / RT mapped button indices and current values
- visual detection method and visual panel
- secure panel
- native style CVars and native art availability
- secure panel-driver mode
- Edit Mode integration state

The most useful lines when validating a Forever build are **Modifier CVars**, **Mapped state**, **Visual panel**, and **Secure panel**.
