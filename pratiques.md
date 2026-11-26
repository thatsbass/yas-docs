# Libérer le disque SQL Server (backups DIFF)

## 1. Situation

La partition SQL Server était à 95 %. Le périmètre approuvé concernait les sauvegardes différentielles de mai 2026, pas les fichiers de données ni les autres backups.

## 2. Localiser la consommation

Identifier le filesystem concerné.

```
df -h /var/opt/mssql
```

Comparer data, log et backup. Ne jamais supprimer manuellement les MDF, NDF ou LDF actifs.

```
du -xhd1 /var/opt/mssql | sort -hr
```

## 3. Sélectionner uniquement les DIFF de mai

Afficher les chemins correspondant au motif glob (pas une regex).

```
find /var/opt/mssql/backup -type f -name '*_DIFF_202605*.bak' -print
```

Exemple fictif de résultat

```
/var/opt/mssql/backup/CRM_DIFF_20260512.bak
```

| Élément | Signification |
| --- | --- |
| `*` | Zéro ou plusieurs caractères. |
| `_DIFF_` | Texte exact désignant le type différentiel selon la convention. |
| `202605` | Année 2026 et mois 05. |
| `*.bak` | Suite éventuelle de caractères puis extension .bak. |
| `'...'` | Empêche le shell de développer le motif avant find. |
| `-print` | Affiche le chemin, sans supprimer. |

## 4. Compter et mesurer

wc -l compte les lignes (un chemin par ligne pour les noms habituels).

```
find /var/opt/mssql/backup -type f -name '*_DIFF_202605*.bak' -print | wc -l
```

%s affiche la taille en octets ; awk additionne la première colonne ($1), puis convertit en Gio à la fin.

```
find /var/opt/mssql/backup -type f -name '*_DIFF_202605*.bak' -printf '%s\n' | awk '{s+=$1} END {printf "%.2f GiB\n", s/1024/1024/1024}'
```

## 5. Valider puis contrôler

Faire valider la liste exacte et la chaîne de restauration par le DBA avant toute suppression. Refaire les commandes de comptage et df -h après l’intervention. Résultat constaté : **95 % → 93 % → 78 %**.

# Trier les backups Talend par mois

## 1. Comprendre la situation

Lors de l’intervention Talend ESB, le disque était occupé à 90 %. La consigne validée consistait à retirer les sauvegardes de février à juin 2026. Le chemin ci-dessous est illustratif : confirmer le répertoire réel.

## 2. Identifier le stockage concerné

Vérifier le taux d’occupation du système de fichiers contenant les sauvegardes.

```
df -h /opt/talend/backup
```

Mesurer les répertoires sur le même filesystem, puis classer les tailles par ordre décroissant.

```
du -xhd1 /opt/talend/backup | sort -hr
```

## 3. Rechercher les sauvegardes par mois

Lister uniquement les noms des fichiers, puis retenir ceux qui contiennent une date correspondant à la période.

```
find /opt/talend/backup -type f -printf '%f\n' | grep -E '2026(02|03|04|05|06)[0-9]{2}'
```

Exemple fictif de résultat

```
backup_20260218.zip
backup_20260410.zip
backup_20260625.zip
```

| Élément | Signification |
| --- | --- |
| `find /opt/talend/backup` | Explorer ce répertoire et ses sous-répertoires. |
| `-type f` | Retenir uniquement les fichiers ordinaires. |
| `-printf '%f\n'` | %f affiche le nom seul ; \n ajoute un retour à la ligne. |
| `|` | Transmet la sortie de find à grep. |
| `grep -E` | Filtre les lignes avec une expression régulière étendue. |
| `2026` | Année littérale. |
| `(02|03|04|05|06)` | Un des cinq mois autorisés ; | signifie OU dans la regex. |
| `[0-9]{2}` | Exactement deux chiffres consécutifs, représentant le jour. |

Le motif cherche une sous-chaîne : il peut accepter une date impossible (20260299) ou une suite de chiffres plus longue. Il ne prouve pas que le fichier est une sauvegarde. Vérifier la convention réelle des noms et l’extension.

## 4. Tester le pattern sans toucher au serveur

Expression régulièreNoms de fichiers (un par ligne)backup\_20260115.zip
backup\_20260218.zip
backup\_20260320.zip
backup\_20260625.zip
backup\_20260701.zip
backup\_20260299.zip

Simulation JavaScript des motifs simples de grep -E ; ce n’est pas une exécution Linux.

## 5. Adapter le filtre et compter les fichiers

grep -o affiche uniquement le fragment trouvé ; sort regroupe les mois identiques ; uniq -c compte les occurrences adjacentes. Ce comptage suppose une seule date pertinente par nom.

```
find /opt/talend/backup -type f -printf '%f\n' | grep -E '2026(02|03|04|05|06)[0-9]{2}' | grep -oE '2026(02|03|04|05|06)' | sort | uniq -c
```

## 6. Prévisualiser les chemins avant toute correction

Exemple si les sauvegardes sont des ZIP : afficher les chemins complets, confirmer les dates et la rétention. Ne pas supprimer à partir de la seule liste de noms.

```
find /opt/talend/backup -type f -name '*.zip' -print
```

Après validation du propriétaire des sauvegardes, supprimer uniquement la liste exacte approuvée selon la procédure interne. Vérifier ensuite avec df -h et contrôler les services concernés.

## 7. Résultat constaté

**90 % → 53 %** : réduction de 37 points de pourcentage. Les noms de fichiers présentés ici sont fictifs.

# Réduire la taille des journaux WSO2 / Grafana

## 1. Identifier les journaux volumineux

-x limite au filesystem ; -h rend les tailles lisibles ; -d1 limite la profondeur ; sort -hr trie les tailles humaines en ordre inverse.

```
du -xhd1 /var/log | sort -hr
```

Mesurer la place occupée par les journaux systemd.

```
journalctl --disk-usage
```

## 2. Lire les événements avant nettoyage

Afficher les cinq premières lignes pour reconnaître le format et les dates.

```
head -n 5 /var/log/messages
```

Consulter les erreurs récentes. Préserver les événements utiles à un incident.

```
journalctl -p err --since "24 hours ago"
```

## 3. Comprendre la rotation et la purge

Action de maintenance déjà étudiée sur le serveur WSO2/Grafana : uniquement après autorisation et validation de la rétention.

```
journalctl --rotate --vacuum-size=1G
```

| Élément | Signification |
| --- | --- |
| `--rotate` | Archive les journaux actifs et ouvre de nouveaux journaux. |
| `--vacuum-size=1G` | Supprime les plus anciens journaux archivés pour viser au maximum 1 Gio d’archives. |
| `1G` | Limite de taille, pas une durée de conservation. |

La taille totale peut dépasser 1 Gio à cause des journaux actifs. Cette commande ne nettoie pas /var/log/messages ni les logs applicatifs indépendants.

## 4. Vérifier après intervention

Comparer les valeurs avant/après. Aucun chiffre de résultat n’est fourni pour ce cas.

```
journalctl --disk-usage
df -h
```

# Retrouver un fichier et comprendre les filtres

## 1. Retrouver un script

Rechercher le nom exact dans /opt. Élargir seulement si nécessaire, pour limiter les I/O.

```
find /opt -type f -name 'mesure.sh'
```

## 2. Trouver les fichiers volumineux

Chercher les fichiers de plus de 1 Gio sur le filesystem de /var.

```
find /var -xdev -type f -size +1G -print
```

| Élément | Signification |
| --- | --- |
| `-xdev` | Ne pas traverser vers un autre filesystem. |
| `-size +1G` | Taille arrondie en unités Gio, supérieure à une unité. |
| `-name` | Filtre sur le nom via un glob. |
| `-iname` | Même filtre, sans distinction majuscule/minuscule. |

## 3. Distinguer glob et regex

| Élément | Signification |
| --- | --- |
| `*.log` | Glob find : tout nom se terminant par .log. |
| `.*\.log$` | Regex grep : tout texte se terminant par .log ; \. représente le point littéral. |
| `^backup_` | Regex : début de ligne backup\_. |
| `[0-9]{2}` | Regex étendue : deux chiffres exactement. |

## 4. Rechercher par ancienneté

Sélectionne les fichiers dont l’âge de modification, arrondi en périodes de 24 h, dépasse 30. Ce n’est pas la date de création.

```
find /backup -type f -mtime +30 -print
```

Ces recherches sont en lecture seule. Les chemins et résultats sont à adapter au serveur.