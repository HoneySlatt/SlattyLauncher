# Compatibilité vérifiée

Une ligne par essai réel contre GOG. Un essai simulé ne figure jamais ici.

| Date | Jeu (id) | Build | Plateforme / runner | Comet | Fonction | Résultat | Notes |
|---|---|---|---|---|---|---|---|
| 2026-10-08 | — | — | — | — | Connexion (navigateur + URL collée) | OK | Jetons dans gnome-keyring ; aucun jeton visible dans `ps` |
| 2026-10-08 | — | — | — | — | Renouvellement de session | OK | `expires_in` ≈ 1 h ; refresh token **non** remplacé, l'ancien reste accepté |
| 2026-10-08 | — | — | — | — | Bibliothèque | OK | 71 jeux, métadonnées gamesdb complètes ; liste depuis le cache |
| 2026-10-08 | Tomb Raider (1724969043) | 1.0 | Windows / umu + Proton-CachyOS, préfixe Heroic | — | Import Heroic + session | OK | `-nolauncher` ; fin détectée après le dernier processus (67 s) |
| 2026-10-08 | Tomb Raider (1724969043) | 1.0 | Windows | — | Liste des achievements | OK | 0/35, jeton du client du jeu accepté par gameplay.gog.com |
| 2026-10-08 | Tomb Raider (1724969043) | 1.0 | Windows | — | Cloud, première synchro | Conflit (attendu) | `saves/profile.dat` local (306 o, 08/10) ≠ cloud (301 o, 03/10), sans historique commun |
| 2026-10-08 | Tomb Raider (1724969043) | 1.0 | Windows | — | Cloud, téléchargement | OK | `slatty cloud diff` : contenu reçu décompressé (pas d'en-tête gzip) |
| 2026-10-08 | Tomb Raider (1724969043) | 1.0 | Windows | — | Cloud, envoi (`--prefer local`) | OK | Version cloud précédente conservée dans les sauvegardes ; `cloud status` après envoi encore à vérifier |
| 2026-10-08 | Undertale (1456487183) | 1.08 | Windows | — | Cloud | Non testable | Aucune sauvegarde ni en local ni dans le cloud |
