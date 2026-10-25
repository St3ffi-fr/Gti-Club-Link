# GTI Club : link multijoueur dans MAME / GTI Club multiplayer link for MAME

[Français](#français) · [English](#english)

---

# Français

Prototype qui permet de jouer à **GTI Club** (Konami, set MAME `gticlub`) à **2, 3 ou 4 joueurs**, chaque joueur sur sa propre instance de MAME, sur un même PC ou sur plusieurs PC d'un réseau local câblé. La puce réseau Konami 056230 (LANC), qui n'était qu'un « wrapper » dans MAME, échange maintenant les données de la borne par paquets UDP.

**Statut : prototype fonctionnel, pas un émulateur complet de la LANC.** Testé avec 2 à 4 instances sur un même PC, et sur plusieurs PC reliés en réseau local câblé. Le Wi-Fi n'est pas pris en charge (voir section 6).

## 1. Comment ça marche

### Constaté (code du driver MAME, désassemblage de la ROM, logs, manuel)
- Le driver `src/mame/konami/gticlub.cpp` utilise la puce `k056230` (`src/devices/machine/k056230.cpp`). Avant ce patch, ses registres ne faisaient rien et aucune trame n'était échangée (le set est marqué `MACHINE_NODEVICE_LAN`).
- Registres : base `0x7e008000` (octet 0 : statut en lecture, mode en écriture ; octet 1 : contrôle ; octet 2 : sub ID). RAM réseau en `0x7e00a000`. La puce lève l'IRQ2 du PowerPC.
- Le DIP « Network ID » (1 à 4) choisit la borne. Les écritures du registre de mode valent `0x08` / `0x28` pour l'ID 1 et `0x0c` / `0x2c` pour l'ID 3 : les bits bas valent `2 x (ID - 1)`.
- Avec le statut initial à `0x08`, le jeu démarre en solo. Avec `0x00`, il affiche NETWORK ERROR.
- Le [manuel opérateur de la version Deluxe](https://www.arcade-museum.com/manuals-videogames/G/GTI-Club.pdf) indique que la borne se relie à **trois autres machines au plus** (4 en tout), chaque machine ayant son Network ID, et que la course en réseau démarre automatiquement quand une autre machine prend un crédit pendant l'écran d'entrée réseau. Ce comportement est géré par le jeu, pas par ce lien.

### Interprétation retenue (déduite du désassemblage, non confirmée par une documentation)
- La RAM réseau est découpée en 8 slots de 256 octets ; chaque borne en occupe deux consécutifs, le premier étant `2 x (ID - 1)`.
- Chaque trame utile fait 128 octets, encadrée par un marqueur de 16 bits en début (`+0x00`) et en fin (`+0x7e`) de slot, comparés à une table de la ROM.
- Le bit `0x08` du statut signifie « ligne libre ».
- Un cycle d'échange = écriture du mode `0x28 | slot`, puis du contrôle `0x37`.
- Les rôles maître/esclave (`NC_MAST`, `NC_SLAV`) sont gérés par le jeu lui-même.

### Ce que fait le patch
À chaque contrôle `0x37` (bit 5 du mode actif), avant la levée de l'IRQ :
1. la borne envoie ses deux slots (512 octets) aux autres, en UDP : soit en **diffusion sur le réseau local** (`peer=auto`, sans adresse IP), soit à une **liste d'adresses** ;
2. elle recopie dans sa RAM réseau les slots reçus de chaque autre borne ;
3. en mode synchronisé (activé par défaut), elle attend d'abord, au plus `wait_ms` millisecondes (250 par défaut), un nouveau paquet de **chaque** borne vue depuis moins de 2 secondes : toutes les machines avancent au rythme de la plus lente. Une borne qui fait trois attentes dépassées de suite n'est plus attendue jusqu'à ce qu'elle réapparaisse.

Format d'un paquet (524 octets) : `G`, `T`, `I`, numéro de slot de la borne, identifiant de session (4 octets), numéro de séquence (4 octets), puis 512 octets de données.

## 2. Utilisation rapide (kit prêt à l'emploi)

Le script `make_kit.sh` assemble `GTIClub.zip` à partir du `gticlub.exe` compilé (section 3). Contenu : l'exe, `gti_link.ini`, `joueur1.bat` à `joueur4.bat`, `test_meme_pc.bat`, `pare-feu.bat`, `ui.ini`, un dossier de configuration par numéro de joueur (`cfg2` à `cfg4`), un dossier `roms\` vide et un `LISEZMOI.txt`. **Les ROMs ne sont pas fournies.**

Sur chaque PC (réseau local câblé, de préférence sur un switch) :
1. Décompresser le kit et placer `gticlub.zip` (ROMs) dans `roms\`.
2. Clic droit sur `pare-feu.bat`, puis « Exécuter en tant qu'administrateur » (une fois par PC).
3. Lancer `joueur1.bat` sur le PC 1, `joueur2.bat` sur le PC 2, et ainsi de suite jusqu'à `joueur4.bat` : **chaque PC doit avoir un numéro différent**.
4. Mettre un crédit, choisir voiture et circuit : les PC se détectent seuls.

Si les PC ne se trouvent pas (plusieurs cartes réseau, VPN, réseau cloisonné), éditer `gti_link.ini` et remplacer `peer=auto` par la liste des adresses IP des **autres** PC (par exemple `peer=192.168.1.20,192.168.1.30`). L'IP s'obtient avec `ipconfig` (ligne « Adresse IPv4 »).

`test_meme_pc.bat` ouvre deux joueurs dans deux fenêtres sur un seul PC (adresses locales explicites).

Chaque lanceur utilise son propre dossier de configuration (`cfg1` à `cfg4`). Les fichiers `cfg2` à `cfg4` du kit forcent le Network ID 2, 3 ou 4 ; ils ont été écrits d'après le driver MAME (valeurs du DIP : ID 1 = 3, ID 2 = 2, ID 3 = 1, ID 4 = 0). En cas de doute, vérifier dans `Tab`, Dip Switches, « Network ID ».

### Configuration (`gti_link.ini`)
Les variables d'environnement ont priorité sur le fichier.

| Clé (ini) | Variable | Rôle |
|---|---|---|
| `peer` | `GTI_PEER` | `auto` (défaut) : diffusion sur le réseau local ; ou liste `ip[:port],ip[:port],...` des autres PC (port 5000 par défaut) ; `off` désactive le lien. |
| `port` | `GTI_PORT` | Port UDP d'écoute local (5000 par défaut). |
| `sync` | `GTI_SYNC` | `1` : verrouillage des machines cycle par cycle (défaut) ; `0` : désactivé. |
| `wait_ms` | `GTI_WAIT_MS` | Attente maximale d'un paquet, en ms (défaut 250, maximum 900). |

- Le fichier `gti_link.log` (créé dans le dossier courant) donne, chaque seconde : `cycles`, `sent`, `recv`, `stale`, `timeouts`, `dupid`, `peers`. `peers` liste les autres joueurs vus (par exemple `2,3`). Si `recv` reste à 0 : vérifier le pare-feu et le réseau, ou utiliser la liste d'IP. Si `dupid` est supérieur à 0, deux PC ont le même Network ID.
- Tous les PC doivent utiliser **la même version** de `gticlub.exe`.
- Le menu de service contient un écran NETWORK CHECK. D'après le manuel, à lancer sur toutes les machines pendant au moins une minute : le repère STATUS doit avancer régulièrement, ERROR rester sous 10 et DOWN à 0.
- Pour masquer l'écran d'avertissement MAME : `ui.ini` contenant `skip_warnings 1` (fourni dans le kit) et l'option `-skip_gameinfo` (déjà dans les lanceurs).

## 3. Compilation

Ordre : partir d'un `k056230.cpp` d'origine (`git checkout src/devices/machine/k056230.cpp`), puis :
```
python3 patch_gti.py
python3 patch_sync.py
python3 patch_defaults.py
python3 patch_ini.py
python3 patch_multi.py
```
(Jusqu'à `patch_ini.py` : version à deux joueurs, une seule adresse de voisin. `patch_multi.py` ajoute les 4 bornes et le mode sans IP.) Chaque script s'arrête avec « ancre introuvable » si le fichier ne correspond pas à la version de MAME prévue. Version de MAME utilisée : `ec20df4a93bef43a92d5974a3299e91f718b7451`.

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

### Garder la version 2 joueurs et la version 4 joueurs
Une seule source diffère (`k056230.cpp`). Ranger les deux versions dans `~/gticlub-variants` (`k056230_2j.cpp` et `k056230_4j.cpp`), puis :
```
bash build_variant.sh 4 win      # ou : 2 win, 4 linux, 2 linux
SRC=$HOME/gticlub-variants/bin/4j OUT=$HOME/gticlub-kit-4j bash make_kit.sh
```
Seul `k056230.cpp` est recompilé. Le kit est écrit pour la version 4 joueurs (la version 2 joueurs ne comprend pas `peer=auto`).

Facultatif : retirer le drapeau `MACHINE_NODEVICE_LAN` des lignes `GAME(...)` de `gticlub.cpp` pour supprimer l'avertissement « réseau non émulé ». Laisser les drapeaux `MACHINE_IMPERFECT_*`.

## 4. Fichiers du projet

| Fichier | Rôle |
|---|---|
| `patch_gti.py` | Transport UDP (Linux et Windows) dans un `k056230.cpp` d'origine. |
| `patch_sync.py` | Session, séquence, verrouillage, statistiques. |
| `patch_defaults.py` | Verrouillage et attente de 250 ms par défaut. |
| `patch_ini.py` | Lecture de `gti_link.ini`. |
| `patch_multi.py` | 2 à 4 bornes : liste de voisins ou diffusion automatique (`peer=auto`). |
| `build_variant.sh` | Compile la version 2 ou 4 joueurs, pour Windows ou Linux. |
| `make_kit.sh` | Assemble `GTIClub.zip`. |
| `deint.py` | Reconstitue `prg.bin` (ROM programme PowerPC) à partir des 4 fichiers entrelacés. |
| `funcs.py` | Retrouve les noms de fonctions laissés dans la ROM et désassemble une fonction par son nom. |
| `k056230.patch` | Le même changement en un seul fichier (`git diff`). |

## 5. Résultats de test

| Essai | Résultat |
|---|---|
| Deux instances sur le même PC | Fonctionne, sans saccades. |
| Deux PC, sans synchronisation | Détection et choix en multi OK ; NETWORK ERROR quelques secondes après le départ. |
| Deux PC, `sync=1`, `wait_ms=30` | Réseau sain (environ 30 cycles/s, aucun paquet perdu) mais le PC le plus lent a des à-coups supérieurs à 30 ms : désynchronisation, erreur. |
| Deux PC, `sync=1`, `wait_ms=250` | Fonctionne ; le PC le plus lent rame un peu mais la partie tient. |
| Quatre instances sur le même PC (version 4 joueurs) | Fonctionne très bien. |
| Plusieurs PC en réseau local câblé (version 4 joueurs) | Fonctionne. |
| Ajout d'un PC en Wi-Fi | Tous les joueurs ramentent (voir section 6). |
| Code réseau seul, 4 processus de test sur Linux | Trames des autres bornes intactes, 30 cycles/s, aucun paquet perdu ; un joueur lent ralentit tous les autres au même rythme ; un joueur qui disparaît coûte environ 0,75 s ; un Network ID en double est détecté (`dupid`) ; la diffusion automatique fonctionne entre processus d'un même PC. |

Cause probable de l'erreur sans verrouillage : la vitesse d'émulation du PC le plus lent passe sous 100 % pendant la course. C'est une conclusion tirée des logs, pas une mesure directe.

## 6. Limites

- La LANC n'est pas émulée (pas de CRC, pas de timing de bus) : le jeu reçoit simplement des trames dans les slots de ses voisins.
- **Réseau câblé recommandé** (si possible sur un switch), comme sur la borne d'origine. En Wi-Fi, la latence irrégulière fait attendre toutes les machines à chaque à-coup, donc tous les joueurs ramentent. Piste non essayée : une tolérance de quelques cycles de retard au lieu d'une attente à chaque cycle.
- Le mode `peer=auto` ne traverse pas les routeurs, les VPN ni les réseaux Wi-Fi qui isolent les appareils, et peut choisir la mauvaise carte réseau s'il y en a plusieurs. Dans ces cas, utiliser la liste d'adresses IP.
- Les écrans NETWORKING CONFIGURATION (« Data receiving failure ») et NETWORK CHECK du menu de service ont été observés avant la version synchronisée ; leur comportement avec les versions récentes n'est pas documenté ici.
- Aucune authentification ni chiffrement : à n'utiliser que sur un réseau de confiance.
- Si une borne disparaît en pleine partie, le jeu peut se figer environ trois quarts de seconde avant de reprendre ; ses dernières données restent dans la RAM réseau, le jeu n'est pas prévenu autrement.
- Un PC lent ralentit tous les joueurs en mode synchronisé. Pistes : build `OPTIMIZE=3`, options MAME `-autoframeskip` ou `-speed 0.7`.
- Non testé : Internet (latence), macOS.
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

A prototype that lets **2, 3 or 4 players** race together in **GTI Club** (Konami, MAME set `gticlub`), each player on their own MAME instance, on the same PC or on several PCs of a wired local network. The Konami 056230 network chip (LANC), which was only a stub in MAME, now exchanges the cabinet's link data over UDP packets.

**Status: working prototype, not a full LANC emulation.** Tested with 2 to 4 instances on one PC, and on several PCs connected through a wired local network. Wi-Fi is not supported (see section 6).

## 1. How it works

### Observed (MAME driver code, ROM disassembly, logs, manual)
- The driver `src/mame/konami/gticlub.cpp` uses the `k056230` chip (`src/devices/machine/k056230.cpp`). Before this patch its registers did nothing and no frame was exchanged (the set is flagged `MACHINE_NODEVICE_LAN`).
- Registers: base `0x7e008000` (byte 0: status on read, mode on write; byte 1: control; byte 2: sub ID). Network RAM at `0x7e00a000`. The chip raises the PowerPC's IRQ2.
- The "Network ID" DIP switch (1 to 4) selects the cabinet. Writes to the mode register were `0x08` / `0x28` for ID 1 and `0x0c` / `0x2c` for ID 3: the low bits equal `2 x (ID - 1)`.
- With the initial status at `0x08`, the game starts in single-player. With `0x00`, it shows NETWORK ERROR.
- The [Deluxe operator's manual](https://www.arcade-museum.com/manuals-videogames/G/GTI-Club.pdf) says a cabinet can be linked with **up to three other machines** (4 in total), each machine having its own Network ID, and that the network race starts automatically when another machine takes a coin during the network entry screen. That behaviour belongs to the game, not to this link.

### Working interpretation (inferred from the disassembly, not confirmed by documentation)
- Network RAM is split into 8 slots of 256 bytes; each cabinet owns two consecutive slots, the first being `2 x (ID - 1)`.
- Each useful frame is 128 bytes, framed by a 16-bit marker at the start (`+0x00`) and the end (`+0x7e`) of the slot, compared against a table in the ROM.
- Status bit `0x08` means "line free".
- One exchange cycle = a write of mode `0x28 | slot`, then control `0x37`.
- Master/slave roles (`NC_MAST`, `NC_SLAV`) are handled by the game itself.

### What the patch does
On each control write `0x37` (mode bit 5 set), before the IRQ is raised:
1. the cabinet sends its two slots (512 bytes) to the others over UDP: either as a **broadcast on the local network** (`peer=auto`, no IP address needed), or to a **list of addresses**;
2. it copies the slots received from each other cabinet into its network RAM;
3. in synchronized mode (on by default), it first waits up to `wait_ms` milliseconds (250 by default) for a fresh packet from **each** cabinet seen in the last 2 seconds, so all machines advance at the pace of the slowest one. A cabinet that causes three timeouts in a row is no longer waited for until it shows up again.

Packet format (524 bytes): `G`, `T`, `I`, the cabinet's slot number, a session ID (4 bytes), a sequence number (4 bytes), then 512 bytes of data.

## 2. Quick start (ready-to-use kit)

`make_kit.sh` builds `GTIClub.zip` from the compiled `gticlub.exe` (section 3). Contents: the exe, `gti_link.ini`, `joueur1.bat` to `joueur4.bat` (players 1 to 4), `test_meme_pc.bat` (single-PC test), `pare-feu.bat` (firewall), `ui.ini`, one configuration folder per player number (`cfg2` to `cfg4`), an empty `roms\` folder and a `LISEZMOI.txt` (French). **ROMs are not included.**

On each PC (wired local network, preferably through a switch):
1. Unzip the kit and put `gticlub.zip` (the ROMs) in `roms\`.
2. Right-click `pare-feu.bat`, then "Run as administrator" (once per PC).
3. Run `joueur1.bat` on PC 1, `joueur2.bat` on PC 2, and so on up to `joueur4.bat`: **each PC must have a different number**.
4. Insert a credit, pick a car and a track: the PCs detect each other.

If the PCs do not find each other (several network cards, a VPN, a segmented network), edit `gti_link.ini` and replace `peer=auto` with the list of IP addresses of the **other** PCs (for example `peer=192.168.1.20,192.168.1.30`). Get an IP with `ipconfig` ("IPv4 Address" line).

`test_meme_pc.bat` opens two players in two windows on a single PC (explicit local addresses).

Each launcher uses its own configuration folder (`cfg1` to `cfg4`). The kit's `cfg2` to `cfg4` force Network ID 2, 3 or 4; they were written from the MAME driver (DIP values: ID 1 = 3, ID 2 = 2, ID 3 = 1, ID 4 = 0). If in doubt, check `Tab`, Dip Switches, "Network ID".

### Configuration (`gti_link.ini`)
Environment variables take priority over the file.

| ini key | Variable | Purpose |
|---|---|---|
| `peer` | `GTI_PEER` | `auto` (default): broadcast on the local network; or a list `ip[:port],ip[:port],...` of the other PCs (port 5000 by default); `off` disables the link. |
| `port` | `GTI_PORT` | Local UDP listening port (5000 by default). |
| `sync` | `GTI_SYNC` | `1`: lock the machines cycle by cycle (default); `0`: disabled. |
| `wait_ms` | `GTI_WAIT_MS` | Maximum wait for a packet, in ms (default 250, maximum 900). |

- The `gti_link.log` file (created in the current folder) records, every second: `cycles`, `sent`, `recv`, `stale`, `timeouts`, `dupid`, `peers`. `peers` lists the other players seen (for example `2,3`). If `recv` stays at 0, check the firewall and the network, or use the IP list. If `dupid` is above 0, two PCs share the same Network ID.
- All PCs must run **the same version** of `gticlub.exe`.
- The service menu has a NETWORK CHECK screen. According to the manual, run it on all machines for at least one minute: the STATUS marker should move steadily, ERROR stay below 10 and DOWN at 0.
- To hide MAME's warning screen: `ui.ini` containing `skip_warnings 1` (included in the kit) and the `-skip_gameinfo` option (already in the launchers).

## 3. Building

Order: start from an unmodified `k056230.cpp` (`git checkout src/devices/machine/k056230.cpp`), then:
```
python3 patch_gti.py
python3 patch_sync.py
python3 patch_defaults.py
python3 patch_ini.py
python3 patch_multi.py
```
(Up to `patch_ini.py`: the two-player version, a single peer address. `patch_multi.py` adds up to 4 cabinets and the no-IP mode.) Each script stops with "ancre introuvable" (anchor not found) if the file does not match the expected MAME version. MAME version used: `ec20df4a93bef43a92d5974a3299e91f718b7451`.

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

### Keeping both the 2-player and the 4-player version
Only one source differs (`k056230.cpp`). Store both versions in `~/gticlub-variants` (`k056230_2j.cpp` and `k056230_4j.cpp`), then:
```
bash build_variant.sh 4 win      # or: 2 win, 4 linux, 2 linux
SRC=$HOME/gticlub-variants/bin/4j OUT=$HOME/gticlub-kit-4j bash make_kit.sh
```
Only `k056230.cpp` is recompiled. The kit is written for the 4-player version (the 2-player version does not understand `peer=auto`).

Optional: remove the `MACHINE_NODEVICE_LAN` flag from the `GAME(...)` lines in `gticlub.cpp` to drop the "network not emulated" warning. Leave the `MACHINE_IMPERFECT_*` flags alone.

## 4. Project files

| File | Purpose |
|---|---|
| `patch_gti.py` | UDP transport (Linux and Windows) added to an unmodified `k056230.cpp`. |
| `patch_sync.py` | Session, sequence numbers, lock-step synchronization, statistics. |
| `patch_defaults.py` | Synchronization and the 250 ms wait on by default. |
| `patch_ini.py` | Reads `gti_link.ini`. |
| `patch_multi.py` | 2 to 4 cabinets: peer list or automatic broadcast (`peer=auto`). |
| `build_variant.sh` | Builds the 2- or 4-player version, for Windows or Linux. |
| `make_kit.sh` | Builds `GTIClub.zip`. |
| `deint.py` | Rebuilds `prg.bin` (PowerPC program ROM) from the 4 interleaved files. |
| `funcs.py` | Finds the function names Konami left in the ROM and disassembles a function by name. |
| `k056230.patch` | The same change as a single file (`git diff`). |

## 5. Test results

| Test | Result |
|---|---|
| Two instances on the same PC | Works, no stutter. |
| Two PCs, no synchronization | Detection and multiplayer selection OK; NETWORK ERROR a few seconds after the start. |
| Two PCs, `sync=1`, `wait_ms=30` | Healthy network (about 30 cycles/s, no lost packets) but the slower PC stutters by more than 30 ms: the machines drift apart, error. |
| Two PCs, `sync=1`, `wait_ms=250` | Works; the slower PC lags a little but the race holds. |
| Four instances on the same PC (4-player version) | Works very well. |
| Several PCs on a wired local network (4-player version) | Works. |
| Adding a PC on Wi-Fi | Every player lags (see section 6). |
| Network code alone, 4 test processes on Linux | Other cabinets' frames intact, 30 cycles/s, no lost packets; one slow player slows everyone to the same pace; a player dropping out costs about 0.75 s; a duplicate Network ID is detected (`dupid`); automatic broadcast works between processes on one PC. |

Probable cause of the error without synchronization: the slower PC's emulation speed drops below 100% during the race. This is a conclusion drawn from the logs, not a direct measurement.

## 6. Limitations

- The LANC is not emulated (no CRC, no bus timing): the game simply receives frames in its neighbours' slots.
- **A wired network is recommended** (preferably through a switch), as on the original cabinets. On Wi-Fi, irregular latency makes every machine wait at each hiccup, so every player lags. Idea not tried yet: tolerate a few cycles of delay instead of waiting every cycle.
- `peer=auto` does not cross routers or VPNs, nor Wi-Fi networks that isolate clients, and may pick the wrong network card if there are several. In those cases, use the IP address list.
- The NETWORKING CONFIGURATION ("Data receiving failure") and NETWORK CHECK screens of the service menu were observed before the synchronized version existed; their behaviour with recent versions is not documented here.
- No authentication or encryption: use only on a trusted network.
- If a cabinet disappears mid-game, the game may freeze for about three quarters of a second before resuming; its last data stays in the network RAM and the game is not otherwise notified.
- A slow PC slows every player down in synchronized mode. Ideas: an `OPTIMIZE=3` build, MAME options `-autoframeskip` or `-speed 0.7`.
- Not tested: the Internet (latency), macOS.
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
