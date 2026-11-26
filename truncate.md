## 1. Situation rencontrée
Lors du Health Check WSO2 / Grafana, un fichier `/var/log/kibana/kibana.log` de 11 Go a été identifié dans nos échanges. La sortie terminal originale et le résultat d'une éventuelle troncature de ce fichier ne sont pas disponibles ici. Les commandes et résultats ci-dessous constituent donc un exemple pédagogique, distinct de l'intervention WebLogic confirmée dans le chapitre précédent.

## 2. Identifier le fichier et vérifier son état
Vérifier la taille, l'inode et les processus qui utilisent le fichier avant toute décision.

```bash
ls -lh /var/log/kibana/kibana.log
stat /var/log/kibana/kibana.log
lsof /var/log/kibana/kibana.log
df -h /var/log/kibana
```

`ls -lh` affiche la taille lisible ; `stat` affiche notamment l'inode et les métadonnées ; `lsof` identifie les processus ayant ouvert le fichier ; `df` mesure le stockage du système de fichiers.

**Exemple de résultat (fictif)**

```text
Taille : 11G
Inode : 123456
```

## 3. Pourquoi vider plutôt que supprimer ?
Un processus peut continuer à écrire dans un fichier supprimé tant que son descripteur reste ouvert. La troncature conserve le nom et l'inode du fichier existant, mais ramène sa taille à zéro. Elle détruit son contenu ; elle ne ferme pas les descripteurs ni ne réinitialise nécessairement leur position d'écriture.

## 4. Tester la méthode sur un fichier de laboratoire
```bash
lab=$(mktemp -d)
printf 'INFO : exemple de journal\n' > "$lab/demo.log"
ls -li "$lab/demo.log"
truncate -s 0 "$lab/demo.log"
ls -li "$lab/demo.log"
```

**Exemple de résultat (fictif, simplifié)**

```text
Avant : inode 123456, taille 26 octets
Après : inode 123456, taille 0 octet
```

Le fichier existe toujours et son inode reste identique. `-s 0` signifie fixer la taille à zéro octet.

## 5. Trois méthodes équivalentes sur un fichier existant
Choisir une seule méthode. Les exemples utilisent le fichier de laboratoire créé précédemment.

```bash
truncate -s 0 "$lab/demo.log"
```

`truncate` fixe explicitement la taille du fichier.

```bash
> "$lab/demo.log"
```

`>` est une redirection du shell : il ouvre le fichier en écriture et le tronque, même sans commande produisant du texte.

```bash
cat /dev/null > "$lab/demo.log"
```

`/dev/null` ne fournit aucune donnée à la lecture ; la redirection vide le fichier. C'est la redirection, et non `cat`, qui provoque la troncature.

## 6. Intervention en production : précautions
Ne pas tronquer directement un fichier actif sans autorisation du responsable, vérification de la rétention, conservation des preuves nécessaires et confirmation du chemin et de l'inode. Préférer la rotation native du service lorsqu'elle est disponible. Ne jamais utiliser ces méthodes sur des fichiers de données de bases de données. Un processus qui écrit sans mode append peut reprendre à son ancien offset et créer un fichier creux ; surveiller la reprise des écritures. Ces commandes peuvent créer un fichier absent : confirmer son existence avant intervention.

## 7. Vérifier après intervention
```bash
ls -li /var/log/kibana/kibana.log
stat /var/log/kibana/kibana.log
df -h /var/log/kibana
lsof /var/log/kibana/kibana.log
tail -n 10 /var/log/kibana/kibana.log
```
Comparer l'inode, la taille, l'espace disponible et la reprise des écritures ; vérifier également le service concerné. Aucun résultat réel après troncature de kibana.log n'est affirmé ici.

## 8. Cas WebLogic : fichier déjà supprimé
Lors de l'incident WebLogic, le fichier n'était plus accessible par son nom. La troncature a été réalisée via son descripteur ouvert, `> /proc/204032/fd/1`, selon le compte rendu précédent. Le disque est passé de 100 % à 8 %. Ce PID historique ne doit jamais être réutilisé sans nouvelle investigation. Voir le [cas WebLogic détaillé](#pratique-weblogic).
