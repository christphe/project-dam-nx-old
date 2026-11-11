# GoldenEye NX : plan de portage Switch

Base : `upstream/release/v0.6.0` (jkdansereau/goldeneye-pc-port, `5b86a9bb`), qui contient
déjà le port macOS arm64 (PR #95, fusionnée en `903095c9`, unifiée en D607).
Objectif : un `.nro` homebrew qui lance le jeu depuis la ROM de l'utilisateur, avec
contrôles modernes (dual-stick, gyro) et les QoL du port PC, sur le modèle de
2ship2harkinian / dusklightdusk / windwaker. Mode **title takeover uniquement**.

## 1. État des lieux

| Domaine | Port PC (v0.6.0) | Switch |
|---|---|---|
| Code du jeu | C décompilé, natif, 64 bits, déjà validé sur arm64 (macOS) | OK (aarch64) |
| Rendu | fast3d → OpenGL (`port/fast3d/gfx_opengl.cpp`), fenêtre SDL | switch-mesa + switch-sdl2 (même voie que 2ship) |
| Audio, threads, fichiers | SDL, pthreads, stdio | OK via libnx/newlib ; chemins `sdmc:/` à adapter |
| Entrées | `SDL_GameController`, presets, remapping | OK via SDL ; gyro via libnx (`hidGetSixAxisSensorStates`) |
| **Mémoire N64** | fenêtre de 4 Gio à `PORT_ADDR_BASE` (0 sur x86-64, 16 Tio fixe sur macOS) | **bloquant** : base fixe impossible, NRO en PIE à adresse aléatoire |

### Le modèle d'adresses d'upstream ([port/include/portaddr.h](../../port/include/portaddr.h))

- L'espace 32 bits de la N64 est une fenêtre d'adresses hôte `[PORT_ADDR_BASE, +4 Gio)` :
  cartouche `0x10…`, encodage relatif à l'image `0x40…–0x6…`, DRAM V1 `0x70…`,
  miroir KSEG0 `0x80…` (mêmes pages que V1), piles des threads `0xa…`.
- Point de conversion unique : `portN64ToHost()` / `portHostToN64()` / `PORT_N64PTR()`,
  et `PORT_ADDR_STRICT` pour attraper les conversions invalides.
- **Invariant : `(u32)ptr` doit faire l'aller-retour.** Le jeu tronque des pointeurs hôtes
  en `u32`/`s32` puis les reconvertit (`PORT_N64PTR(T, (s32)ptr + off)`). Ça ne marche que si
  les 32 bits bas d'une adresse hôte sont son adresse N64, donc si la base est alignée sur 4 Gio.
- Symboles absolus : `dram_syms.S` (`cfb_16`, `_bssSegmentEnd`, `animations_frame_buffer` =
  `PORT_DRAM_V1_BASE + off`) et `romassets_*.s` (labels de la ROM en `0x10000000 + offset`).
- `PORT_ADDR_BASE` est une constante de compilation.

### Contraintes Horizon
- Espace d'adressage 39 bits, régions (heap, alias, stack, code) placées aléatoirement par processus :
  pas de base fixe connue à la compilation.
- Pas de `mmap MAP_FIXED` ; `svcMapMemory` ne mappe que dans la région « stack » et rend la source
  inaccessible : pas de double vue RW simple.

## 2. Modèle mémoire proposé pour la Switch

### Résultat du test console (2026-11-11, `tools/switch/memprobe`, title takeover, 3 lancements)

Régions du processus hbloader : heap `0x1a1600000` (8 Gio), alias `0xbd8800000` (6 Gio),
**stack `0x08000000–0x80000000`** (1920 Mio), ASLR `0x08000000–0x1000000000` (64 Gio).

| Méthode | Résultat |
|---|---|
| A. `svcMapMemory` heap → région stack | OK pour cart `0x10000000` et dram `0x70000000` ; les piles (`0xA0000000`) sont hors région |
| B. `svcMapPhysicalMemory` → alias | refusé, `0xfa01` (InvalidState) |
| C. code memory (`svcMapProcessCodeMemory` + RW) → ASLR | OK pour les 3 blocs avec base unique `H = 0x100000000` (11 bases possibles) |
| **D. disposition PC, H = 0** (A pour cart+dram, C pour les piles) | **OK, 3/3** ; `(u32)ptr` = adresse N64 partout |

**Décision : `PORT_ADDR_BASE = 0` sur Switch, disposition identique au PC.**
- Placement au tout début de `main` (avant tout thread : libnx place les piles de threads au hasard
  dans la région stack).
- Risque : la pile du thread principal est déjà dans la région stack, placée au hasard à la création
  du processus (`0x7596xxxx` lors du test). Collision avec cart/dram ≈ 2 % par lancement du jeu hôte.
  Détectée au démarrage → message « relancer » au début, puis repli sur la méthode C (base variable)
  si besoin.
- À vérifier : lecture de fichier (IPC fs) directement dans ces blocs ; sinon buffer intermédiaire.

### Repli : base variable

**Base variable, alignée sur 4 Gio, miroir V2 replié en logiciel.**

1. `PORT_ADDR_BASE` devient une variable (`g_portAddrBase`) sur Switch, fixée au démarrage ;
   les autres plateformes gardent la constante.
2. Au démarrage, on choisit une base `H` alignée sur 4 Gio et on y place la mémoire dont les
   32 bits bas doivent valoir l'adresse N64 : cartouche à `H+0x10000000`, DRAM à `H+0x70000000`,
   piles à `H+0xa0000000`. `(u32)ptr` refait alors l'aller-retour, comme sur PC et macOS.
3. **Pas de vue V2** : `portN64ToHost()` replie `0x80xxxxxx` sur `0x70xxxxxx` au lieu de mapper
   les mêmes pages deux fois.
4. Repli si une seule base `H` ne tient dans aucune région : une base par zone
   (`g_base[a >> 28] + a`, chaque zone placée à une adresse dont les 32 bits bas valent son
   adresse N64). Même invariant, contrainte de placement plus souple.
5. Symboles absolus : les labels ROM (`romassets_*.s`) sont des adresses N64 consommées comme des
   `u32` et restent absolus ; les 3 symboles DRAM deviennent des macros sur `portN64ToHost()`
   dans le build Switch.

**Test sur console avant tout (homebrew jetable, title takeover) :**
- afficher les régions du processus (`svcGetInfo` : heap, alias, stack, ASLR, adresse et taille) ;
- trouver une adresse `H` alignée sur 4 Gio telle que `H+0x10000000…H+0xb0000000` tombe dans une
  région où l'on peut mapper (`svcMapMemory` → région stack ; à défaut, la variante 4) ;
- mapper les trois zones, écrire et relire.

Mode de mise au point sur PC : un build Linux avec `PORT_ADDR_BASE` variable et non nulle,
`PORT_ADDR_STRICT=1` et le repli V2→V1, pour déboguer tout ce qui ne dépend pas de Horizon.

## 3. Étapes

### Étape 0 : environnement (≈ ½ journée)
- `Dockerfile.build` basé sur `local/switchwakerhd-build:latest` (devkitA64, libnx,
  switch-sdl2, switch-mesa, switch-glad), comme 2ship et dusklight, plus les outils hôtes
  Linux pour compiler la version Linux de référence.
- `compose.yaml` avec la ROM montée **en lecture seule**, jamais copiée dans l'image.
- Vérifier que la version Linux compile et lance le menu dans le conteneur.

### Étape 1 : intégrer #95 ✅ (fait par upstream dans v0.6.0)

### Étape 2 : test de placement mémoire sur console (≈ ½ journée)
Le homebrew décrit en §2. Décide entre la base unique et la base par zone.

### Étape 3 : base variable + repli V2 (1–2 semaines)
- `portaddr.h` : `PORT_ADDR_BASE` en variable sous `PLATFORM_SWITCH` (et sous un flag de test Linux) ;
  repli V2→V1 dans `portN64ToHost()`.
- `dram.c` : placement Switch (une seule vue) ; macros pour les symboles DRAM.
- Build Linux de test (base variable non nulle, `PORT_ADDR_STRICT=1`) puis balayage de tous les
  niveaux (`tools_pc/level_sweep*.sh`) et du multi 4 joueurs.
- Critère : les 21 niveaux et le multi jouables, sans erreur STRICT.

### Étape 4 : cible Switch (≈ 1 semaine)
- `PLATFORM_SWITCH` dans `port/include/platform.h` ; branche `NX_ROOT` dans CMake
  (sur le modèle de `../windwaker/cmake/SwitchPort.cmake`).
- Contexte GL : SDL2 + mesa ; vérifier que les shaders générés par `gfx_cc.cpp` passent.
- Chemins : config, sauvegardes et ROM dans `sdmc:/switch/goldeneye-nx/` ; romfs pour l'icône.
- `main.c` : `appletSetFocusHandlingMode`, sortie propre par le menu HOME, gestionnaire de crash
  libnx à la place de `crash.c`.
- Piles des threads réduites (8 Mo chacune sur PC ; Horizon ne fait pas d'overcommit).
- Empaquetage `.nro` (nacp, icône) ; à terme un forwarder.
- Critère : menu titre puis Dam jouable sur console, en mode docké et portable.

### Étape 5 : contrôles et QoL Switch
- Presets Joy-Con / Pro Controller (`docs/dev/QOL-INVENTORY.md`, `CONTROLLER-INPUT-PLAN.md`) ;
  dual-stick par défaut.
- **Visée gyro** (sensibilité, activation en visée seulement, recentrage).
- Joy-Con seul par joueur pour le split-screen 4 joueurs.
- Vibration HD (`RUMBLE-PLAN.md`).
- Résolution selon le mode (720p portable / 1080p docké).

### Étape 6 : performances
- Cible : 60 fps stables sur l'A57 à 1 GHz. Risques : fast3d (interprétation des display lists,
  création de shaders à la volée) et le driver GL.
- Leviers : `-O2`/LTO, cache de shaders sur disque, tri des appels GL, overclock optionnel.
  Un renderer deko3d (comme windwaker) seulement si GL ne suffit pas.

## 4. Risques

| Risque | Mitigation |
|---|---|
| Aucune région Horizon ne permet de placer les zones à la bonne adresse modulo 4 Gio | test de l'étape 2 avant tout code ; variante base par zone ; en dernier recours, auditer les troncatures (`portHostToN64` partout) |
| Code qui compare des adresses V1 et K0 | repli V2→V1 ; à surveiller avec `PORT_ADDR_STRICT` |
| Performances GL / mesa | mesurer tôt (étape 4) avant d'investir dans les QoL |
| Mémoire | title takeover uniquement ; piles de threads réduites |

## 5. Hors périmètre pour l'instant
- ROM PAL/JP.
- Mode en ligne.
- Packs de textures HD.
