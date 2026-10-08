# SlattyLauncher — plan

Lanceur GOG natif (Rust + Iced 0.14), Linux/Wayland/NixOS d'abord, sans client Galaxy, sans WebView.
Aucune API GOG n'est officielle pour un lanceur tiers : tout ce qui suit repose sur la rétro-ingénierie
du client Galaxy et peut casser sans préavis.

Légende : **[V]** vérifié (code lu ou test exécuté) · **[C]** documentation communautaire / RE · **[S]** supposé · **[?]** inconnu.

## Décisions

| Sujet | Décision |
|---|---|
| Licence | GPL-3.0-or-later (reprise possible de logique gogdl/Heroic, GPL-3.0, avec attribution) |
| Connexion | navigateur externe, l'utilisateur colle l'URL finale (`embed.gog.com/on_login_success?...code=`) — seul mécanisme vérifié sans WebView |
| Jetons | trousseau système (`keyring` v4 → Secret Service) ; sans trousseau : erreur explicite, jamais de fichier en clair |
| Runner | `umu-run` + Proton déjà présent ; préfixe par jeu, réutilisation possible d'un préfixe Heroic (Heroic jamais modifié) |
| Fin de session | superviseur dédié `PR_SET_CHILD_SUBREAPER`, fin = plus aucun descendant |
| Cloud | synchro par dossiers (remote-config `cloudStorage`), pas d'IStorage ; base de référence par fichier (SHA-256 local + hash distant) |
| Achievements | Comet en processus compagnon supervisé ; jetons transmis par fichier privé (0600, `$XDG_RUNTIME_DIR`), jamais en argv |
| Installation | Galaxy v2 (génération 2, Windows, jeu de base, une langue) au jalon M3 |
| Stockage | `config.toml` (utilisateur) · `state.db` SQLite (indispensable) · `~/.cache/slatty/<user_id>/` (reconstructible) |

## Architecture

```
crates/core   logique complète, testable sans interface
crates/cli    `slatty` : diagnostics, essais d'intégration, mode superviseur caché
crates/gui    `slatty-gui` (M2) : Iced, consomme uniquement l'API du cœur
```

Modules du cœur : `auth`, `account`, `credentials`, `http`, `db`, `library`, `gameinfo`, `install`,
`runner`, `session`, `cloud::{locations, scan, plan, transport, sync}`, `comet`.

## Politique cloud

- Changement local = SHA-256 ≠ base ; changement distant = hash distant ≠ base. Pas de comparaison de dates.
- Avant lancement : distant seul modifié → téléchargement ; les deux → lancement bloqué, choix explicite.
- Après la fin réelle de la session : nouvel état distant relu, puis envoi.
- Tout remplacement local : copie de sauvegarde horodatée, puis fichier temporaire + `rename`.
- Conflit : les deux versions conservées (sauvegarde locale + téléchargement de la version distante dans le dossier de conflits).
- Dossier local absent ou vide alors que la base n'est pas vide → plan « suspect », aucune suppression.
- Liste distante vide alors que la base n'est pas vide → plan « suspect », aucune suppression locale.
- Suppressions propagées seulement fichier par fichier et sur confirmation explicite.
- Hors ligne : rien n'est modifié localement, la base n'avance pas.
- Un verrou par (compte, jeu).
- Aucune fusion de fichiers binaires.

## Jalons

| Jalon | Résultat observable | Taille |
|---|---|---|
| M0 | devShell Nix, squelette, `slatty doctor` | S |
| P0 | CLI : connexion, bibliothèque, import, lancement supervisé, cloud, Comet — sur un jeu déjà installé | L |
| M1 | cœur consolidé, reprise, suite de tests anti-perte de données | M/L |
| M2 | interface Iced : bibliothèque, fiche, Jouer, cloud/conflits, achievements, paramètres | M |
| M3 | installation Galaxy v2 d'un jeu | L |
| M4 | distribuable : mises à jour, réparation, DLC, langues, désinstallation, plusieurs runners | L |

### Critères P0 (essais réels, consignés dans `docs/compat.md`)

1. `slatty auth login` : connexion, jetons dans le trousseau, renouvellement, aucun secret dans les logs ni dans `ps`.
2. `slatty library sync` puis `slatty library list` hors ligne.
3. `slatty import --from-heroic <id>` ou `slatty import <dossier>`.
4. `slatty launch` : la session ne se termine qu'après le dernier processus (test automatisé + jeu à lanceur intermédiaire).
5. `slatty cloud status|sync` : aller-retour réel vérifié par SHA-256, sauvegarde locale créée, dossier vide → aucune suppression.
6. Achievement obtenu en jouant, visible ensuite sur le compte GOG.

### État P0 (code terminé, essais réels à faire — voir `docs/P0-essais.md`)

Vérifié localement :
- Superviseur : double fork, `setsid`, arrêt sur demande (tests automatisés) ; essai réel umu + Proton-CachyOS
  dans un préfixe jetable, depuis `$HOME` et avec un dossier de travail sur `/NAS` [V].
- umu (`waitforexitandrun`) attend déjà les processus Windows détachés ; le subreaper reste un filet de sécurité [V].
- pressure-vessel ne voit pas les dossiers hors des chemins partagés (ex. `/tmp/nix-shell.*`) ;
  `STEAM_COMPAT_INSTALL_PATH` est défini comme Heroic le fait [V].
- Import Heroic en lecture seule sur Tomb Raider, Undertale, DOOM (2016) [V].
- Moteur cloud : 20 scénarios simulés (conflits, suppressions, réseau, compte, dossier déplacé, casse, évasion de chemin, verrou) [V, simulé].
- Comet : jetons hors argv, fichier supprimé dès l'écoute, arrêt par SIGINT, nettoyage (faux jetons) [V].

Non vérifié contre GOG : tout le reste (connexion, bibliothèque, cloud réel, achievements).

### État M2 (premier jet)

Interface Iced : connexion (navigateur + collage), bibliothèque avec recherche et jaquettes en cache disque,
fiche de jeu, Jouer/Arrêter via le même flux que la CLI, état cloud (vérifier, synchroniser, garder local/cloud),
achievements lus sur GOG, Tab / Maj+Tab / Échap. Tests sans affichage (`iced_test`) avec données marquées « [FICTIF] » ;
démarrage réel vérifié sous niri (Wayland).
Non fait : liste des téléchargements (rien à télécharger avant M3), paramètres, import depuis l'interface,
mesure avec une très grande bibliothèque, mise à l'échelle fractionnaire, manette.

### Ajout au périmètre : gestion manuelle des achievements (demandée le 2026-10-09)

`slatty achievements <id> --unlock|--unlock-all|--clear` et boutons Débloquer / Réinitialiser / Tout débloquer
dans la fiche de jeu, pour tout jeu possédé (installé ou non). Mécanisme [C, Comet + gog_achievements] :
`POST gameplay.gog.com/clients/{client}/users/{user}/achievements/{id}` avec `{"date_unlocked": date|null}`,
jeton du client Galaxy du jeu. Confirmation obligatoire avant toute écriture. Visible sur le profil public,
probablement contraire aux conditions de GOG. Ne remplace pas le critère P0 (achievement obtenu en jouant via Comet).

### État M3 (code terminé, essai réel à faire)

`slatty install` et section Installation de l'interface : build Windows génération 2, jeu de base, une langue.
Téléchargement dans `.<dossier>.slatty-partial`, MD5 compressé et décompressé par morceau, publication par
`rename` uniquement si tout est vérifié, reprise par revérification, tâche persistée (`install_jobs`),
espace disque vérifié, chemins dangereux refusés, annulation. Préfixe créé par `wineboot` au premier lancement
(vérifié réellement avec Proton-CachyOS). 11 tests simulés de l'installateur [V, simulé].
Non géré : fichiers « support » (scripts d'installation GOG, dont d'éventuelles clés de registre), redistribuables,
DLC, liens symboliques, conteneurs « small files » sans morceaux, mises à jour, réparation, désinstallation.

## Inconnues dominantes

- ~~Rotation du refresh token~~ : vérifié, pas de rotation, l'ancien jeton reste accepté [V]. Acceptation d'un POST sur `/token` : non testé.
- Sémantique du hash `aadd86936a80ee8a369579c3926f1b3c` : ni contenu vide ni gzip vide ; entrées ignorées comme gogdl.
- Envoi conditionnel (`If-Match`) non connu : une modification distante arrivant entre la relecture et l'envoi
  est écrasée (fenêtre de l'ordre de la durée d'un envoi ; test `residual_window_after_fresh_listing_is_not_detected`).
- ~~Décompression des téléchargements cloud~~ : vérifiée sur Tomb Raider [V].
- Service factice `GalaxyCommunication.exe` non géré en P0 (Kingdom Come, Cuphead, DOOM 3 en ont besoin).
- User-Agent Galaxy nécessaire ou non pour `cloudstorage.gog.com` (repris de gogdl par prudence).

## Dépendances non Rust

Proton + umu-launcher (Python), Comet (binaire externe), `GalaxyCommunication.exe` factice (C/mingw, certains jeux),
Secret Service (gnome-keyring), SQLite (C, intégré), pile Vulkan/Wayland, `xdg-open`.
