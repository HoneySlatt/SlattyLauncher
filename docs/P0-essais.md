# P0 — essais réels contre GOG

Ces essais confirment l'intégration réelle ; les tests automatisés ne vérifient que notre logique.
Consigner chaque résultat (réussi ou non) dans `docs/compat.md`.

Préparation :

```bash
cd ~/Projects/SlattyLauncher
nix develop
cargo build
# `nix develop` puts target/debug in PATH: `slatty` and `slatty-gui` are available after `cargo build`.
slatty doctor
```

Heroic ne doit pas avoir de jeu GOG en cours pendant les essais (port Comet 9977, sauvegardes partagées).

## 1. Connexion

```bash
slatty auth login
slatty auth status
ps -eo args | grep -c refresh_token   # attendu : 1 (la ligne grep elle-même)
slatty auth refresh
slatty auth probe-rotation
```

À relever : le code est-il accepté ; durée de validité ; `rotated` / `previous refresh token still accepted`.
Ne jamais coller l'URL ou un jeton ailleurs que dans le terminal de `slatty auth login`.

## 2. Bibliothèque hors ligne

```bash
slatty library sync
slatty library list
# couper le réseau
slatty library list witcher
```

## 3. Import (lecture seule de Heroic)

```bash
slatty import --from-heroic 1456487183   # Undertale
slatty import --from-heroic 1724969043   # Tomb Raider
slatty launch-spec 1724969043
```

## 4. Session

```bash
slatty launch 1724969043 --no-cloud --no-comet
```

Quitter le jeu normalement. Attendu : « Session ended » seulement après la fermeture complète
(le lanceur `-nolauncher` est contourné ; refaire avec un jeu à lanceur intermédiaire, ex. Cyberpunk).

## 5. Cloud — Undertale (petites sauvegardes)

Sauvegarde manuelle préalable du dossier local s'il existe :
`~/Documents/Heroic/Prefixes/Undertale/drive_c/users/steamuser/AppData/Local/UNDERTALE`.

```bash
slatty cloud status 1456487183     # rien n'est modifié
slatty cloud sync 1456487183
slatty cloud status 1456487183     # attendu : up to date / 0 action
```

Aller-retour : modifier la sauvegarde en jouant (`slatty launch 1456487183 --no-comet`), vérifier l'envoi ;
puis, depuis Heroic ou une autre machine, une modification distante doit être téléchargée avec une copie
de l'ancienne version dans `~/.local/share/slatty/backups/`.

À vérifier en particulier : le contenu téléchargé est identique octet pour octet à l'original
(compression gzip gérée correctement).

## 6. Achievement réel — Tomb Raider (ou DOOM 2016)

```bash
slatty achievements 1724969043
slatty launch 1724969043
# jouer jusqu'à obtenir un achievement non encore débloqué
```

Attendu en fin de session : « Achievement unlocked on GOG: … ». Puis vérifier sur le profil GOG
(site web) que l'achievement apparaît. `~/.local/state/slatty/logs/comet.log` aide en cas d'échec
(il peut contenir des identifiants de jeu : ne pas le publier tel quel).
