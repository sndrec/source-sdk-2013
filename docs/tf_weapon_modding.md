# Team Fortress Weapon Modding 101

This document explains how weapons are described and registered in the Source SDK's Team Fortress code base. It shows where the weapon enumeration lives, how the weapon info parser works, and outlines the general steps for adding your own new base weapons, inherited weapons, and projectile classes.

## Base weapon definitions

Weapons are enumerated in `tf_shareddefs.h` under `ETFWeaponType`. Every entry maps to an in‑game weapon ID. New weapons should be appended at the end of the enum to avoid breaking demos:

```c
enum ETFWeaponType
{
    TF_WEAPON_NONE = 0,
    TF_WEAPON_BAT,
    TF_WEAPON_BAT_WOOD,
    TF_WEAPON_BOTTLE,
    ...
    TF_WEAPON_FLAME_BALL,
    // ADD NEW WEAPONS HERE TO AVOID BREAKING DEMOS
    TF_WEAPON_COUNT
};
```

This enumeration can be seen in `tf_shareddefs.h` lines 404–522.

## Parsing weapon info

The parsing logic is implemented in `tf_weapon_parse.cpp`. `CTFWeaponInfo` extends `FileWeaponInfo_t` and reads attributes from KeyValues files. Primary and secondary fire parameters are loaded using `GetInt` and `GetFloat` calls, and the projectile type is resolved via `g_szProjectileNames`:

```c++
m_WeaponData[TF_WEAPON_PRIMARY_MODE].m_nDamage = pKeyValuesData->GetInt("Damage", 0);
m_WeaponData[TF_WEAPON_PRIMARY_MODE].m_flRange = pKeyValuesData->GetFloat("Range", 8192.0f);
...
const char *pszProjectileType = pKeyValuesData->GetString("ProjectileType", "projectile_none");
for (i = 0; i < TF_NUM_PROJECTILES; i++)
{
    if (FStrEq(pszProjectileType, g_szProjectileNames[i]))
    {
        m_WeaponData[TF_WEAPON_PRIMARY_MODE].m_iProjectile = i;
        break;
    }
}
```

See lines 61–93 of `tf_weapon_parse.cpp` for the full parser.

## Adding a new base weapon

1. **Enumerate the weapon.**  Add a new value at the end of `ETFWeaponType` in `tf_shareddefs.h` so it has a unique ID.
2. **Map the name and damage type.**  Append the weapon's enumeration name to the `g_aWeaponNames` array in `tf_shareddefs.cpp` and add a corresponding entry to `g_aWeaponDamageTypes`. The compile-time assertions in that file ensure the arrays stay in sync with `TF_WEAPON_COUNT`.
3. **Create a weapon script.**  Author a KeyValues file under `scripts/` describing the weapon's stats. The parser above will read properties such as `Damage`, `ProjectileType`, and `WeaponType`.
4. **Match the script filename to the entity name.**  The engine loads `<entity>.txt` from `scripts/`, where `<entity>` is the string passed to `LINK_ENTITY_TO_CLASS`. If they differ—e.g., `tf_weapon_rocketlauncher_sixclip` without `scripts/tf_weapon_rocketlauncher_sixclip.txt`—commands like `give` will fail and the item won't equip.
5. **Implement the C++ class.**  Derive a class from `CTFWeaponBaseGun` or another appropriate base. Use `DECLARE_CLASS`, `DECLARE_NETWORKCLASS`, and if server code is needed, `DECLARE_DATADESC`.
6. **Register the weapon entity.**  In the server file, use `LINK_ENTITY_TO_CLASS` and `PRECACHE_WEAPON_REGISTER` so the engine knows about your class. Example from `tf_projectile_flare.cpp`:

```c++
LINK_ENTITY_TO_CLASS( tf_projectile_flare, CTFProjectile_Flare );
PRECACHE_WEAPON_REGISTER( tf_projectile_flare );
```

7. **Update the item schema.** (usually `scripts/items/items_game.txt`). This defines lots of useful metadata used by the game.

   Example snippet:

   ```txt
    "32000"
    {
      "name"  "My Awesome New Rocket Launcer"
      "prefab"  "twilight"
      "item_class"  "tf_weapon_rocketlauncher_sixclip"
      "craft_class" "weapon"
      "craft_material_type" "weapon"
      "capabilities"
      {
        "nameable"    "1"
      }
      "tags"
      {
        "can_deal_damage"   "1"
        "can_deal_gib_damage" "1"
        "can_be_equipped_by_soldier_or_demo"  "1"
        "can_deal_posthumous_damage"  "1"
        "can_deal_critical_damage"  "1"
        "can_deal_long_distance_damage" "1"
      }
      "show_in_armory"  "1"
      "baseitem"      "0"
      "item_type_name"  "My Awesome New Rocket Launcer"
      "item_name" "My Awesome New Rocket Launcer"
      "item_slot" "primary"
      "item_quality"  "unique"
      "propername"  "1"
      "min_ilevel"  "1"
      "max_ilevel"  "1"
      "image_inventory" "backpack/weapons/w_models/w_rocketlauncher"
      "image_inventory_size_w"    "128"
      "image_inventory_size_h"    "82"
      "model_player"  "models/weapons/c_models/c_rocketlauncher/c_rocketlauncher.mdl"
      "attach_to_hands" "1"
      "used_by_classes"
      {
        "soldier" "1"
      }
      "static_attrs"
      {
        "min_viewmodel_offset"          "10 -3 -10"       
      }
      "attributes"
      {
        "reduced_healing_from_medics"
        {
          "attribute_class" "mult_healing_from_medics"
          "value" "0.1"
        }
      }
      "mouse_pressed_sound" "ui/item_heavy_gun_pickup.wav"
      "drop_sound"    "ui/item_heavy_gun_drop.wav"
    }
   ```

## Adding a new inherited weapon

If your new weapon doesn't have any notable special new behaviour, it might be possible to define it entirely within the item schema, without touching C++ or recompiling the game.
This process is described in step #7 of the guide to adding a new base weapon. Attributes can be added to weapons in the item schema, allowing you to define weapons with different behaviours and stats without necessarily having to define new base weapon types.

## Loading custom items

To make new weapons available in the in-game inventory so players can equip them, list their item definition indexes in `scripts/custom_item_ids.txt` inside your mod directory (e.g., `game/mod_tf/scripts/custom_item_ids.txt`). Each line should contain a single numeric index:

```txt
32000
32001
```

## Creating a projectile class

Projectiles typically inherit from `CTFBaseRocket` or `CTFProjectile_Scripted`. The server implementation includes the registration macros shown earlier. The header must declare networking macros so the object can replicate to clients:

```c++
class CTFProjectile_Flare : public CTFBaseRocket
{
    DECLARE_CLASS( CTFProjectile_Flare, CTFBaseRocket );
    DECLARE_NETWORKCLASS();
    DECLARE_DATADESC();
    ...
};
```

The corresponding source file defines the network table and links the entity:

```c++
IMPLEMENT_NETWORKCLASS_ALIASED( TFProjectile_Flare, DT_TFProjectile_Flare )
BEGIN_NETWORK_TABLE( CTFProjectile_Flare, DT_TFProjectile_Flare )
    SendPropBool( SENDINFO( m_bCritical ) ),
END_NETWORK_TABLE()

LINK_ENTITY_TO_CLASS( tf_projectile_flare, CTFProjectile_Flare );
```

Refer to `tf_projectile_flare.cpp` and `.h` for a full example.

## Attribute hooks

Custom item attributes can modify behavior at runtime through the `CALL_ATTRIB_HOOK_*` macros. These evaluate equipped attribute values and apply them to variables. For example, in `tf_weapon_rocketlauncher.cpp` the projectile type can be overridden:

```c++
int iProjectile = 0;
CALL_ATTRIB_HOOK_INT( iProjectile, override_projectile_type );
if (iProjectile == 0)
    iProjectile = GetWeaponProjectileType();
```

Projectiles also query attributes from their launcher, such as in `tf_projectile_flare.cpp`:

```c++
float flRadius = TF_FLARE_DET_RADIUS;
CALL_ATTRIB_HOOK_FLOAT_ON_OTHER( m_hLauncher, flRadius, mult_explosion_radius );
```

These hooks allow item definitions to dynamically alter stats without additional code changes.

See `tf_weapon_rocketlauncher.*` and `tf_projectile_flare.*` for reference implementations of a weapon that fires projectiles and a projectile class.

