# LAST ZONE: BATTLE SURVIVAL — Production Blueprint

*Version 1.0 · Numbers marked "target" or "est." are planning values to be tuned by playtesting and profiling, not measured results.*

---

## 1. Vision and pillars

**One-line pitch:** a fast, mobile-first 3D battle royale where 50 players parachute onto an island, loot, fight, and outlast a shrinking zone in ~12–15 minutes.

| Pillar | What it means in practice |
|---|---|
| Fast matches | Drop-to-first-fight under 60 s; total match 12–15 min |
| Fair | Server-authoritative; cosmetics only, no pay-to-win |
| Readable on a phone | Big touch targets, clear audio cues, minimap and compass |
| Runs on mid-range Android | 30 fps floor on 3 GB RAM devices, 60 fps option on strong devices |
| Original | All characters, maps, weapons, sounds and UI created in-house or properly licensed |

**Target platforms:** Android first (API 24+), PC (Windows) for testing/optional release, iOS later.

---

## 2. Where the project stands today

| Area | Web prototype (done) | Production (to build) |
|---|---|---|
| Engine | Three.js in one HTML page | Unity + C# |
| Players | 1 human + 7–15 local bots | 50 real players + backfill bots (clearly labeled) |
| Map | Flat 300 m island, generated | Hand-authored terrain, ~1.5 × 1.5 km |
| Weapons | 5 (pistol, SMG, rifle, shotgun, sniper) | 15–20 with attachments |
| Character | Your Player.blend, leg/arm swing | Rigged, full animation set |
| Networking | None | Dedicated authoritative servers |
| Accounts | None | Guest + linked accounts |
| Not built | Interiors, vehicles, sound, settings, lobby/party, ranked | All of these |

**What carries over:** weapon stats as a starting balance, zone timing structure, control layout, HUD layout, bot behavior states, and the character model (needs rigging).

---

## 3. Technology decisions

| Layer | Recommendation | Alternatives | Why |
|---|---|---|---|
| Engine | Unity (current LTS), URP render pipeline, C# | Unreal Engine 5 | Best mobile tooling and Android market fit; smaller build sizes |
| Netcode | Dedicated-server model. Evaluate **Photon Fusion** vs **Netcode for GameObjects** vs **Mirror** in a 50-player load test before committing | Netcode for Entities (DOTS) | 50 players with interest management is the hard requirement; pick by measurement |
| Game servers | Headless Unity dedicated-server builds, one process per match | — | Same code as client simulation |
| Orchestration | Agones on Kubernetes, or a managed fleet service (e.g. Unity Multiplay, AWS GameLift) | Self-managed VMs | Auto-scale match servers by demand |
| Backend | Nakama or PlayFab (accounts, storage, matchmaking, friends) | Custom Node/Go + Postgres | Avoids building auth/social from scratch |
| Database | PostgreSQL (profiles, inventory), Redis (sessions, queues) | — | Standard, well supported |
| Analytics/crash | Unity Analytics or Firebase; Crashlytics | — | Needed for live tuning |
| Source control | Git + Git LFS (or Plastic SCM) | — | Large binary assets |
| CI/CD | GitHub Actions or Unity Build Automation → Android AAB | — | Repeatable builds and tests |

> Requirement to confirm before building: pricing, player limits, and licensing for any hosted service. Nothing here should be assumed available until it is configured and tested.

---

## 4. System architecture

```
 ┌────────────┐      HTTPS       ┌───────────────────────────┐
 │  Android/  │ ───────────────► │  Backend services         │
 │  PC client │  auth, profile,  │  Auth · Profile · Friends │
 │  (Unity)   │  friends, queue  │  Party · Matchmaker       │
 └─────┬──────┘                  │  Inventory · Store        │
       │                         └────────┬──────────────────┘
       │ UDP (game traffic)               │ allocate server
       │                         ┌────────▼──────────────────┐
       └───────────────────────► │  Fleet orchestrator       │
                                 │  (Agones / GameLift)      │
                                 └────────┬──────────────────┘
                                          │ spawns
                                 ┌────────▼──────────────────┐
                                 │  Match server (headless)  │
                                 │  Authoritative simulation │
                                 │  30 Hz tick               │
                                 └────────┬──────────────────┘
                                          │ results
                                 ┌────────▼──────────────────┐
                                 │  Postgres · Redis · Logs  │
                                 └───────────────────────────┘
```

**Separation rules:** game logic (simulation) never calls the backend directly; the match server reports results to a results service; the client never decides hits, damage, loot, or zone state.

---

## 5. Unity project structure

```
LastZone/
├─ Assets/
│  ├─ _Project/
│  │  ├─ Core/            # bootstrap, state machine, service locator, config
│  │  ├─ Gameplay/
│  │  │  ├─ Player/       # controller, camera, stamina, health, armor
│  │  │  ├─ Weapons/      # WeaponDef (ScriptableObject), firing, recoil, attachments
│  │  │  ├─ Loot/         # LootTable, spawners, pickups, inventory
│  │  │  ├─ Zone/         # ZoneSchedule, damage, visuals
│  │  │  ├─ Drop/         # plane path, freefall, parachute
│  │  │  ├─ Bots/         # AI states, perception, difficulty
│  │  │  └─ Vehicles/     # later phase
│  │  ├─ Net/             # transport, snapshots, prediction, lag comp, relevancy
│  │  ├─ Services/        # backend clients: Auth, Profile, Party, Matchmaking
│  │  ├─ UI/              # screens, HUD, minimap, touch controls, settings
│  │  ├─ Audio/           # mixer, banks, footstep system
│  │  ├─ VFX/             # muzzle, impacts, explosions (pooled)
│  │  ├─ Art/             # Characters, Weapons, Environment, Materials, Textures
│  │  ├─ Data/            # ScriptableObjects: weapons, items, loot, zone, difficulty
│  │  └─ Scenes/          # Boot, Menu, Lobby, Island, Training
│  ├─ Plugins/
│  └─ Tests/              # EditMode + PlayMode tests
├─ Server/                # dedicated-server build config, Dockerfile, fleet config
├─ Backend/               # cloud functions or service code, DB migrations
├─ Docs/                  # design docs, balance sheets, asset license log
└─ Tools/                 # editor scripts, build scripts, load-test bots
```

---

## 6. Gameplay systems specification

### 6.1 Match flow (state machine)
`Lobby → Countdown → Deployment (plane) → Drop → Playing (zone stages) → Final circle → Ended → Results`
- Minimum start: 20 real players or 60 s timer, then backfill with bots (labeled "BOT" on the scoreboard and kill feed).

### 6.2 Player controller
| Property | Target value |
|---|---|
| Walk / run / crouch / prone speed | 3.5 / 5.5 / 2 / 1 m/s |
| Sprint | 7.5 m/s, drains stamina (5 s full, 8 s regen) |
| Jump | 1.1 m apex; vault low cover |
| Health / armor | 100 / up to 100 (3 armor tiers; helmets reduce headshot damage) |
| Freefall / parachute fall | ~55 m/s / ~7 m/s, auto-open at 60 m |
| Camera | Third-person over-shoulder; optional first-person aim-down-sights |

### 6.3 Weapons (data-driven; starting values)
| Weapon | Damage | Rate (rounds/s) | Mag | Reload | Range (effective) | Notes |
|---|---|---|---|---|---|---|
| Pistol | 22 | 4 | 12 | 1.1 s | 40 m | Spawn weapon |
| SMG | 9 | 16 | 35 | 1.5 s | 30 m | High recoil |
| Assault rifle | 15 | 10 | 30 | 1.7 s | 70 m | Attachments: scope, grip, stock, mag |
| Shotgun | 8 × 9 pellets | 1.2 | 6 | 2.1 s | 15 m | Falloff after 10 m |
| Sniper | 70 | 0.8 | 5 | 2.4 s | 200 m | 4×/8× scope; bullet travel time |
| Melee | 35 | 1.5 | — | — | 1.5 m | Fallback |
| Grenade | 90 center | — | — | 3 s fuse | 5 m radius | Frag, smoke, flash |

Each `WeaponDef` ScriptableObject holds: damage curve vs distance, headshot multiplier, fire rate, spread (hip/ADS/moving), recoil pattern, reload time, mag size, ammo type, attachment slots, sounds, VFX. All values live in data, never in code.

### 6.4 Damage model
`final = base × distanceFalloff × hitZoneMultiplier` → armor absorbs 50% until depleted (higher tiers absorb more) → health. Head 2.0×, torso 1.0×, limbs 0.75×. Server-only.

### 6.5 Loot
- Rarity: common 60%, uncommon 25%, rare 10%, epic 4%, legendary 1% (weights per zone).
- High-risk areas (military base, factory) use tables with higher rare weights.
- Spawn points are authored on the map; the server rolls items at match start, so no two matches are identical.
- Supply drops: 3 per match, timed with zone stages, marked on the map, contain top-tier gear.

### 6.6 Safe zone schedule (50-player target)
| Stage | Wait | Shrink | End radius | Damage/s |
|---|---|---|---|---|
| 1 | 120 s | 90 s | 65% | 1 |
| 2 | 90 s | 75 s | 40% | 2 |
| 3 | 75 s | 60 s | 22% | 4 |
| 4 | 60 s | 45 s | 10% | 7 |
| 5 | 45 s | 30 s | 3% | 12 |

The next circle center is chosen server-side inside the previous one and shown to all players when the wait begins.

### 6.7 Bots
Behavior states: `Land → Loot → Roam → Rotate to zone → Investigate sound → Engage → Flee/Heal`. Perception uses view cone + hearing (footsteps, gunfire). Difficulty controls reaction time, accuracy, target-switching, and loot quality (Easy/Normal/Hard as in the prototype). Bots follow the same damage rules as players and are always labeled.

### 6.8 Vehicles (later phase)
Jeep and bike first. Wheel-collider physics on the server with client interpolation; fuel, damage states, 4 seats, roadkill damage by speed.

---

## 7. Networking design

| Topic | Design |
|---|---|
| Authority | Server owns movement result, hits, damage, inventory, loot, zone, vehicles |
| Tick / snapshot rate | Server sim 30 Hz; snapshots to clients 15–20 Hz (adaptive) |
| Client prediction | Predict own movement and firing effects; reconcile on server correction |
| Interpolation | Remote entities rendered ~100 ms behind |
| Lag compensation | Server rewinds hitboxes up to ~200 ms for hitscan validation |
| Interest management | Send only entities within a relevance radius (e.g. 150 m, vehicles/aircraft farther) |
| Bandwidth | Target ≤ 30–40 KB/s per client average; quantize positions/rotations, delta-compress |
| Reliability | Unreliable channel for movement, reliable for events (kills, pickups, zone) |
| Reconnection | Match server holds a slot for 60 s; client resumes with session token |
| Disconnect handling | AFK/disconnected player becomes a bot only after the hold window, labeled |
| Anti-cheat basics | Server validates speed, fire rate, ammo, line of sight, and pickup range; rate-limits inputs; logs anomalies; obfuscated client build; report system; ban service |

Test conditions required: 50-player synthetic load test, plus packet loss 5–10%, latency 50–250 ms, jitter, and forced disconnects.

---

## 8. Map blueprint

**Size:** about 1.5 × 1.5 km for 50 players (tune after playtests). Terrain from Unity Terrain with heightmap; roads via spline tools; buildings modular.

| Zone (POI) | Type | Loot tier | Notes |
|---|---|---|---|
| Harbor Town | Residential/commercial | Medium | Dense, many interiors, river crossing |
| Old Factory | Industrial | High | Multi-floor, catwalks, tight fights |
| Fort Ridge | Military training area | Highest | Obstacle course, towers, bunker |
| Pine Village | Residential | Low–medium | Spread out, forest cover |
| Quarry | Industrial | Medium | Elevation, long sightlines for snipers |
| Bridge Junction | Roads/bridges | Low | Chokepoints, vehicle route |
| Wilderness | Forests, hills, river | Low | Rotation routes, ambush spots |

**Modular kit:** wall/floor/roof pieces, doors, windows, stairs, props on a 2 m grid so new POIs can be assembled quickly. Every building has at least 2 entrances and a defensible upper floor.

**Optimization:** terrain LOD, tile-based streaming, baked occlusion culling, LOD0–LOD3 on props, GPU instancing for foliage, blob or baked shadows on low tier, mixed lighting with baked global illumination.

---

## 9. Art, animation, and audio specification

**Characters:** 8–15k triangles, 1024² atlas textures, one shared humanoid rig, 4–6 outfit variants at launch. Your Player.blend is the starting point and needs a skeleton, weights, and animations.
**Animation set:** idle, walk, run, sprint, crouch (idle/walk), prone, jump, land, aim, fire (per weapon class), reload (per class), heal, throw, parachute, freefall, hit-react, death, revive.
**Weapons:** 2–5k triangles each, shared material atlas, attachment sockets.
**Audio:** footsteps per surface, gun fire and reload per weapon, distance filtering, hit markers, explosion, ambience, UI clicks, adaptive music (menu, drop, final circle). Use original or properly licensed assets; keep a license log.
**VFX (pooled):** muzzle flash, tracer, impact per surface, smoke, explosion, zone edge, supply-drop flare.

---

## 10. UI/UX map

`Boot → Login (guest / linked) → Main menu → Mode select → Party lobby → Matchmaking → Loading → Match → Results → Menu`

Screens: main menu, character selection, mode selection (training, vs bots, solo, duo, squad, custom, ranked later), friends and party, inventory and loadout, store (cosmetics), settings (graphics, audio, controls, sensitivity, gyro), HUD (health, armor, ammo, minimap, compass, zone timer, players left, kill feed, ping), results (placement, kills, damage, survival time), loading and connection status.

Rule: no button ships unless it works.

---

## 11. Performance budgets (targets)

| Metric | Low tier | Mid tier | High tier |
|---|---|---|---|
| Frame rate | 30 fps | 30–45 fps | 60 fps |
| Resolution scale | 0.6–0.75 | 0.85 | 1.0 |
| Draw calls | < 150 | < 250 | < 400 |
| Visible triangles | < 150k | < 250k | < 400k |
| Memory | < 1.2 GB | < 1.8 GB | < 2.5 GB |
| Shadows | Off / blob | Cascade 1 | Cascade 2 |
| Install size | < 150 MB base, extra content via downloadable bundles | | |

Techniques: object pooling, addressables with async loading, texture compression (ASTC), SRP batcher, GPU instancing, no per-frame allocations in gameplay code, thermal throttling detection with auto quality drop.

---

## 12. Backend services and data model

| Service | Responsibility |
|---|---|
| Auth | Guest sign-in; link Google/Apple later; session tokens |
| Profile | Name, level, stats, settings |
| Friends/Party | Friend list, invites, party leader, ready state |
| Matchmaker | Queue by mode/region/party size; later skill-based (MMR) |
| Fleet/Allocation | Reserve a match server and hand out address + join token |
| Results | Receive match report, update stats/XP/rank |
| Inventory/Store | Cosmetics ownership, purchases, entitlements |
| Moderation | Reports, bans, chat filtering |

Core tables: `players`, `sessions`, `parties`, `party_members`, `matches`, `match_players`, `inventory_items`, `cosmetic_defs`, `purchases`, `reports`, `bans`.

---

## 13. Security, privacy, compliance

- Server-side validation of everything that affects results; never trust client state.
- TLS for all service calls; signed join tokens for match servers; rate limiting.
- Privacy policy, data-deletion path, and age-appropriate design; check COPPA/GDPR and regional rules before public launch.
- Get the right content rating (IARC on Google Play) and disclose any purchases.
- Store credentials in a secrets manager, never in the client build.

---

## 14. Monetization (fair)

Cosmetic skins, emotes, parachutes, weapon skins, and a battle pass. No paid stats, no paid damage, no paid loot advantages. All prices and store rules need current verification with Google Play billing requirements before launch.

---

## 15. Development roadmap

| Phase | Goal | Main deliverables | Exit criteria |
|---|---|---|---|
| 0. Pre-production | Lock decisions | Unity project, netcode load-test of 50 bots, art style guide, asset license log | Netcode choice validated |
| 1. Vertical slice | One fun match offline | Player controller, 3 weapons, small map, loot, zone, bots | Complete match playable on a phone |
| 2. Battle royale core | Full loop | Large map, parachute, all weapons, armor, HUD, lobby, results, bot difficulty | Stable 30 fps on low tier |
| 3. Online alpha | Real multiplayer | Dedicated servers, matchmaking, parties, prediction, lag comp, validation | 50-player match across real devices, acceptable under packet loss |
| 4. Beta | Content and polish | Vehicles, interiors, more POIs, cosmetics, progression, audio pass, ranked prototype | Crash-free sessions above target, balance pass done |
| 5. Soft launch | Real-world test | Limited regions, analytics, live-ops tools, anti-cheat tuning | Retention and stability targets met |
| 6. Launch | Public release | Store listing, support, moderation, monitoring | Go/no-go checklist passed |

**Team (est. minimum for phases 1–4):** 1 producer, 2 gameplay programmers, 1–2 network/backend programmers, 1 tech artist, 2 3D artists, 1 animator, 1 level designer, 1 UI/UX designer, 1 audio designer, 1 QA. A smaller team is possible but the schedule scales accordingly.

---

## 16. Test plan

**Automated:** unit tests for damage, loot rolls, zone math; PlayMode tests for match flow; server simulation tests; CI build on every merge.
**Manual checklist:** controls on 3 phone sizes, all weapons, all pickups, parachute edge cases, zone damage, win/loss, reconnect, low-end thermal test, low battery, interruptions (calls/notifications), and one-handed/left-handed layouts.
**Network:** 50-player soak test, packet loss, latency spikes, disconnect storms, server crash recovery.
**Balance:** weapon time-to-kill table, loot heatmaps, match length distribution, drop-spot popularity, bot vs human win rates.

---

## 17. Key risks and mitigations

| Risk | Mitigation |
|---|---|
| 50-player netcode too heavy on mobile bandwidth | Early load test, interest management, adaptive snapshot rate |
| Low-end phones overheat or stutter | Strict budgets, quality auto-scaling, profile every sprint |
| Cheating | Server authority first, telemetry-based detection, fast ban pipeline |
| Scope creep | Ship phases in order; no feature starts until the previous exit criteria pass |
| Asset licensing problems | Original assets or documented licenses only; no copied proprietary content |
| Server cost | Region-limited launch, autoscaling, bots to fill low-population matches |

---

## 18. Immediate next actions

1. Decide engine and netcode, then run the 50-player load test.
2. Create the Unity project using the folder structure above and set up Git LFS and CI.
3. Rig and animate the Player.blend character (Unity humanoid rig).
4. Port the prototype's weapon and zone values into ScriptableObjects.
5. Build the phase-1 vertical slice on a real Android phone before adding online features.
