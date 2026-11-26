# Libérer le disque WebLogic lorsqu’un log supprimé reste ouvert

## 1. Contexte de l’incident
Le 24 novembre 2026, la partition `/opt` du serveur WebLogic / OSB (`srv-appmid-wl`, instance `OSBSERVER1`) était saturée à 100 %. Le fichier responsable était `OSBSERVER1.out`. Les chiffres ci-dessous reprennent le compte rendu évoqué précédemment ; ils ne constituent pas une nouvelle mesure du serveur.

## 2. Vérifier le disque et les inodes
```bash
df -h /opt
```
`df` mesure les blocs occupés sur le système de fichiers contenant `/opt`. L’option `-h` affiche les tailles lisibles. Résultat rapporté : capacité 397 Go, 377 Go utilisés, aucun espace disponible, 100 %.
```bash
df -i /opt
```
`-i` affiche les inodes (structures de métadonnées des fichiers). Avec 12 % d’inodes utilisés, la saturation concerne les blocs de stockage et non le nombre de fichiers.

## 3. Comprendre pourquoi df et du ne correspondent pas
```bash
du -xh --max-depth=1 /opt | sort -hr
```
| Élément | Signification |
|---|---|
| `du` | Mesure les blocs occupés par les fichiers accessibles par leur nom. |
| `-x` | Reste sur le même système de fichiers. |
| `-h` | Affiche des tailles lisibles. |
| `--max-depth=1` | Affiche les totaux au premier niveau. |
| `\|` | Transmet la sortie à sort. |
| `sort -hr` | Trie les tailles lisibles dans l’ordre décroissant. |
Résultat rapporté : environ 31 Go visibles contre 377 Go utilisés selon df. Cet écart d’environ 346 Go oriente vers un fichier supprimé mais ouvert. D’autres causes existent : fichiers masqués par un montage, métadonnées ou permissions insuffisantes.

## 4. Rechercher les fichiers supprimés encore ouverts
```bash
lsof -nP +L1
```
| Élément | Signification |
|---|---|
| `lsof` | Liste les fichiers ouverts par les processus. |
| `-n` | Ne résout pas les adresses IP en noms. |
| `-P` | Ne convertit pas les ports en noms de services. |
| `+L1` | Sélectionne les fichiers ouverts dont le nombre de liens est inférieur à 1 (zéro). |
Le fichier `OSBSERVER1.out` apparaissait comme supprimé (`deleted`) mais restait ouvert par `startWebLogic.sh` (PID 204032) et Java (PID 204099). Sa taille rapportée était d’environ 346 Go. Ne pas additionner deux lignes lsof si elles désignent le même inode : il s’agit du même fichier.

## 5. Comprendre les descripteurs de fichiers
Un descripteur est un numéro utilisé par un processus pour accéder à un fichier ouvert. `0` désigne généralement l’entrée standard, `1` la sortie standard (stdout), `2` la sortie d’erreur (stderr). Linux expose ces références dans `/proc/PID/fd/`.
```bash
ls -l /proc/204032/fd/1
```
`204032` est le PID observé lors de cet incident, et `1` le descripteur de sortie standard. Ces valeurs changent selon le processus : ne jamais les réutiliser sans vérification. Le lien peut pointer vers le journal supprimé. Vérifier également le type de fichier, le périphérique et l’inode, ainsi que les processus qui le détiennent.
```bash
du -hL /proc/204032/fd/1
```
`-L` suit le lien symbolique vers le fichier réel ; `-h` rend la taille lisible.
```bash
tail -n 30 /proc/204032/fd/1
```
`tail -n 30` lit les 30 dernières lignes. Ne consulter que si le descripteur désigne bien un fichier régulier et si l’accès aux logs est autorisé.

## 6. Pourquoi supprimer un log ne libère pas toujours son espace
La suppression retire le nom du fichier du répertoire. Tant qu’un processus conserve une référence ouverte, Linux garde les blocs associés. `du` ne voit plus le nom, mais `df` comptabilise encore les blocs. L’espace est libéré quand la dernière référence est fermée, ou lorsque le fichier régulier est tronqué.

## 7. Action corrective réalisée — intervention destructive
La solution à privilégier est une réouverture des logs prise en charge par le service, ou un redémarrage contrôlé si nécessaire. Lors de cet incident, une troncature du fichier encore ouvert a été rapportée, sans redémarrage WebLogic :
```bash
> /proc/204032/fd/1
```
Le symbole `>` est une redirection du shell : sans commande produisant du texte, il ouvre le fichier cible et le tronque à zéro octet. Il ne supprime pas le descripteur. Cette action détruit le contenu du journal ; elle n’est pas une commande de Health Check ordinaire. Avant toute répétition : validation du responsable, préservation des preuves selon la rétention, confirmation du PID actuel, du descripteur, du type régulier et de l’inode. Un processus peut continuer à écrire dans le fichier invisible, et une écriture sans mode append peut recréer un fichier creux de grande taille. Prévoir une réouverture normale des logs et une correction durable. Ne jamais appliquer à des fichiers de données ou à un descripteur inconnu.

## 8. Vérifier le résultat
```bash
df -h /opt
```
| Indicateur | Avant | Après |
|---|---|---|
| Utilisation de /opt | 100 % | 8 % |
| Espace utilisé | 377 Go | 29 Go |
| Log ouvert | environ 346 Go | environ 5,5 Mo |
| Redémarrage WebLogic | Non | Non |
L’espace disponible rapporté après intervention était de 348 Go. Vérifier aussi les processus, les ports attendus, les nouveaux logs et la disponibilité du service. La validation fonctionnelle et la cause initiale de croissance du journal restaient à confirmer.

## 9. Prévenir la récidive
Contrôler la rotation du journal stdout de WebLogic, sa rétention, les alertes de croissance et les erreurs répétitives. Éviter la suppression manuelle des logs actifs. Documenter l’intervention et escalader vers le responsable WebLogic pour la correction durable.
