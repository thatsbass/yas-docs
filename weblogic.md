# Libérer le disque WebLogic (log supprimé mais ouvert)

Incident du 24 novembre 2026 — srv-appmid-wl, WebLogic / OSB, OSBSERVER1. Les résultats ci-dessous sont des extraits ou synthèses des observations retrouvées, pas une transcription intégrale du terminal. Les champs absents sont signalés.

## 1. Confirmer la saturation

```bash
df -h /opt
```

**Résultat observé — extrait ou synthèse**

```text
/dev/vda1  397G  377G  0  100%  /opt
```

**Interprétation :** La partition est pleine. Aucun espace disponible pour les écritures ordinaires.

## 2. Vérifier les inodes

```bash
df -i /opt
```

**Résultat observé — extrait ou synthèse**

```text
IUse% : 12 %
```

**Interprétation :** Les inodes ne sont pas saturés. Le problème concerne les blocs de stockage.

## 3. Chercher les répertoires volumineux

```bash
du -xh --max-depth=1 /opt | sort -hr | head -10
```

**Résultat observé — extrait ou synthèse**

```text
31G  /opt
25G  /opt/Oracle
```

**Interprétation :** df indique 377G mais du ne retrouve que 31G : environ 346G restent inexpliqués. -x limite au même système de fichiers ; -h affiche les tailles lisibles ; sort -hr trie du plus grand au plus petit ; head limite à dix lignes.

## 4. Identifier les fichiers supprimés encore ouverts

```bash
lsof -nP +L1 | head -30
```

**Résultat observé — extrait ou synthèse**

```text
COMMAND   PID     USER   FD  TYPE DEVICE SIZE/OFF      NLINK NODE
startWebL 204032  wlapp  1w  REG  252,1  371276582912  0     9964460
java      204099  …      …   …    …      …             …     9964460
NAME : …/OSBSERVER1/logs/OSBSERVER1.out (deleted)
```

**Interprétation :** +L1 sélectionne les fichiers sans lien (NLINK=0). -n désactive la résolution DNS et -P la conversion des ports. Les deux processus référencent le même inode 9964460 : ne pas additionner leurs tailles.

## 5. Identifier les processus

```bash
ps -fp 204032,204099
```

**Résultat observé — extrait ou synthèse**

```text
PID 204032 : startWebLogic.sh
PID 204099 : Java
```

**Interprétation :** -f affiche les détails ; -p sélectionne les PID. Les deux processus appartiennent à WebLogic.

## 6. Vérifier le descripteur ouvert

```bash
ls -l /proc/204032/fd/1
```

**Résultat observé — extrait ou synthèse**

```text
/proc/204032/fd/1 -> …/OSBSERVER1.out (deleted)
```

**Interprétation :** Le descripteur 1 correspond à stdout. Le lien permet encore d’accéder au fichier supprimé. Le descripteur 2 correspond à stderr.

## 7. Mesurer le fichier invisible

```bash
du -hL /proc/204032/fd/1
```

**Résultat observé — extrait ou synthèse**

```text
346G  /proc/204032/fd/1
```

**Interprétation :** -L suit le lien symbolique. La taille retrouvée explique l’écart entre df et du. Le fichier visible OSBSERVER1.out était distinct et vide (0 octet).

## 8. Examiner les dernières lignes

```bash
tail -n 30 /proc/204032/fd/1
```

**Résultat observé — extrait ou synthèse**

```text
Sortie détaillée non retrouvée.
```

**Interprétation :** Cette commande lit les 30 dernières lignes du journal. Le contenu exact ne doit pas être inventé.

## 9. Action corrective réalisée

```bash
> /proc/204032/fd/1
```

**Résultat observé — extrait ou synthèse**

```text
Aucune sortie terminal documentée.
```

**Interprétation :** La redirection tronque le fichier ouvert. Action destructive : validation, préservation des logs et vérification du PID, du descripteur et de l’inode indispensables. Préférer une réouverture des logs supportée par le service lorsque possible.

## 10. Vérifier la libération du stockage

```bash
df -h /opt
```

**Résultat observé — extrait ou synthèse**

```text
Utilisé : 29G | Disponible : 348G | Utilisation : 8 %
```

**Interprétation :** La partition est passée de 100 % à 8 %, sans redémarrage WebLogic. Le log ouvert était ensuite d’environ 5,5 Mo selon le compte rendu. La validation fonctionnelle et la cause de croissance restent à confirmer.

## 11. Prévenir la récidive

Contrôler la rotation de stdout, la rétention et la croissance des logs. Ne pas supprimer un journal actif manuellement. Vérifier les services et les nouveaux logs après intervention.
