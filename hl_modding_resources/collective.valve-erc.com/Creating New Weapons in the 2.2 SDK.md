> [Mon Jul 08, 2002 / 05:29pm PDT] Robert Prouse

### Creating New Weapons in the 2.2 SDK

There are plenty of tutorials out there on how to create new weapons, but, while we were working on our mod, Tour of Duty, we found that all of the tutorials were missing some steps that you must take, didn't explain how things like weapon accuracy are adjusted, and most importantly, none explained how to get client side prediction working in the 2.2 SDK. Judging by the number of posts I see on the mailing lists, client side events are causing a lot of problems.

I am assuming that you have the 2.2 Half-life SDK installed and you can compile both the client dll and the server dll. If not, then you better head out and read a few other tutorials first. And now for a few conventions:

Filenames are in green,
Original code is in blue,
Added code is in red.

To demonstrate everything you will need to add a new weapon to your mod, I am going to show you how we added an M16 to Tour of Duty. To keep things simple, I am going to simplify the weapon a bit here and try to work with the original 2.2 SDK.

Please note though that you cannot just drop this weapon directly into your mod, you will need to add models and sounds, or modify the code to use your models and sounds. I also may have forgotten some of the includes at the start of some of the files. Our mod is fairly different, so I had to improvise here and there. If this is the case, it should be pretty easy for you to figure out what is missing.

Okay, let's get at it...


#### Server Side Modifications

We will start working on the server dll, so open up the hl project and begin by adding a .cpp and .h file to the project for your weapon. These files should be in the dlls directory of the source, for this project, we will call them m16.cpp and m16.h. Open up m16.h and add the following code which will define the CM16 class;

``dlls\m16.h``

```cpp
#ifndef TOD_M16_H
#define TOD_M16_H

class CM16 : public CBasePlayerWeapon
{
public:
    virtual void Spawn( void );
    virtual void Precache( void );
    virtual int  iItemSlot( void ) { return M16_SLOT; }
    virtual int  GetItemInfo(ItemInfo *p);
    virtual int  AddToPlayer( CBasePlayer *pPlayer );
    
    virtual void PrimaryAttack( void );
    virtual BOOL Deploy( void );
    virtual void Reload( void );
    virtual void WeaponIdle( void );
    
    virtual BOOL UseDecrement( void ) { return TRUE; }

private:
    int  m_iShell;
    unsigned short m_event;
};

class CM16AmmoClip : public CBasePlayerAmmo
{
    virtual void Spawn( void );
    virtual void Precache( void );
    virtual BOOL AddAmmo( CBaseEntity *pOther ) ;
};
```

Notice that we have two classes here, CM16 and CM16AmmoClip. The first is for the weapon itself and the second is for ammo clips for the weapon. I am also using the SecondaryAttack method to switch the weapon between single shot and automatic fire. You can also use SecondaryAttack to do things like zoom in on sniper rifles or fire the weapon a different way, such as launching a grenade from an M203 mounted to the weapon.

```cpp
enum m16a1_e
{
    M16_IDLE_SIL = 0,
    M16_IDLE,
    M16_FIRE1,
    M16_FIRE2,
    M16_FIRE3,
    M16_RELOAD,
    M16_DEPLOY,
};
#endif
```

These are your models animations and must match those in your model. The names can be changed, but they must match the order of animations in the model. To check a weapon's animations, you can open it up in a program like **Half-Life Model Viewer** and view the animations. Remember, it is the order of the animations that is important, not what they are called in the model.

Now open m16.cpp and add the following code.

##### dlls\m16.cpp

```cpp
#include "extdll.h"
#include "util.h"
#include "cbase.h"
#include "monsters.h"
#include "weapons.h"
#include "nodes.h"
#include "player.h"
#include "soundent.h"
#include "gamerules.h"

#include "m16.h"
```

These are the standard includes for all weapons, plus I have added an include for our header file.

```cpp
LINK_ENTITY_TO_CLASS( weapon_m16, CM16 );
LINK_ENTITY_TO_CLASS( ammo_m16, CM16AmmoClip );
```

These link the weapon and ammo clip classes to entities that can be added to maps (with a modified FGD) or given to the player when they spawn.

```cpp
void CM16::Spawn( )
{
    pev->classname = MAKE_STRING("weapon_m16"); // hack to allow for old names
    Precache( );
    SET_MODEL(ENT(pev), M16_MODEL_WORLD);
    m_iId          = WEAPON_M16;    
    m_iDefaultAmmo = M16_DEFAULT_AMMO;
    FallInit();  // get ready to fall down.
}
```

This is our spawn function. Notice that I tend to use defines for most things that we may want to tweak later such as the models, the weapon id, etc. We will add these defines to weapons.h later.

```cpp
void CM16::Precache( void )
{
    PRECACHE_MODEL(M16_MODEL_1STPERSON);
    PRECACHE_MODEL(M16_MODEL_3RDPERSON);
    PRECACHE_MODEL(M16_MODEL_WORLD);
    
    m_iShell = PRECACHE_MODEL ("models/shell.mdl");// brass shell
           
    PRECACHE_SOUND (M16_SOUND_FIRE1);
    PRECACHE_SOUND (M16_SOUND_FIRE2);   
    
    m_event = PRECACHE_EVENT( 1, "events/m16.sc" );
}
```

This is where we load any of the models or sounds that our weapon uses. All of these must exist in your mod's directory or you can use ones in the valve directory. Also notice the "events/m16.sc", this allows us to link this weapon to a client side event. More on this later.

```cpp
int CM16::GetItemInfo(ItemInfo *p)
{
    p->pszName   = STRING(pev->classname);
    p->pszAmmo1  = "ammo_m16";              // The type of ammo it uses
    p->iMaxAmmo1 = M16_MAX_AMMO;            // Max ammo the player can carry
    p->pszAmmo2  = NULL;                    // No secondary ammo
    p->iMaxAmmo2 = -1;
    p->iMaxClip  = M16_DEFAULT_AMMO;        // The clip size
    p->iSlot     = M16_SLOT - 1;            // The number in the HUD
    p->iPosition = M16_POSITION;            // The position in a HUD slot
    p->iFlags    = ITEM_FLAG_NOAUTOSWITCHEMPTY | ITEM_FLAG_SELECTONEMPTY;
    p->iId       = m_iId = WEAPON_M16;      // The weapon id
    p->iWeight   = M16_WEIGHT;              // for autoswitching
    
    return 1;
}
```

Here we set the information for the weapon. The secondary ammo has been set to NULL and -1 because the M16 does not have a secondary ammo type (like M203 grenades). Notice that the iSlot is zero based here, but in the method iItemSlot() in the header, it is 1 based. There are five slots and five positions under each slot. Every weapon in your mod (including any half-life weapons you leave in) must have a unique slot/position combination, and must have a unique id.

```cpp
int CM16::AddToPlayer( CBasePlayer *pPlayer )
{
    if ( CBasePlayerWeapon::AddToPlayer( pPlayer ) )
    {
          MESSAGE_BEGIN( MSG_ONE, gmsgWeapPickup, NULL, pPlayer->pev );
          WRITE_BYTE( m_iId );
          MESSAGE_END();
          return TRUE;
    }
    return FALSE;
}

BOOL CM16::Deploy( )
{
    return DefaultDeploy( M16_MODEL_1STPERSON, M16_MODEL_3RDPERSON, 
                          M16_DEPLOY, "mp5" );
}
```

There isn't anything you will want to change in AddToPlayer. The only interesting things in Deploy are the last two parameters. M16_DEPLOY is the deploy animation and "mp5" is the series of animations in the player model that this weapon will use. Valid values for standard HL player models are; "crowbar", "trip", "onehanded", "python", "shotgun", "guass", "mp5", "rpg", "egon", "squeak", "hive" and "bow". Choose one that suits your weapon.

```cpp
void CM16::PrimaryAttack()
{
    // don't fire underwater
    if (m_pPlayer->pev->waterlevel == 3)
    {
          PlayEmptySound( );
          m_flNextPrimaryAttack = 0.15;
          return;
    }
    
    // don't fire if empty
    if (m_iClip <= 0)
    {
          PlayEmptySound();
          m_flNextPrimaryAttack = 0.15;
          return;
    }

    // Weapon sound
    m_pPlayer->m_iWeaponVolume = NORMAL_GUN_VOLUME;
    m_pPlayer->m_iWeaponFlash  = NORMAL_GUN_FLASH;

    // one less round in the clip
    m_iClip--;
    
    // add a muzzle flash
    m_pPlayer->pev->effects = (int)(m_pPlayer->pev->effects) | EF_MUZZLEFLASH;
    
    // player "shoot" animation
    m_pPlayer->SetAnimation( PLAYER_ATTACK1 );
    
    // fire off a round
    Vector vecSrc(m_pPlayer->GetGunPosition());
    Vector vecAim(m_pPlayer->GetAutoaimVector(AUTOAIM_2DEGREES));    
    Vector vecAcc(VECTOR_CONE_6DEGREES);
    Vector vecDir(m_pPlayer->FireBulletsPlayer( 1,           // how many shots 
                                                vecSrc,      
                                                vecAim,      
                                                vecAcc,      // accuracy
                                                8192,        // max range
                                                BULLET_PLAYER_M16, // bullet type
                                                0,           // tracer frequency
                                                0,           // damage
                                                m_pPlayer->pev, 
                                                m_pPlayer->random_seed ));
    
    // Fire off the client side event
    PLAYBACK_EVENT_FULL( FEV_NOTHOST, m_pPlayer->edict(), m_event, 0.0, 
                         (float *)&g;_vecZero, (float *)&g;_vecZero, 
                         vecDir.x, vecDir.y, 0, 0, (m_iClip ? 0 : 1), 0 );
    
    // Add a delay before the player can fire the next shot
    m_flNextPrimaryAttack = UTIL_WeaponTimeBase() + M16_FIRE_DELAY;    
    m_flTimeWeaponIdle    = UTIL_WeaponTimeBase() + 
                            UTIL_SharedRandomFloat(m_pPlayer->random_seed, 
                                        M16_FIRE_DELAY + 1, M16_FIRE_DELAY + 2);
}
```

This is the meat of the weapon, it fires off one round when the player presses the left mouse button. Most of this is explained by the comments in the code. If you want to make the weapon more accurate, make the number of degrees of accuracy smaller. If you set tracer frequency to anything other than 0, the weapon will fire tracers. If you set damage to 0, the weapon will use the default damage for the bullet type.

```cpp
void CM16::Reload( void )
{
    DefaultReload( M16_DEFAULT_AMMO, M16_RELOAD, M16_RELOAD_TIME );
}

void CM16::WeaponIdle( void )
{
    ResetEmptySound( );
    
    m_pPlayer->GetAutoaimVector( AUTOAIM_5DEGREES );
    
    if (m_flTimeWeaponIdle > UTIL_WeaponTimeBase())
          return;
        
    SendWeaponAnim( M16_IDLE );
    
    m_flTimeWeaponIdle = UTIL_SharedRandomFloat(m_pPlayer->random_seed, 10, 15);
}

void CM16AmmoClip::Spawn( void )
{ 
    Precache( );
    SET_MODEL(ENT(pev), "models/w_9mmARclip.mdl");
    CBasePlayerAmmo::Spawn( );
}

void CM16AmmoClip::Precache( void )
{
    PRECACHE_MODEL ("models/w_9mmARclip.mdl");
    PRECACHE_SOUND("items/9mmclip1.wav");
}

BOOL CM16AmmoClip::AddAmmo( CBaseEntity *pOther ) 
{ 
    int bResult = (pOther->GiveAmmo(M16_DEFAULT_AMMO, "ammo_m16", 
                                    M16_MAX_AMMO) != -1);
    if (bResult)
    {
        EMIT_SOUND(ENT(pev), CHAN_ITEM, "items/9mmclip1.wav", 1, ATTN_NORM);
    }
    return bResult;
}
```

These methods round out our weapon. Now lets get on to weapons.h and add those defines I was talking about.

##### dlls\weapons.h

```cpp
#define WEAPON_HANDGRENADE    12
#define WEAPON_TRIPMINE       13
#define WEAPON_SATCHEL        14
#define WEAPON_SNARK          15


#define WEAPON_M16            16
```

This must be a unique value.

```cpp
// weapon weight factors (for auto-switching)   (-1 = noswitch)
#define CROWBAR_WEIGHT    0
#define GLOCK_WEIGHT     10
#define PYTHON_WEIGHT    15
#define MP5_WEIGHT       15


#define M16_WEIGHT       15
```

Our M16 will have the same switching weight as the MP5

```cpp
#define M16_MODEL_1STPERSON "models/v_m16a1.mdl" 
#define M16_MODEL_3RDPERSON "models/p_m16a1.mdl" 
#define M16_MODEL_WORLD     "models/w_m16a1.mdl" 
#define M16_SOUND_FIRE1     "weapons/m16a1_fire-1.wav" 
#define M16_SOUND_FIRE2     "weapons/m16a1_fire-2.wav" 
#define M16_SOUND_VOLUME    0.85 
#define M16_FIRE_DELAY      0.085 // was: .15 For comparison, glock's is 0.2 
#define M16_RELOAD_TIME     2.0
#define M16_DEFAULT_AMMO    30 
#define M16_MAX_AMMO        180
#define M16_SLOT            2
#define M16_POSITION        1
```

Add this in with the other defines in weapons.h and adjust the slot and position to an empty slot/postion. Change the sounds and models to your own, or use HL models and sounds.

```cpp
// bullet types
typedef    enum
{
    BULLET_NONE = 0,
    BULLET_PLAYER_9MM, // glock
    BULLET_PLAYER_MP5, // mp5
    BULLET_PLAYER_357, // python
    BULLET_PLAYER_BUCKSHOT, // shotgun
    BULLET_PLAYER_CROWBAR, // crowbar swipe


    BULLET_PLAYER_M16,
  

    BULLET_MONSTER_9MM,
    BULLET_MONSTER_MP5,
    BULLET_MONSTER_12MM,
} Bullet;
```

This adds our new bullet type to the game. Now lets set up the damage this bullet does. Open up combat.cpp and find the function CBaseEntity::FireBulletsPlayer. Now scroll down a little bit further and add the following line;

##### dlls\combat.cpp

```cpp
else switch(iBulletType)
{
    default:
    case BULLET_PLAYER_9MM:        
        pEntity->TraceAttack(pevAttacker, gSkillData.plrDmg9MM, 
                             vecDir, &tr;, DMG_BULLET); 
        break;
        
    case BULLET_PLAYER_MP5:


    case BULLET_PLAYER_M16:
        pEntity->TraceAttack(pevAttacker, gSkillData.plrDmgMP5, 
                             vecDir, &tr;, DMG_BULLET); 
        break;
```

This gives the M16 the same damage as the MP5. If you want it to have it's own damage, you will have to create your own damage variable like gSkillData.plrDmgMP5. Just do a search for plrDmgMP5 and everywhere you find it, add in code for a plrDmgM16.

Now we need to do some tedious work to allow your weapon to be used in the game. Most of it doesn't require much thought, so let's just do it. Start by opening func_break.cpp

##### dlls\func_break.cpp

```cpp
const char *CBreakable::pSpawnObjects[] =
{
    NULL,                // 0
    "item_battery",        // 1
    "item_healthkit",    // 2
    "weapon_9mmhandgun",// 3
    "ammo_9mmclip",        // 4
    "weapon_9mmAR",        // 5
    "ammo_9mmAR",        // 6
    "ammo_ARgrenades",    // 7
    "weapon_shotgun",    // 8
    "ammo_buckshot",    // 9
    "weapon_crossbow",    // 10
    "ammo_crossbow",    // 11
    "weapon_357",        // 12
    "ammo_357",            // 13
    "weapon_rpg",        // 14
    "ammo_rpgclip",        // 15
    "ammo_gaussclip",    // 16
    "weapon_handgrenade",// 17
    "weapon_tripmine",    // 18
    "weapon_satchel",    // 19
    "weapon_snark",        // 20
    "weapon_hornetgun",    // 21


    "weapon_m16",
    "ammo_m16",


};
```

And now, move on to weapons.cpp to precache our weapon;

##### dlls\weapons.cpp

```cpp
// called by worldspawn
void W_Precache(void)
{
    memset(CBasePlayerItem::ItemInfoArray, 0, 
           sizeof(CBasePlayerItem::ItemInfoArray));
    memset(CBasePlayerItem::AmmoInfoArray, 0, 
           sizeof(CBasePlayerItem::AmmoInfoArray));
    giAmmoIndex = 0;

    // custom items...


    // m16
    UTIL_PrecacheOtherWeapon( "weapon_m16" );
    UTIL_PrecacheOther( "ammo_m16" );
```

Now, if you want to automatically give your weapon to a player in a game, you can do something like this... Open up multiplay_gamerules.cpp and give the player the weapon when they spawn.

##### dlls\multiplay_gamerules.cpp

```cpp
void CHalfLifeMultiplay :: PlayerSpawn( CBasePlayer *pPlayer )
{
    BOOL        addDefault;
    CBaseEntity    *pWeaponEntity = NULL;

    pPlayer->pev->weapons |= (1<Touch( pPlayer );
        addDefault = FALSE;
    }

    if ( addDefault )
    {
        pPlayer->GiveNamedItem( "weapon_crowbar" );
        pPlayer->GiveNamedItem( "weapon_9mmhandgun" );
        pPlayer->GiveAmmo( 68, "9mm", _9MM_MAX_CARRY );// 4 full reloads


        pPlayer->GiveNamedItem( "weapon_m16" );
        pPlayer->GiveAmmo( M16_MAX_AMMO, "ammo_m16", M16_MAX_AMMO ); 


    }
}
```

#### Client Side Modifications

That's it for server side changes, now lets move on to the client. Open up the cl_dll workspace and add in the m16.h and m16.cpp files from the server project. Now create the script file in your mod's events directory. This should just be an empty file called m16.sc. While you are at it, you will also want to add sprites and a hud text file to the sprites directory. These are the sprites that appear at the top of the screen when you select the weapon. There are a few good tutorials on how to do that, so I will leave that to them...

Now it is time to add in the client side events. Open up ev_hldm.cpp and add an include for m16.h to the top after the other includes.

##### cl_dll\ev_hldm.cpp

```cpp
#include "m16.h";
```

Next we will add the following line near the top.

##### cl_dll\ev_hldm.cpp

```cpp
void EV_TripmineFire( struct event_args_s *args  );
void EV_SnarkFire( struct event_args_s *args  );


void EV_FireM16( struct event_args_s *args  );
```

Then add the following code at the bottom;

```cpp
//======================
//        M16 START
//======================
void EV_FireM16( event_args_t *args )
{
    int idx;
    vec3_t origin;
    vec3_t angles;
    vec3_t velocity;

    vec3_t ShellVelocity;
    vec3_t ShellOrigin;
    int shell;
    vec3_t vecSrc, vecAiming;
    vec3_t up, right, forward;
    float flSpread = 0.01;

    idx = args->entindex;
    VectorCopy( args->origin, origin );
    VectorCopy( args->angles, angles );
    VectorCopy( args->velocity, velocity );

    AngleVectors( angles, forward, right, up );
    
    if ( EV_IsLocal( idx ) )
    {
        // Add muzzle flash to current weapon model
        EV_MuzzleFlash();
        gEngfuncs.pEventAPI->EV_WeaponAnimation(M16_FIRE1 + 
                                                gEngfuncs.pfnRandomLong(0,2), 2);

        // This gives it a bit of kick, adjust the numbers to your liking
        V_PunchAxis( 0, gEngfuncs.pfnRandomFloat( -2, 2 ) );
    }

    // Eject shells, adjust the last three numbers to move the start point that the
    // shells eject from to fit your model
    shell = gEngfuncs.pEventAPI->EV_FindModelIndex ("models/shell.mdl"); 
    EV_GetDefaultShellInfo( args, origin, velocity, ShellVelocity, 
                            ShellOrigin, forward, right, up, 12, -10, 7 );
    EV_EjectBrass(ShellOrigin, 
                  ShellVelocity, 
                  angles[ YAW ], 
                  shell, 
                  TE_BOUNCE_SHELL); 

    // Play the fire sound
    switch( gEngfuncs.pfnRandomLong( 0, 1 ) )
    {
    case 0:
        gEngfuncs.pEventAPI->EV_PlaySound( idx, origin, CHAN_WEAPON, 
                                       "weapons/m16a1_fire-1.wav", 
                                       1, ATTN_NORM, 0, 
                                       94 + gEngfuncs.pfnRandomLong( 0, 0xf ) );
        break;
    case 1:
        gEngfuncs.pEventAPI->EV_PlaySound( idx, origin, CHAN_WEAPON, 
                                       "weapons/m16a1_fire-2.wav", 
                                       1, ATTN_NORM, 0, 
                                       94 + gEngfuncs.pfnRandomLong( 0, 0xf ) );
        break;
    }

    // Fire off the bullets client side
    EV_GetGunPosition( args, vecSrc, origin );
    VectorCopy( forward, vecAiming );
    EV_HLDM_FireBullets( idx, 
                       forward, 
                       right, 
                       up, 
                       1, 
                       vecSrc, 
                       vecAiming, 
                       8192, 
                       BULLET_PLAYER_M16, 
                       0, 
                       0, 
                       args->fparam1,     // These are the accuracy passed from
                       args->fparam2 );   // PLAYBACK_EVENT_FULL on the server
}
```

Most of this is self explanatory. Notice the similarities between the EV_HLDM_FireBullets() call on the client side and the m_pPlayer->FireBulletsPlayer() call on the server side.

Now we need to add in the event hooks. Open up hl_events.cpp and add;

##### cl_dll\hl\hl_events.cpp

```cpp
void EV_HornetGunFire( struct event_args_s *args );
void EV_SnarkFire( struct event_args_s *args );


void EV_FireM16( struct event_args_s *args  );
```

Then further down;

```cpp
void Game_HookEvents( void )
{
    gEngfuncs.pfnHookEvent( "events/python.sc",       EV_FirePython );
    gEngfuncs.pfnHookEvent( "events/gauss.sc",        EV_FireGauss );
    gEngfuncs.pfnHookEvent( "events/gaussspin.sc",    EV_SpinGauss );
    gEngfuncs.pfnHookEvent( "events/crossbow1.sc",    EV_FireCrossbow );
    gEngfuncs.pfnHookEvent( "events/crossbow2.sc",    EV_FireCrossbow2 );
    gEngfuncs.pfnHookEvent( "events/shotgun1.sc",     EV_FireShotGunSingle );
    gEngfuncs.pfnHookEvent( "events/shotgun2.sc",     EV_FireShotGunDouble );
    gEngfuncs.pfnHookEvent( "events/egon_fire.sc",    EV_EgonFire );
    gEngfuncs.pfnHookEvent( "events/egon_stop.sc",    EV_EgonStop );
    gEngfuncs.pfnHookEvent( "events/firehornet.sc",   EV_HornetGunFire );
    gEngfuncs.pfnHookEvent( "events/snarkfire.sc",    EV_SnarkFire );
    gEngfuncs.pfnHookEvent( "events/rpg.sc",          EV_FireRpg );
    gEngfuncs.pfnHookEvent( "events/tripfire.sc",     EV_TripmineFire );


    gEngfuncs.pfnHookEvent( "events/m16.sc",          EV_FireM16 );
```

Tired yet? Okay, we're almost done. Open up hl_weapons.cpp and add an include for your weapon after all the other includes.

##### cl_dll\hl\hl_weapons.cpp

```cpp
#include "extdll.h"
#include "util.h"
#include "cbase.h"
#include "monsters.h"
#include "weapons.h"
#include "nodes.h"
#include "player.h"

#include "usercmd.h"
#include "entity_state.h"
#include "demo_api.h"
#include "pm_defs.h"
#include "event_api.h"
#include "r_efx.h"

#include "../hud_iface.h"
#include "../com_weapons.h"
#include "../demo.h"


#include "m16.h"
```

Then add a global for your weapon near the top with the rest of the weapons.

```cpp
CTripmine g_Tripmine;
CSqueak g_Snark;


CM16        g_M16;
```

Add a HUD_PrepEntity for the weapon in HUD_InitClientWeapons().

```cpp
HUD_PrepEntity( &g;_Tripmine , &player; );
HUD_PrepEntity( &g;_Snark    , &player; );


HUD_PrepEntity( &g;_M16      , &player; );
```

Lastly, add your weapon to the switch statement in HUD_WeaponsPostThink()

```cpp
switch ( from->client.m_iId )
{
    case WEAPON_CROWBAR:
        pWeapon = &g;_Crowbar;
        break;
  
    case WEAPON_GLOCK:
        pWeapon = &g;_Glock;
        break;


    case WEAPON_M16:
        pWeapon = &g;_M16;
        break;
```

Whew, that's it! You should now have a functioning weapon. Compile the client and server dll's and you should be on your way.
