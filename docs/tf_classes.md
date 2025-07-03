# Defining New TF Classes

This document explains the key steps to add a new playable class to the Team Fortress portion of the Source SDK 2013 code base.

## 1. Enumerate the class

Edit `src/game/shared/tf/tf_shareddefs.h` and extend the `ETFClass` enum. New values must be added **after** the existing ones to avoid breaking demos.

```cpp
enum ETFClass
{
    TF_CLASS_UNDEFINED = 0,
    // existing classes ...
    TF_CLASS_ENGINEER,
    TF_CLASS_CIVILIAN,
    TF_CLASS_COUNT_ALL,
    TF_CLASS_RANDOM,
    TF_CLASS_YOURCLASS      // add your class here
};
```

Update any helper macros such as `TF_LAST_NORMAL_CLASS` if the new class should be considered a normal playable class.

## 2. Provide class names

`tf_shareddefs.h` declares several arrays used throughout the code and UI. Append the name of your class to each array in the same order as the enum:

```cpp
extern const char *g_aPlayerClassNames[TF_CLASS_MENU_BUTTONS];
extern const char *g_aPlayerClassNames_NonLocalized[TF_CLASS_MENU_BUTTONS];
extern const char *g_aRawPlayerClassNames[TF_CLASS_MENU_BUTTONS];
```

Their definitions live in `src/game/shared/tf/tf_class_info.cpp` and must also be updated.

## 3. Describe class data

Gameplay properties such as health, speed and default loadout are defined by `TFPlayerClassData_t` in `src/game/shared/tf/tf_classdata.h`. Class data is parsed from `scripts/playerclasses.txt` at runtime. Add a new block in that script describing the attributes for your class.

Example snippet:

```txt
"YourClass"
{
    "model"     "models/yourclass.mdl"
    "health"    "100"
    "speed"     "300"
    // additional keys...
}
```

See the existing entries for reference.

## 4. Implement class behaviour

Create new `CTFYourClass` files derived from `CTFPlayerClassShared` (and the server/client specific subclasses). Implement class abilities, weapons and any networking required.

Register the class in the player spawning code so that it can be selected in game.

## 5. Generate placeholder assets with Codex

When prototyping, you may not have final art ready. You can instruct Codex to
replicate an existing class so the new one has usable models and HUD icons:

1. Duplicate `models/player/scout.mdl` (and its material files) to a new folder
   matching your class name, e.g., `models/player/yourclass`.
2. Copy the Scout HUD icons from
   `materials/vgui/hud/scout` and rename them to `yourclass`.
3. Edit `scripts/playerclasses.txt` so the `"model"` key for your class points
   to the duplicated model path.

These placeholder assets let you test functionality until custom content is
ready.

## 6. Update resources and UI

Provide icons, models and any HUD assets referenced by the script and code. Place them under `game/mod_tf` in the appropriate folders (e.g., `materials`, `models`, `resource`).

## 7. Compile and test

Rebuild the SDK projects and run the game with your mod to ensure the new class functions correctly.

