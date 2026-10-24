# Gti-Club-Link
Fork of mame 0.289 to play Gti club on LAN 
# GTI Club : link multijoueur dans MAME / GTI Club multiplayer link for MAME

[Français](#français) · [English](#english)

---

# Français

Prototype qui permet de jouer à deux à **GTI Club** (Konami, set MAME `gticlub`) entre deux instances de MAME, sur le même PC ou sur deux PC d'un même réseau. La puce réseau Konami 056230 (LANC), qui n'était qu'un « wrapper » dans MAME, échange maintenant les données de la borne par paquets UDP.

**Statut : prototype fonctionnel, pas un émulateur complet de la LANC.** Testé avec deux bornes (Network ID 1 et 2), sur un seul PC (deux fenêtres) et sur deux PC Windows. Le kit prêt à l'emploi (section 2) a été testé sur un seul PC ; son usage sur deux PC reprend la configuration déjà validée à la main.

## 1. Comment ça marche

### Constaté (code du driver MAME, désassemblage de la ROM, logs)
- Le driver `src/mame/konami/gticlub.cpp` utilise la puce `k056230` (`src/devices/machine/k056230.cpp`). Avant ce patch, ses registres ne faisaient rien et aucune trame n'était échangée (le set est marqué `MACHINE_NODEVICE_LAN`).
- Registres : base `0x7e008000` (octet 0 : statut en lecture, mode en écriture ; octet 1 : contrôle ; octet 2 : sub ID). RAM réseau en `0x7e00a000`. La puce lève l'IRQ2 du PowerPC.
- Le DIP « Network ID » (1 à 4) choisit la borne. Les écritures du registre de mode valent `0x08` / `0x28` pour l'ID 1 et `0x0c` / `0x2c` pour l'ID 3 : les bits bas valent `2 x (ID - 1)`.
- Avec le statut initial à `0x08`, le jeu démarre en solo. Avec `0x00`, il affiche NETWORK ERROR.

### Interprétation retenue (déduite du désassemblage, non confirmée par une documentation)
- La RAM réseau est découpée en 8 slots de 256 octets ; chaque borne en occupe deux consécutifs, le premier étant `2 x (ID - 1)`.
- Chaque trame utile fait 128 octets, encadrée par un marqueur de 16 bits en début (`+0x00`) et en fin (`+0x7e`) de slot, comparés à une table de la ROM.
- Le bit `0x08` du statut signifie « ligne libre ».
- Un cycle d'échange = écriture du mode `0x28 | slot`, puis du contrôle `0x37`.
- Les rôles maître/esclave (`NC_MAST`, `NC_SLAV`) sont gérés par le jeu lui-même.

### Ce que fait le patch
À chaque contrôle `0x37` (bit 5 du mode actif), avant la levée de l'IRQ :
1. la borne envoie ses deux slots (512 octets) à l'autre en UDP ;
2. elle recopie dans sa RAM réseau les slots reçus de l'autre borne ;
3. en mode synchronisé (activé par défaut), elle attend d'abord au plus `wait_ms` millisecondes (250 par défaut) un nouveau paquet du voisin : les deux machines avancent au rythme de la plus lente.

Format d'un paquet (524 octets) : `G`, `T`, `I`, numéro de slot de la borne, identifiant de session (4 octets), numéro de séquence (4 octets), puis 512 octets de données.

## 2. Utilisation rapide (kit prêt à l'emploi)

Le script `make_kit.sh` assemble `GTIClub.zip` à partir du `gticlub.exe` compilé (section 3). Contenu : l'exe, `gti_link.ini`, `joueur1.bat`, `joueur2.bat`, `test_meme_pc.bat`, `pare-feu.bat`, `ui.ini`, un dossier `roms\` vide et un `LISEZMOI.txt`. **Les ROMs ne sont pas fournies.**

Sur chaque PC :
1. Décompresser le kit et placer `gticlub.zip` (ROMs) dans `roms\`.
2. Éditer `gti_link.ini` : `peer=` suivi de l'adresse IP de l'**autre** PC et de `:5000` (exemple : `peer=192.168.1.20:5000`). L'IP s'obtient avec `ipconfig` (ligne « Adresse IPv4 »).
3. Clic droit sur `pare-feu.bat`, puis « Exécuter en tant qu'administrateur » (une fois par PC).
4. Lancer `joueur1.bat` sur un PC et `joueur2.bat` sur l'autre.
5. Mettre un crédit, choisir voiture et circuit : chaque PC détecte l'autre.

`test_meme_pc.bat` ouvre les deux joueurs dans deux fenêtres sur un seul PC.

Chaque lanceur utilise son propre dossier de configuration (`cfg1` / `cfg2`), donc le joueur 1 est toujours Network ID 1 et le joueur 2 toujours ID 2. Le fichier `cfg2\gticlub.cfg` du kit force l'ID 2 ; il a été écrit d'après le format des fichiers MAME. Si « Network ID » n'affiche pas 2 (`Tab`, Dip Switches), le régler une fois à la main.

### Configuration (`gti_link.ini`)
Les variables d'environnement ont priorité sur le fichier.

| Clé (ini) | Variable | Rôle |
|---|---|---|
| `peer` | `GTI_PEER` | `ip:port` de l'autre PC (obligatoire : sans lui, la liaison est désactivée). |
| `port` | `GTI_PORT` | Port UDP d'écoute local (par défaut, celui de `peer`). |
| `sync` | `GTI_SYNC` | `1` : verrouillage des deux machines cycle par cycle (défaut) ; `0` : désactivé. |
| `wait_ms` | `GTI_WAIT_MS` | Attente maximale d'un paquet, en ms (défaut 250, maximum 900). |

- Le fichier `gti_link.log` (créé dans le dossier courant) donne, chaque seconde : `cycles`, `sent`, `recv`, `stale`, `timeouts`. Si `recv` reste à 0 : vérifier l'IP, le pare-feu et le réseau.
- Les deux PC doivent utiliser **la même version** de `gticlub.exe`.
- Pour masquer l'écran d'avertissement MAME : `ui.ini` contenant `skip_warnings 1` (fourni dans le kit) et l'option `-skip_gameinfo` (déjà dans les lanceurs).

## 3. Compilation

Ordre : partir d'un `k056230.cpp` d'origine (`git checkout src/devices/machine/k056230.cpp`), puis :
```
python3 patch_gti.py
python3 patch_sync.py
python3 patch_defaults.py
python3 patch_ini.py
```
Chaque script s'arrête avec « ancre introuvable » si le fichier ne correspond pas à la version de MAME prévue. Version de MAME utilisée : `git rev-parse HEAD` = `ec20df4a93bef43a92d5974a3299e91f718b7451`.

### Linux (testé sous Ubuntu dans WSL)
GCC 13 a produit des erreurs internes sur ce code ; clang a fonctionné :
```
make SUBTARGET=gticlub SOURCES=src/mame/konami/gticlub.cpp REGENIE=1 USE_QTDEBUG=0 PRECOMPILE=0 OPTIMIZE=1 SYMBOLS=0 OVERRIDE_CC=clang OVERRIDE_CXX=clang++ -j2
```

### Windows, depuis Ubuntu (cross-compilation)
```
sudo apt install mingw-w64
sudo ln -sf /usr/bin/x86_64-w64-mingw32-g++-posix /usr/x86_64-w64-mingw32/bin/x86_64-w64-mingw32-g++
sudo ln -sf /usr/bin/x86_64-w64-mingw32-gcc-posix /usr/x86_64-w64-mingw32/bin/x86_64-w64-mingw32-gcc
sudo ln -sf /usr/bin/x86_64-w64-mingw32-gcc-ar-posix /usr/x86_64-w64-mingw32/bin/x86_64-w64-mingw32-gcc-ar
export MINGW64=/usr/x86_64-w64-mingw32
make SUBTARGET=gticlub SOURCES=src/mame/konami/gticlub.cpp REGENIE=1 TARGETOS=windows CROSS_BUILD=1 PTR64=1 PRECOMPILE=0 OPTIMIZE=3 SYMBOLS=0 -j2
bash make_kit.sh
```
Pour changer le niveau d'optimisation, supprimer d'abord le dossier objets du build Windows dans `build/` : sinon `make` garde les anciens fichiers. Ne pas lancer `make clean` si l'on veut conserver aussi le build Linux.

Facultatif : retirer le drapeau `MACHINE_NODEVICE_LAN` des lignes `GAME(...)` de `gticlub.cpp` pour supprimer l'avertissement « réseau non émulé ». Laisser les drapeaux `MACHINE_IMPERFECT_*`.

## 4. Fichiers du projet

| Fichier | Rôle |
|---|---|
| `patch_gti.py` | Transport UDP (Linux et Windows) dans un `k056230.cpp` d'origine. |
| `patch_sync.py` | Session, séquence, verrouillage, statistiques. |
| `patch_defaults.py` | Verrouillage et attente de 250 ms par défaut. |
| `patch_ini.py` | Lecture de `gti_link.ini`. |
| `make_kit.sh` | Assemble `GTIClub.zip`. |
| `deint.py` | Reconstitue `prg.bin` (ROM programme PowerPC) à partir des 4 fichiers entrelacés. |
| `funcs.py` | Retrouve les noms de fonctions laissés dans la ROM et désassemble une fonction par son nom. |

## 5. Résultats de test

| Essai | Résultat |
|---|---|
| Deux instances sur le même PC | Fonctionne, sans saccades. |
| Deux PC, sans synchronisation | Détection et choix en multi OK ; NETWORK ERROR quelques secondes après le départ. |
| Deux PC, `sync=1`, `wait_ms=30` | Réseau sain (environ 30 cycles/s, aucun paquet perdu) mais le PC le plus lent a des à-coups supérieurs à 30 ms : désynchronisation, erreur. |
| Deux PC, `sync=1`, `wait_ms=250` | Fonctionne ; le PC le plus lent rame un peu mais la partie tient. |
| Kit `GTIClub.zip`, un seul PC (`test_meme_pc.bat`) | Fonctionne. |

Cause probable de l'erreur sans verrouillage : la vitesse d'émulation du PC le plus lent passe sous 100 % pendant la course. C'est une conclusion tirée des logs, pas une mesure directe.

## 6. Limites

- La LANC n'est pas émulée (pas de CRC, pas de timing de bus) : le jeu reçoit simplement des trames dans les slots de son voisin.
- Les écrans NETWORKING CONFIGURATION (« Data receiving failure ») et NETWORK CHECK du menu de service ne se terminent pas avec succès ; ils ne sont pas nécessaires pour jouer.
- Seuls les Network ID 1 et 2 ont été testés. Liaison point à point uniquement (un seul voisin).
- Aucune authentification ni chiffrement, et les paquets ne sont pas filtrés par adresse source : à n'utiliser que sur un réseau de confiance ou via un VPN.
- Si le PC voisin disparaît en pleine partie, le jeu peut se figer environ trois quarts de seconde (trois attentes de 250 ms) avant de reprendre.
- Un PC lent ralentit les deux joueurs en mode synchronisé. Pistes : build `OPTIMIZE=3`, options MAME `-autoframeskip` ou `-speed 0.7`.
- Non testé : Internet (latence), plus de deux bornes, macOS, Wi-Fi.
- Environ 50 secondes de démarrage (réglage du volant) : non traité.

## 7. Notes de rétro-ingénierie

Adresses du code PowerPC de la ROM `gticlub` (adresse dans `prg.bin` = adresse - `0xFF800000`). Les adresses de PC des logs MAME sont approximatives à cause du recompilateur dynamique.

| Adresse | Nom | Rôle supposé |
|---|---|---|
| `FF84413C` | `lanc_sync_frame` | Attend l'activité puis une fenêtre de silence (statut bit `0x08`). |
| `FF844224` | `comm_line_free` | Mesure l'état de la ligne sur 384 ticks, IRQ masquées. |
| `FF8442B4` | `lanc_init_trigger` | Lance le premier cycle (contrôle `0x10` puis `0x36`). |
| `FF84434C` | `lanc_init` | Initialise les tampons et la LANC. |
| `FF844644` | `lanc_interrupt` | Gestionnaire d'IRQ : vérifie les marqueurs, recopie les 8 slots, compte les erreurs par nœud. |
| `FF844B14` | `lanc_interval` | Déclenche un cycle (mode `0x28 | slot`, contrôle `0x37`). |
| `FF844C14` | `lanc_main` | Initialise les marqueurs des slots. |
| `FF844E24` / `FF844FFC` | `lanc_read` / `lanc_write` | Lecture et écriture des trames côté jeu. |
| `FF845A88` | `dev_check_lanc` | Test de la RAM de la LANC (motif modulo 255). |
| `FF87927C`, `FF879180`, `FF8794C4`, `FF8797A4` | `self_networking_configuration`, `net_write`, `net_master`, `net_slave` | Configuration réseau. |

Textes de la ROM utiles : `NETWORK ERROR`, `CHECK NETWORK CABLES`, `LANC STATUS ERROR`, `CARS LINKED PLAY IS AVAILABLE.`, et un écran de diagnostic (`LANC ID`, `lanc interrupted`, `maybe mid error`, `slave/master`, `nr_node`).

## 8. Licence

`k056230.cpp` est sous licence BSD-3-Clause (voir son en-tête) : conserver les en-têtes en modifiant le fichier. Les ROMs ne sont pas fournies et ne doivent pas être distribuées. Pour diffuser un exécutable compilé, mettre aussi les sources modifiées à disposition. (Information générale, pas un avis juridique.) Ce projet n'est affilié ni à Konami ni à l'équipe MAME.

---

# English

A prototype that lets two players race together in **GTI Club** (Konami, MAME set `gticlub`) between two MAME instances, on the same PC or on two PCs on the same network. The Konami 056230 network chip (LANC), which was only a stub in MAME, now exchanges the cabinet's link data over UDP packets.

**Status: working prototype, not a full LANC emulation.** Tested with two cabinets (Network ID 1 and 2), on a single PC (two windows) and on two Windows PCs. The ready-to-use kit (section 2) was tested on a single PC; using it on two PCs reuses the configuration already validated by hand.

## 1. How it works

### Observed (MAME driver code, ROM disassembly, logs)
- The driver `src/mame/konami/gticlub.cpp` uses the `k056230` chip (`src/devices/machine/k056230.cpp`). Before this patch its registers did nothing and no frame was exchanged (the set is flagged `MACHINE_NODEVICE_LAN`).
- Registers: base `0x7e008000` (byte 0: status on read, mode on write; byte 1: control; byte 2: sub ID). Network RAM at `0x7e00a000`. The chip raises the PowerPC's IRQ2.
- The "Network ID" DIP switch (1 to 4) selects the cabinet. Writes to the mode register were `0x08` / `0x28` for ID 1 and `0x0c` / `0x2c` for ID 3: the low bits equal `2 x (ID - 1)`.
- With the initial status at `0x08`, the game starts in single-player. With `0x00`, it shows NETWORK ERROR.

### Working interpretation (inferred from the disassembly, not confirmed by documentation)
- Network RAM is split into 8 slots of 256 bytes; each cabinet owns two consecutive slots, the first being `2 x (ID - 1)`.
- Each useful frame is 128 bytes, framed by a 16-bit marker at the start (`+0x00`) and the end (`+0x7e`) of the slot, compared against a table in the ROM.
- Status bit `0x08` means "line free".
- One exchange cycle = a write of mode `0x28 | slot`, then control `0x37`.
- Master/slave roles (`NC_MAST`, `NC_SLAV`) are handled by the game itself.

### What the patch does
On each control write `0x37` (mode bit 5 set), before the IRQ is raised:
1. the cabinet sends its two slots (512 bytes) to the other one over UDP;
2. it copies the slots received from the other cabinet into its network RAM;
3. in synchronized mode (on by default), it first waits up to `wait_ms` milliseconds (250 by default) for a fresh packet from the peer, so both machines advance at the pace of the slower one.

Packet format (524 bytes): `G`, `T`, `I`, the cabinet's slot number, a session ID (4 bytes), a sequence number (4 bytes), then 512 bytes of data.

## 2. Quick start (ready-to-use kit)

`make_kit.sh` builds `GTIClub.zip` from the compiled `gticlub.exe` (section 3). Contents: the exe, `gti_link.ini`, `joueur1.bat` (player 1), `joueur2.bat` (player 2), `test_meme_pc.bat` (single-PC test), `pare-feu.bat` (firewall), `ui.ini`, an empty `roms\` folder and a `LISEZMOI.txt` (French). **ROMs are not included.**

On each PC:
1. Unzip the kit and put `gticlub.zip` (the ROMs) in `roms\`.
2. Edit `gti_link.ini`: `peer=` followed by the IP address of the **other** PC and `:5000` (example: `peer=192.168.1.20:5000`). Get the IP with `ipconfig` ("IPv4 Address" line).
3. Right-click `pare-feu.bat`, then "Run as administrator" (once per PC).
4. Run `joueur1.bat` on one PC and `joueur2.bat` on the other.
5. Insert a credit, pick a car and a track: each PC detects the other.

`test_meme_pc.bat` opens both players in two windows on a single PC.

Each launcher uses its own configuration folder (`cfg1` / `cfg2`), so player 1 is always Network ID 1 and player 2 always ID 2. The kit's `cfg2\gticlub.cfg` forces ID 2; it was written from the format of MAME config files. If "Network ID" does not show 2 (`Tab`, Dip Switches), set it by hand once.

### Configuration (`gti_link.ini`)
Environment variables take priority over the file.

| ini key | Variable | Purpose |
|---|---|---|
| `peer` | `GTI_PEER` | `ip:port` of the other PC (required: without it the link is disabled). |
| `port` | `GTI_PORT` | Local UDP listening port (defaults to the one in `peer`). |
| `sync` | `GTI_SYNC` | `1`: lock both machines cycle by cycle (default); `0`: disabled. |
| `wait_ms` | `GTI_WAIT_MS` | Maximum wait for a packet, in ms (default 250, maximum 900). |

- The `gti_link.log` file (created in the current folder) records, every second: `cycles`, `sent`, `recv`, `stale`, `timeouts`. If `recv` stays at 0, check the IP, the firewall and the network.
- Both PCs must run **the same version** of `gticlub.exe`.
- To hide MAME's warning screen: `ui.ini` containing `skip_warnings 1` (included in the kit) and the `-skip_gameinfo` option (already in the launchers).

## 3. Building

Order: start from an unmodified `k056230.cpp` (`git checkout src/devices/machine/k056230.cpp`), then:
```
python3 patch_gti.py
python3 patch_sync.py
python3 patch_defaults.py
python3 patch_ini.py
```
Each script stops with "ancre introuvable" (anchor not found) if the file does not match the expected MAME version. MAME version used: `git rev-parse HEAD` = `ec20df4a93bef43a92d5974a3299e91f718b7451`.

### Linux (tested on Ubuntu under WSL)
GCC 13 hit internal compiler errors on this code; clang worked:
```
make SUBTARGET=gticlub SOURCES=src/mame/konami/gticlub.cpp REGENIE=1 USE_QTDEBUG=0 PRECOMPILE=0 OPTIMIZE=1 SYMBOLS=0 OVERRIDE_CC=clang OVERRIDE_CXX=clang++ -j2
```

### Windows, from Ubuntu (cross-compilation)
```
sudo apt install mingw-w64
sudo ln -sf /usr/bin/x86_64-w64-mingw32-g++-posix /usr/x86_64-w64-mingw32/bin/x86_64-w64-mingw32-g++
sudo ln -sf /usr/bin/x86_64-w64-mingw32-gcc-posix /usr/x86_64-w64-mingw32/bin/x86_64-w64-mingw32-gcc
sudo ln -sf /usr/bin/x86_64-w64-mingw32-gcc-ar-posix /usr/x86_64-w64-mingw32/bin/x86_64-w64-mingw32-gcc-ar
export MINGW64=/usr/x86_64-w64-mingw32
make SUBTARGET=gticlub SOURCES=src/mame/konami/gticlub.cpp REGENIE=1 TARGETOS=windows CROSS_BUILD=1 PTR64=1 PRECOMPILE=0 OPTIMIZE=3 SYMBOLS=0 -j2
bash make_kit.sh
```
To change the optimization level, first delete the Windows build's object folder under `build/`, otherwise `make` keeps the old objects. Do not run `make clean` if you also want to keep the Linux build.

Optional: remove the `MACHINE_NODEVICE_LAN` flag from the `GAME(...)` lines in `gticlub.cpp` to drop the "network not emulated" warning. Leave the `MACHINE_IMPERFECT_*` flags alone.

## 4. Project files

| File | Purpose |
|---|---|
| `patch_gti.py` | UDP transport (Linux and Windows) added to an unmodified `k056230.cpp`. |
| `patch_sync.py` | Session, sequence numbers, lock-step synchronization, statistics. |
| `patch_defaults.py` | Synchronization and the 250 ms wait on by default. |
| `patch_ini.py` | Reads `gti_link.ini`. |
| `make_kit.sh` | Builds `GTIClub.zip`. |
| `deint.py` | Rebuilds `prg.bin` (PowerPC program ROM) from the 4 interleaved files. |
| `funcs.py` | Finds the function names Konami left in the ROM and disassembles a function by name. |

## 5. Test results

| Test | Result |
|---|---|
| Two instances on the same PC | Works, no stutter. |
| Two PCs, no synchronization | Detection and multiplayer selection OK; NETWORK ERROR a few seconds after the start. |
| Two PCs, `sync=1`, `wait_ms=30` | Healthy network (about 30 cycles/s, no lost packets) but the slower PC stutters by more than 30 ms: the machines drift apart, error. |
| Two PCs, `sync=1`, `wait_ms=250` | Works; the slower PC lags a little but the race holds. |
| `GTIClub.zip` kit, single PC (`test_meme_pc.bat`) | Works. |

Probable cause of the error without synchronization: the slower PC's emulation speed drops below 100% during the race. This is a conclusion drawn from the logs, not a direct measurement.

## 6. Limitations

- The LANC is not emulated (no CRC, no bus timing): the game simply receives frames in its neighbour's slots.
- The NETWORKING CONFIGURATION ("Data receiving failure") and NETWORK CHECK screens of the service menu do not complete successfully; they are not needed to play.
- Only Network IDs 1 and 2 were tested. Point-to-point only (a single peer).
- No authentication or encryption, and packets are not filtered by source address: use only on a trusted network or through a VPN.
- If the peer PC disappears mid-game, the game may freeze for about three quarters of a second (three 250 ms waits) before resuming.
- A slow PC slows both players down in synchronized mode. Ideas: an `OPTIMIZE=3` build, MAME options `-autoframeskip` or `-speed 0.7`.
- Not tested: the Internet (latency), more than two cabinets, macOS, Wi-Fi.
- About 50 seconds of start-up time (steering wheel setup): not addressed.

## 7. Reverse-engineering notes

Addresses in the PowerPC code of the `gticlub` ROM (address in `prg.bin` = address - `0xFF800000`). PC addresses in MAME logs are approximate because of the dynamic recompiler.

| Address | Name | Presumed role |
|---|---|---|
| `FF84413C` | `lanc_sync_frame` | Waits for activity, then for a quiet window (status bit `0x08`). |
| `FF844224` | `comm_line_free` | Measures the line state over 384 ticks with IRQs masked. |
| `FF8442B4` | `lanc_init_trigger` | Starts the first cycle (control `0x10` then `0x36`). |
| `FF84434C` | `lanc_init` | Initializes the buffers and the LANC. |
| `FF844644` | `lanc_interrupt` | IRQ handler: checks markers, copies the 8 slots, counts per-node errors. |
| `FF844B14` | `lanc_interval` | Triggers a cycle (mode `0x28 | slot`, control `0x37`). |
| `FF844C14` | `lanc_main` | Initializes the slot markers. |
| `FF844E24` / `FF844FFC` | `lanc_read` / `lanc_write` | Game-side frame read and write. |
| `FF845A88` | `dev_check_lanc` | LANC RAM test (modulo-255 pattern). |
| `FF87927C`, `FF879180`, `FF8794C4`, `FF8797A4` | `self_networking_configuration`, `net_write`, `net_master`, `net_slave` | Network configuration. |

Useful ROM strings: `NETWORK ERROR`, `CHECK NETWORK CABLES`, `LANC STATUS ERROR`, `CARS LINKED PLAY IS AVAILABLE.`, and a diagnostic screen (`LANC ID`, `lanc interrupted`, `maybe mid error`, `slave/master`, `nr_node`).

## 8. License

`k056230.cpp` is under the BSD-3-Clause license (see its header): keep the headers when modifying the file. ROMs are not provided and must not be distributed. To distribute a compiled executable, also make the modified sources available. (General information, not legal advice.) This project is not affiliated with Konami or the MAME team.
