---
layout: verc
title: The Power of CVARs!
page.author: Caleb 'Ghoul' Delnay
page.date: 2002-02-02 02:02am PST
page.categories: [VERC, half-life, coding]
permalink: /hl_modding_resources/collective.valve-erc.com/The Power of CVARs.html
---

#### The Power of CVARs!

> [Sat Feb 02, 2002 / 02:02am PST] Caleb 'Ghoul' Delnay

Lets face it, people like to customize things. They like to make things the way they want it. In Half-Life, thats where CVARs step in. CVARs are int varibles managed by the server, one CVAR is mp_teamplay.

This CVAR manages teamplay and allows server admins to change whether the game rules are teamplay or not. CVARs can control everything, like what weapons you spawn with, how fast you move, how high you jump, whatever you want. Here I'll give you the basics of setting up and using your own CVARs.

Now, boot up your copy of MSVC++ or whatever you use to edit the SDK and open the HL workspace. Take a look at **game.cpp**. You should see something like cvar_t blah blah. Lets make a CVAR that controls the amount of armor you spawn with, so somewhere towards the top of the file add in:

```cpp
cvar_t spawnarmoramt = {"mp_spawnarmoramt","50",FCVAR_SERVER };
```

This is one of the most important parts of a CVAR. The spawnarmoramt is the real CVARs name. The mp_spawnarmoramt is the console command to change the CVAR while in game. The 50 is the base value for the CVAR. Finally, the FCVAR_SERVER is... erm... I guess that says its a server CVAR, just leave it alone. Now you need to scroll way down until you get to the GameDLLInit function. Here you should see CVAR_REGISTER all over. Add in:

```cpp
CVAR_REGISTER (&spawnarmoramt;);
```

This will declare your CVAR. Now we need to bust open **multiplayer_gamerules.cpp** and take a look at the PlayerSpawn function. At the bottom of the function add in:

```cpp
pPlayer->pev->armorvalue = CVAR_GET_FLOAT("mp_spawnarmoramt");
```

This is where everything comes into play. The CVAR_GET_FLOAT function gets the value of the CVAR in quotes. So whatever mp_spawnarmoramt is, the players armor value (pPlayer->pev->armorvalue) will be set to that value. CVAR_GET_FLOAT value can also be used in if statements:

```cpp
if (CVAR_GET_FLOAT("mp_spawnarmoramt") > 0)
```

That would check mp_spawnarmoramt and if its higher than 0 then the if statement would be executed. There you have it, the overwelming power of CVARs. Muwhahahaha!

Alrighty, now for the **settings.scr**. This file controls all the Advanced Options in the Create Game menu for Internet and LAN games. Windows says its a screen saver file, but it isn't. Open up notepad and then browse over to it in your mod directory (copy and paste it there from the valve directory). Now, just take a look at some of the normal sections already in the file. Lets add in our armor cvar:

```
"mp_spawnarmoramt"
{
     "Player Spawns w/ X Armor"
     { NUMBER 0.000000 999.000000 }
     { "50.000000" }
}
```

The first line is the CVAR name, then an openening bracket, then the next line is its title on the Advanced Options page. The line after that is what the value is (a BOOL, STRING, NUMBER, or LIST) and its allowed range. The last line is the base value. Now add in those CVARs and allow admins around the world to customize their servers! Good day!
