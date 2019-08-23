---
layout: verc-page
title: Event System in SDK 2.2
page.author: Valve Software 
page.date: 2002-01-29 04:13pm PST
page.categories: [VERC, half-life, coding]
permalink: /hl_modding_resources/collective.valve-erc.com/Event System in SDK 2.2.html 
---
> [Tue Jan 29, 2002 / 04:13pm PST] Valve Software

> **HL SDK 2.2**
>
> This article was originally made available as part of version 2.2 of the Half-Life SDK. As such, all limitations, restrictions, license agreements, copyrights and trademarks from the original document apply here.

### Event System

One of the major additions to the SDK is the client event system. The event system allows a MOD to conserve network bandwidth by creating a shorthand code that is used to playback on the client an arbitrarily complex set of events, such as temporary entity creation, sound effects, animations, etc.

#### A. Creating the Event ( Server )

Events are created in the game .dll by calling ``PRECACHE_EVENT`` with the name of the event script ( .sc ) file, e.g.:

```cpp
m_usDoorGoUp = PRECACHE_EVENT( 1, "events/door/doorgoup.sc" );
```

The first parameter should always be 1 for now. The return value from precaching an event on the server is an unsigned short ( event tags will actually only range from 0 to 1023 ) which can then be used to "play back" the event from the server ( or client ) code. The above code simply associates the script file "events/door/doorgoup.sc" in the current server game directory with the variable m_usDoorGoUp which is an unsigned short. All precached events are sent to the client during the connection phase. If the client does not have the specified .sc file, then the server will auto-download that file to the client ( if server downloading is enabled, etc. ). For now, the script file contents are not actually used, instead the script file names are just aliases for the events. However, the SDK would allow a MOD author to create an event scripting language, script events in that language, and have the client.dll parse and handle the scripts ( from the .sc files ). This would be particularly useful in allowing rapid prototyping and tweaking of effects without having to hard code them into the client .dll.

#### B. Creating / Hooking the Event ( Client )

In order for the event to be handled by the client.dll, the client.dll must "hook" the named .sc file. This is accomplished in events.cpp of the client.dll source. For example:

```cpp
gEngfuncs.pfnHookEvent( "events/door/doorgoup.sc", EV_TFC_DoorGoUp );
```

This function specifies that when the server sends the unsigned short identifier that references the "events/door/doorgoup.sc" script file to the client, that the associated function ( ``EV_TFC_DoorGoUp`` ) is invoked. This function is where the MOD author would play sounds, etc. Each such event function is passed a generalized set of parameters including the origin and orientation of the event, the entity the event is associated with and, optionally, some user-specified parameters ( integer, float, and boolean, are supported, for examples ).

As examples and for reference, several of the Half-Life weapons have been converted to the event system, and the hooked playback functions for these events can be studied in ev_hldm.cpp in the client.dll source.

#### C. Playing back the Event ( Server side )

Playing back the Event:

The way to invoke an event from the game .dll is to invoke the ``PLAYBACK_EVENT`` or ``PLAYBACK_EVENT_FULL`` API macro or call. ``PLAYBACK_EVENT`` simply invokes ``PLAYBACK_EVENT_FULL`` with all default parameters:

```cpp
#define PLAYBACK_EVENT( flags, who, index ) \
        PLAYBACK_EVENT_FULL( flags, who, index, 0, \
        (float *)&g;_vecZero, \
        (float *)&g;_vecZero, \
        0.0, 0.0, 0, 0, 0, 0 );
```

The form for PLAYBACK_EVENT_FULL is:

```cpp
void PLAYBACK_EVENT_FULL( int flags, 
                          const edict_t *pInvoker, 
                          unsigned short eventindex, float delay, 
                          float *origin, float *angles, 
                          float fparam1, float fparam2, 
                          int iparam1, int iparam2, 
                          int bparam1, int bparam2 );
```

Valid flags can be found in event_flags.h in the common source directory and allow for skipping the local host, sending the event reliably - events can be dropped by default --, and updating rather than adding identical events for a particular entity in the queue. Events are generally associated with a particular entity. If so, then the origin and angles will be taken from that entity and don't need to be specified in the playback call. Otherwise, or as an override, you can specify origin and angles in the playback call. The eventindex is simply the value returned when you precached the event. The delay is the amount of time in seconds ( e.g., 0.1 seconds ) to queue up the event on the client before it is fired. The additional fields are for passing event-specific parameters in to the client.dll: two floating point parameters, two integers, and two boolean values. Thus, playing back an event is as simple as invoking PLAYBACK_EVENT_FULL:

```cpp
PLAYBACK_EVENT_FULL( 0, 
                     m_pPlayer->edict(), 
                     m_usGaussSpin, 
                     0.0, 
                     (float *)&g;_vecZero, 
                     (float *)&g;_vecZero, 
                     0.0, 0.0, 
                     110, 
                     0, 0, 0 );
```

#### D. Playing back the Event ( Client side )

You could always fill in the arguments structure and directly invoke an event from the client .dll, but there are other ways to queue up an event ( especially a delayed event ) from the client .dll. You can either use the event API call ``gEngfuncs.pEventAPI->EV_PlaybackEvent`` or the direct API call gEngfuncs.pfnPlaybackEvent. The prototype for both are as above for the server side call ( NOTE: you should use a NULL edict_t for the pInvoker, since the client doesn't use edict_t pointers ).

#### E. The Event API

Your client side event hook code can make use of any client.dll functionality. Particularly relevant functionality has been placed into the event API ( ``gEngfuncs.pEventAPI->`` ) calls available in the client .dll. These APIs allow for starting and stopping sounds, looking up model indices, getting local player viewheights ( eye position relative to player origin ), ducking status, bounding boxes, retrieving a client entity index from a client-side trace result, setting up and performing client-side tracelines against other players and world objects, setting the view model weapon animation, playing back events, and tracing textures. By far the most complex, and useful, set of functions here is the ray tracing / traceline functionality. A good example of how to use this can be found in ``EV_HLDM_FireBullets()`` in ev_hldm.cpp of the client.dll.

#### F. The Effects API

Your client side code ( event system or anywhere else in the client.dll ) can also make use of the effects API ( ``gEngfuncs.pEfxAPI->`` ) to generate various temporary entities, particle systems, etc. In addition, the effects APIs support the creation of custom client-side temporary entities and particle systems. Available API calls are listed in r_efx.h in the common source directory. Additional effects functionality includes creation of sprite effects, beam entities, and entity and dynamic lights on the client.
