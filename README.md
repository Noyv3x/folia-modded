# folia-modded

Notes and starting brief for getting a modded Minecraft server to tick in parallel regions the way Folia does. Nothing here runs yet. This file is what I hand to Devin as the first prompt, so it is written as instructions.

## What I want

A NeoForge server where far-apart groups of loaded chunks tick on different threads, and where Create still works. Create first because it is the mod my players care about and because it is close to a worst case: rotation networks, moving contraptions, trains, wireless redstone. If Create can be made to work, most tech mods can.

Later: Create Aeronautics, then storage and pipe mods. Not now.

## Why this is hard

Folia is a set of patches on top of Paper. It only runs plugins. Mods assume one server thread owns the whole world, and they keep a lot of state in static fields and per-level maps. Put that on several threads and you get races, not crashes, which is worse.

So there are two separate jobs. One is getting region ticking onto a mod loader at all. The other is fixing each mod. The first is bigger and I do not know yet if it is feasible for one person plus an agent. Finding that out is the point of the first tasks.

## How I think it should work

These are my assumptions. Tell me where they are wrong.

Folia keeps a gap of unloaded chunks between regions and merges regions before they touch. A shaft network only connects blocks that are next to each other, so if all of it is loaded it sits inside one region. That means I should not need any cross-thread sync for kinetics. No lock-free structures.

What I do need is for Create's network table to stop being one map per level. It should live in per-region data, the same way Folia's own RegionizedData works, with a merge callback that joins two tables and a split callback that hands each network to the region that owns its blocks. Both happen between ticks, so no locking there either.

Things that really do cross regions: redstone links, the train graph, a contraption entity moving from one region to another. For those I want message passing through the region scheduler and I accept one tick of delay. The train graph can live on the global tick.

Every level, chunk and block entity access should assert it is on the owning thread. I would rather the server crash loudly than corrupt a world.

## Tasks, in order

Do them one at a time and stop after each so I can read the result. The first three are research and should produce a written report in docs/, not code.

1. Read Folia's region code. It is under io.papermc.paper.threadedregions (ThreadedRegionizer, TickRegionScheduler, TickRegions, RegionizedData, RegionizedWorldData). List what it depends on outside that package, and mark which of those dependencies exist only in Paper. I am writing class names from memory, so check them.

2. Folia's regionizer sits on Paper's rewritten chunk system. That chunk system is also published as a standalone mod, Moonrise. Check whether Moonrise has a NeoForge build for the version we pick, and whether the regionizer could be built on top of it instead of on Paper. If yes, that is the route. If no, say what is missing.

3. Pick the Minecraft version. Create's main NeoForge target is 1.21.1 as far as I know. Find the closest Folia branch and tell me how far apart they are and which side should move.

4. Go through Create's source and list every piece of mutable state that is static or keyed by level. Start from the fields on the main Create class (torque propagator, redstone link handler, railway manager) and follow from there. For each one write down who reads it, who writes it, and whether it could be per-region, has to be global, or needs messages.

5. Only after 1 to 4: a minimal mod that does nothing but assert thread ownership on level access, run against vanilla NeoForge plus Create, to see how many places would break. This gives a number for how much work the Create side is.

6. Then a go or no-go from you with reasons. If it is a go, propose the first real milestone. My guess is a server that region-ticks vanilla content only, with no mods, on NeoForge.

## Rules

Do not add locks or concurrent collections to make a race go away. If two regions touch the same data, the design is wrong and I want to hear about it.

Do not copy Folia or Create code into this repo without checking the licenses first and telling me what they allow.

Keep reports short. If you are unsure whether something is true, say so instead of guessing.

## How I will test it later

Two rotation networks far apart, running in different regions. Load the chunks between them so the regions merge, connect the networks, disconnect, unload the middle so they split. Compare speed and stress every tick against the same build on a normal single-threaded server. Then leave it running for a day with deadlock detection on.
