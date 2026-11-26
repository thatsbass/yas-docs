# Introduction
Le Health Check serveur est un ensemble de contrôles préventifs visant à garantir la disponibilité, la stabilité et les performances d’un serveur Linux. Ce guide est orienté Red Hat Enterprise Linux (RHEL), Support Applicatif N1/N2, et couvre les serveurs SQL Server, Talend ESB, WSO2, Grafana et Jenkins. Il exclut les tests métier et la supervision F5/SolarWinds.
## Objectifs
Détecter les saturations, contrôler les services, anticiper les incidents, vérifier les sauvegardes et documenter les interventions. Les seuils sont indicatifs, non des procédures officielles Yas.
# Architecture et précautions
| Couche | Contrôles |
|---|---|
| Système | OS, kernel, uptime, horloge |
| Ressources | CPU, RAM, swap |
| Stockage | Partitions, inodes, I/O |
| Réseau | Interfaces, routage, DNS, ports |
| Exploitation | Processus, services, logs, backups |
| Sécurité | Accès, SELinux, TLS |
## Règles de production
- Commencer en lecture seule et identifier l’environnement.
- Ne jamais redémarrer, supprimer ou modifier sans validation.
- Vérifier les politiques de rétention, les dépendances et les preuves avant intervention.
- Ne jamais supprimer les fichiers actifs d’une base de données.
- Comparer les mesures avant et après, documenter et escalader selon l’impact.
# Identification du serveur
```bash
ssh utilisateur@adresse_ip
whoami
id
hostname
hostnamectl
cat /etc/os-release
cat /etc/redhat-release
uname -r
uname -a
date
timedatectl
uptime
who -b
last -x reboot | head
```
`uptime` indique le temps de fonctionnement, les sessions et les charges moyennes à 1, 5 et 15 minutes. Un redémarrage inattendu est à investiguer. Une horloge incorrecte affecte les certificats, transactions et corrélations de logs.
# CPU et Load Average
```bash
lscpu
nproc
top
ps -eo pid,ppid,user,comm,%cpu,%mem --sort=-%cpu | head -n 15
mpstat -P ALL 1 5
sar -u 1 5
sar -u
```
| Champ top | Signification |
|---|---|
| us | CPU utilisateur |
| sy | CPU noyau |
| ni | Priorité ajustée |
| id | CPU inactif |
| wa | Attente I/O |
| hi / si | Interruptions matérielles / logicielles |
| st | Temps retenu par hyperviseur |
| CPU soutenu | Statut |
| <70% | Normal |
| 70–85% | Warning |
| 85–95% | Élevé |
| >95% | Critique |
Le load average compte les tâches exécutables et en attente non interruptible. Comparer au nombre de CPU logiques, sans assimiler ce rapport à un pourcentage CPU. Un pic isolé ne suffit pas. `sysstat` est nécessaire pour sar/mpstat ; historique si collecte activée.
## Cas Talend : CPU à 95 %
```bash
uptime
nproc
top
ps -eo pid,user,comm,%cpu --sort=-%cpu | head
top -H -p 2456
```
Identifier le job et ses threads, consulter les logs, escalader. Ne pas tuer Java directement.
# Mémoire RAM
```bash
free -h
ps aux --sort=-%mem | head -n 15
ps -eo pid,user,comm,rss,%mem --sort=-rss | head -n 15
cat /proc/meminfo
sar -r
journalctl -k --since '24 hours ago' | grep -Ei 'out of memory|oom-kill|killed process'
dmesg -T | grep -Ei 'oom|out of memory|killed process'
```
| Champ | Signification |
|---|---|
| total | Mémoire physique |
| used | Mémoire utilisée selon la version de free |
| free | Mémoire inutilisée |
| shared | Mémoire partagée |
| buff/cache | Buffers et caches |
| available | Estimation de mémoire disponible sans swap important |
Prioriser `available` plutôt que `free`. RSS est généralement en Kio. Seuils RAM disponible : >30 % normal, 15–30 % surveillance, 5–15 % élevé, <5 % critique, à corréler avec la pression mémoire.
## Cas WSO2
16 Go de RAM, 250 Mo disponibles, swap 3,8/4 Go : vérifier free, vmstat, processus et événements OOM. Causes possibles : fuite Java, trafic ou JVM mal dimensionnée. Escalader avant modification.
# Swap
```bash
free -h
swapon --show
vmstat 1 5
sysctl vm.swappiness
```
| Champ vmstat | Description |
|---|---|
| r | Tâches exécutables |
| b | Tâches bloquées |
| swpd | Swap utilisée |
| si / so | Entrées / sorties swap |
Une swap occupée sans échanges soutenus n’est pas nécessairement un incident. Ne pas lancer swapoff ni modifier swappiness sans validation.
# Stockage et partitions
```bash
df -h
df -hT
lsblk
lsblk -f
findmnt
findmnt -T /var/opt/mssql
cat /etc/fstab
du -xhd1 /var | sort -hr
du -xhd1 /var/opt/mssql | sort -hr
find /var -xdev -type f -size +1G -exec ls -lh {} \;
find /var -xdev -type f -printf '%s\t%p\n' | sort -nr | head -n 20 | numfmt --field=1 --to=iec
find /opt -type f -name 'mesure.sh'
lsof +L1
```
| Utilisation | Statut |
|---|---|
| <70% | Normal |
| 70–79% | Warning |
| 80–89% | Élevé |
| 90–94% | Critique |
| ≥95% | Urgent |
`df` mesure le filesystem, `du` les fichiers accessibles, `lsblk` les périphériques. `du -xhd1` reste sur un filesystem, affiche en unités lisibles et limite la profondeur. Les scans globaux peuvent être coûteux. Un fichier supprimé mais ouvert reste occupé jusqu’à fermeture du descripteur. Vérifier lsof avant tout redémarrage. Les blocs réservés, métadonnées et montages masqués expliquent aussi des écarts df/du.
# Inodes
```bash
df -i
du --inodes -x -d1 /var | sort -nr
find /var/log -xdev -type f | wc -l
```
Un inode stocke les métadonnées d’un fichier. Un épuisement empêche la création de fichiers même si des Go restent disponibles. Rechercher caches, sessions, petits logs et fichiers temporaires.
# Performances disque et LVM
```bash
iostat -xz 1 5
pidstat -d 1 5
iotop -o
journalctl -k --since today | grep -Ei 'I/O error|buffer I/O|filesystem error|XFS|EXT4'
pvs
vgs
lvs
lvs -a -o lv_name,vg_name,lv_size,data_percent,metadata_percent
smartctl -H /dev/sda
smartctl -a /dev/sda
```
| I/O | Signification |
|---|---|
| r/s, w/s | Opérations par seconde |
| rkB/s, wkB/s | Débits |
| await | Latence moyenne |
| aqu-sz | File d’attente moyenne |
| %util | Temps de traitement I/O |
Latence et file croissantes peuvent indiquer saturation. %util seul est insuffisant pour SSD modernes. SMART dépend du matériel et peut être inaccessible en VM/SAN. Vérifier les pools LVM thin et leurs métadonnées. Ne pas lancer fsck/xfs_repair sur un filesystem monté en écriture.
# Réseau et connectivité
```bash
ip addr
ip -4 addr
ip -br addr
ip link show
ip route
ip route get 192.168.1.100
ping -c 4 192.168.1.100
cat /etc/resolv.conf
getent hosts serveur.example.com
nslookup serveur.example.com
dig serveur.example.com
ss -tulnp
ss -lntp '( sport = :443 )'
nc -vz -w 5 192.168.1.100 443
curl -I --connect-timeout 5 --max-time 15 https://serveur.example.com
curl -sS -o /dev/null --connect-timeout 5 --max-time 15 -w 'HTTP: %{http_code}\nDNS: %{time_namelookup}s\nTCP: %{time_connect}s\nTLS: %{time_appconnect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n' https://serveur.example.com
ss -s
ss -tan state established
ss -tanH | awk '{print $1}' | sort | uniq -c | sort -nr
ip -s link
ethtool eth0
ethtool -S eth0
sar -n DEV 1 5
systemctl status firewalld
firewall-cmd --get-active-zones
firewall-cmd --list-all-zones
nft list ruleset
```
| Résultat | Interprétation |
|---|---|
| Connected | TCP accessible, pas forcément application saine |
| Refused | Connexion rejetée |
| Timed out | Absence de réponse dans le délai |
| No route to host | Routage ou rejet réseau |
| LISTEN | Port en écoute |
| ESTAB | Connexion établie |
| TIME-WAIT | Connexion récemment fermée |
| CLOSE-WAIT | Fermeture distante reçue, fermeture locale attendue |
| SYN-SENT | Tentative de connexion |
ICMP peut être filtré. Vérifier erreurs et drops sur interfaces. Ne jamais désactiver le firewall pour contourner un incident. dig/nslookup nécessitent généralement bind-utils.
# Processus et descripteurs
```bash
ps aux
pgrep -af java
pgrep -af sqlservr
ps -p 2456 -o pid,ppid,user,stat,etime,%cpu,%mem,cmd
pstree -p
ps -eo pid,ppid,stat,comm | awk '$3 ~ /^Z/'
ps -eo pid,ppid,stat,wchan:32,comm | awk '$3 ~ /^D/'
cat /proc/2456/limits
ls -1 /proc/2456/fd | wc -l
cat /proc/sys/fs/file-nr
```
| État | Description |
|---|---|
| R | Exécutable |
| S | Attente interruptible |
| D | Attente non interruptible |
| Z | Zombie |
| T | Suspendu |
Un zombie a terminé mais son parent n’a pas récupéré son statut. Un état D persistant peut indiquer un problème I/O ou NFS. Trop de descripteurs entraîne « Too many open files ».
# Services systemd
```bash
systemctl list-units --type=service --state=running
systemctl --failed
systemctl status nginx --no-pager
systemctl is-active nginx
systemctl is-enabled nginx
systemctl show nginx -p MainPID -p NRestarts -p ExecMainStatus -p ActiveEnterTimestamp
systemctl list-dependencies nginx
journalctl -u nginx -n 100 --no-pager
journalctl -u nginx --since '1 hour ago'
```
| État | Signification |
|---|---|
| active (running) | En cours |
| active (exited) | Commande terminée, unité active |
| inactive | Inactif, parfois attendu |
| failed | Échec |
| activating/deactivating | Transition |
Services possibles : sshd, nginx, httpd, mssql-server, grafana-server, jenkins, docker, chronyd, rsyslog, crond. WSO2 et Talend dépendent de l’installation. Diagnostic : statut → logs → ressources et ports → cause → validation → correction → contrôle.
# Logs système et applicatifs
| Chemin | Contenu |
|---|---|
| /var/log/messages | Système |
| /var/log/secure | Authentification |
| /var/log/cron | Tâches planifiées |
| /var/log/audit/audit.log | Audit |
| /var/log/dnf.log | Packages |
| /var/log/nginx/ et /var/log/httpd/ | Serveurs web |
| /var/log/journal/ | Journal persistant |
| /var/opt/mssql/log/ | SQL Server |
```bash
head -n 20 /var/log/messages
tail -n 100 /var/log/messages
tail -f /var/log/messages
tail -F /var/log/messages
grep -inE 'error|fatal|failed|exception' /var/log/messages
grep -inE -C 5 'error|fatal|exception' application.log
zgrep -inE 'error|exception|failed' application.log.gz
zless application.log.gz
journalctl -p err -b
journalctl -p err --since '24 hours ago'
journalctl --since '2026-11-26 08:00:00' --until '2026-11-26 09:00:00'
journalctl -k -p warning --since today
du -sh /var/log
du -xhd1 /var/log | sort -hr
journalctl --disk-usage
```
DEBUG : détails ; INFO : fonctionnement ; WARNING : anomalie ; ERROR : échec ; CRITICAL/FATAL : grave. Les chemins varient selon configuration.
| Erreur | Piste |
|---|---|
| No space left on device | Disque/inodes |
| Permission denied | ACL/SELinux/permissions |
| Connection refused/timed out | Réseau/service |
| Out of memory | RAM |
| Too many open files | Descripteurs |
| Read-only file system | Montage/stockage |
| Address already in use | Conflit port |
| Segmentation fault | Crash |
| I/O error | Stockage |
# Rotation et compression
```bash
cat /etc/logrotate.conf
ls -l /etc/logrotate.d/
logrotate -d /etc/logrotate.conf
```
Simulation sans modification. Exemple de configuration à adapter à la réouverture des logs du service :
```text
/opt/application/logs/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    dateext
}
```
Rotation quotidienne, 30 rotations, compression différée, fichiers manquants ignorés, fichiers vides non tournés, suffixe daté.
## Compression et archives (après validation)
```bash
gzip application-20261101.log
gzip -k application-20261101.log
gunzip application-20261101.log.gz
gzip -t application-20261101.log.gz
tar -czf logs-novembre.tar.gz logs-novembre/
tar -tzf logs-novembre.tar.gz
zip -r logs-novembre.zip logs-novembre/
unzip -t logs-novembre.zip
```
Prévoir l’espace supplémentaire ; préférer un volume approuvé si le disque est presque plein.
## Purge journald (action corrective)
```bash
journalctl --disk-usage
# Après validation de la rétention :
journalctl --rotate --vacuum-size=1G
# Alternative selon politique :
journalctl --rotate --vacuum-time=30d
journalctl --disk-usage
df -h
```
La rotation archive les journaux actifs, puis vacuum supprime les archives anciennes selon la limite. Ce n’est pas une purge de tous les logs /var/log.
# Sauvegardes et rétention
```bash
ls -lh /backup
du -sh /backup
ls -lht /backup | head -n 15
find /backup -type f -mtime +30 -print
find /backup -type f -newermt '2026-05-01' ! -newermt '2026-06-01'
find /backup -type f -name '*.bak' -printf '%s\n' | awk '{s+=$1} END {printf "%.2f GiB\n",s/1024/1024/1024}'
systemctl status crond
crontab -l
ls -l /etc/cron.d/
systemctl list-timers --all
tail -n 100 /var/log/cron
```
Vérifier présence, date, taille, exécution, intégrité et restauration. La date de modification n’est pas nécessairement la date de sauvegarde ; mtime est arrondi par jours. Exemple fictif de rétention : quotidien 30 jours, hebdomadaire 90 jours, mensuel 12 mois. Appliquer exclusivement la politique validée et les dépendances de restauration.
# Cas pratique : SQL Server
Situation : partition à 95 %, suppression ciblée des backups DIFF de mai 2026. Les fichiers MDF/NDF/LDF dans data ne doivent jamais être supprimés manuellement. Les répertoires backup et log peuvent varier.
```bash
df -h /var/opt/mssql
du -xhd1 /var/opt/mssql | sort -hr
find /var/opt/mssql/backup -type f -printf '%f\n' | sed -nE 's/.*_(FULL|DIFF|LOG)_([0-9]{6})[0-9]{2}.*/\1 \2/p' | sort | uniq -c
find /var/opt/mssql/backup -type f -name '*_DIFF_202605*.bak' -print
find /var/opt/mssql/backup -type f -name '*_DIFF_202605*.bak' | wc -l
find /var/opt/mssql/backup -type f -name '*_DIFF_202605*.bak' -printf '%s\n' | awk '{s+=$1} END {printf "%.2f GiB\n",s/1024/1024/1024}'
```
Après validation DBA et contrôle des dépendances :
```bash
find /var/opt/mssql/backup -type f -name '*_DIFF_202605*.bak' -ok rm -- {} \;
df -h /var/opt/mssql
find /var/opt/mssql/backup -type f -name '*_DIFF_202605*.bak' | wc -l
```
Résultat de référence : 95 % → 93 % → 78 %, baisse de 17 points.
# Cas pratique : Talend ESB
Situation : 90 % d’utilisation ; backups datés AAAAMMJJ de février à juin 2026. Répertoire illustratif à vérifier.
```bash
find /opt/talend/backup -type f -printf '%f\n' | grep -E '2026(02|03|04|05|06)[0-9]{2}'
find /opt/talend/backup -type f -printf '%f\n' | grep -oE '2026(02|03|04|05|06)[0-9]{2}' | cut -c1-6 | sort | uniq -c
find /opt/talend/backup -type f \( -name '*202602*' -o -name '*202603*' -o -name '*202604*' -o -name '*202605*' -o -name '*202606*' \) -print
```
Après validation de la sélection exacte et de la rétention :
```bash
find /opt/talend/backup -type f \( -name '*202602*' -o -name '*202603*' -o -name '*202604*' -o -name '*202605*' -o -name '*202606*' \) -ok rm -- {} \;
df -h /opt/talend
du -sh /opt/talend/backup
```
Résultat de référence : 90 % → 53 %, baisse de 37 points.
# Sécurité et permissions
```bash
who
w
last -a | head -n 20
lastb | head -n 20
journalctl -u sshd --since today | grep -Ei 'failed|invalid user'
grep -i 'failed password' /var/log/secure
ls -ld /opt/application
stat /opt/application/config.properties
namei -l /opt/application/config.properties
getfacl /opt/application/config.properties
getenforce
sestatus
ausearch -m AVC,USER_AVC -ts today
dnf updateinfo list --security
dnf check-update
```
SELinux enforcing applique les règles, permissive journalise, disabled désactive. Ne pas désactiver SELinux ni utiliser chmod 777 comme solution. Le code retour 100 de dnf check-update signifie mises à jour disponibles. Installation soumise à la gestion des changements.
# Certificats TLS
```bash
openssl x509 -in certificat.crt -noout -subject -issuer -dates
openssl s_client -connect serveur.example.com:443 -servername serveur.example.com </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
openssl x509 -in certificat.crt -noout -checkend 2592000
openssl s_client -connect serveur.example.com:443 -servername serveur.example.com -verify_hostname serveur.example.com -verify_return_error </dev/null
```
| Durée restante | Statut |
|---|---|
| >60 jours | Normal |
| 30–60 jours | Surveillance |
| 7–29 jours | Renouvellement prioritaire |
| <7 jours | Critique |
| Expiré | Incident potentiel |
Contrôler le certificat effectivement présenté, le nom et la chaîne intermédiaire. checkend retourne un statut non nul si expiration dans la période.
# Horloge et NTP
```bash
timedatectl status
systemctl status chronyd
chronyc tracking
chronyc sources -v
```
Contrôler sources NTP, décalage et synchronisation entre serveurs.
# Tâches planifiées
```bash
systemctl status crond
crontab -l
crontab -u utilisateur -l
cat /etc/crontab
ls -l /etc/cron.d/
journalctl -u crond --since '24 hours ago'
systemctl list-timers --all
```
Vérifier exécution et résultats des backups, purges, rotations, rapports et synchronisations. Accès aux crontabs d’autres utilisateurs uniquement avec autorisation.
# Docker et Podman
```bash
systemctl status docker
docker ps
docker ps -a
docker stats --no-stream
docker logs --tail 100 nom-conteneur
docker logs --since 1h nom-conteneur
docker inspect --format 'Status={{.State.Status}} OOM={{.State.OOMKilled}} Restarts={{.RestartCount}}' nom-conteneur
docker inspect --format '{{json .State.Health}}' nom-conteneur
docker system df
docker system df -v
docker volume ls
podman ps -a
podman stats --no-stream
podman system df
podman logs --tail 100 nom-conteneur
```
Podman rootless : visibilité selon utilisateur. Un état running ne garantit pas un healthcheck réussi. Ne jamais lancer docker system prune -a --volumes sans analyse et autorisation.
# Diagnostic Java / JVM
```bash
pgrep -af java
jps -l
ps -p 2456 -o pid,user,%cpu,%mem,rss,etime,cmd
jcmd 2456 VM.flags
jcmd 2456 GC.heap_info
jstat -gcutil 2456 1000 5
top -H -p 2456
# Après validation si nécessaire :
jcmd 2456 Thread.print
```
Heap saturée : OOM ; Full GC fréquent : lenteurs ; nombreux threads : mémoire ; deadlock : blocage ; descripteurs épuisés : connexions en échec. Outils JDK et même utilisateur souvent nécessaires. Les diagnostics peuvent avoir un impact.
# Contrôles avancés
```bash
cat /proc/pressure/cpu
cat /proc/pressure/memory
cat /proc/pressure/io
ulimit -a
sysctl kernel.threads-max
systemctl show nginx -p LimitNOFILE -p TasksCurrent -p TasksMax
coredumpctl list
coredumpctl info
findmnt -t nfs,nfs4
nfsstat -m
journalctl -k --since '24 hours ago' | grep -Ei 'hardware error|machine check|mce|edac'
ipcs -a
```
PSI mesure les retards liés à la contention. ulimit concerne la session actuelle. NFS bloqué peut entraîner des tâches D. Sur matériel physique contrôler RAID, température, ventilateurs et alimentations. Ne pas supprimer les IPC avec ipcrm sans validation. Core dumps peuvent contenir des secrets.
# Tendances et anticipation
Observer les métriques dans le temps : exemple d’utilisation disque sur cinq jours : 62, 67, 73, 81, 87 %. Estimer le délai avant seuil : (capacité × seuil / 100 − espace utilisé) / croissance quotidienne, si croissance constante positive. Corréler avec événements et cycles de charge.
# Script Bash de collecte
Ce script de base ne redémarre ni ne supprime rien. Tester hors production ; certains contrôles nécessitent des privilèges et des packages. Les sorties peuvent contenir des données sensibles.
```bash
#!/bin/bash
export LC_ALL=C
section() { printf '\n==== %s ====\n' "$1"; }
section 'Identification'
date; hostname; whoami; cat /etc/os-release; uname -r; uptime
section 'CPU'
nproc; cat /proc/loadavg; top -bn1 | head -n 15
section 'RAM / Swap'
free -h; vmstat 1 3; swapon --show
awk '/^MemTotal:/ {t=$2} /^MemAvailable:/ {a=$2} END {if(t>0) {p=100*a/t; printf "Available: %.2f%%\n",p; if(p<15) print "WARNING: RAM"}}' /proc/meminfo
section 'Disques / Inodes'
df -hT; df -i
df -Pl | awk 'NR>1 {u=$5; gsub(/%/,"",u); if(u>=90) print "CRITICAL",$6,u"%"; else if(u>=80) print "WARNING",$6,u"%"}'
df -Pil | awk 'NR>1 {u=$5; gsub(/%/,"",u); if(u~/^[0-9]+$/ && u>=80) print "WARNING INODES",$6,u"%"}'
section 'Processus'
ps -eo pid,user,comm,%cpu,%mem --sort=-%cpu | head -n 11
ps -eo pid,user,comm,%cpu,%mem --sort=-%mem | head -n 11
section 'Services / Réseau'
systemctl --failed --no-pager; ip -br addr; ip route; ss -s; ss -tulnp
section 'Logs'
journalctl -p err --since '24 hours ago' --no-pager -n 50
journalctl -k --since '24 hours ago' --no-pager | grep -Ei 'out of memory|oom-kill|killed process'
journalctl --disk-usage
section 'Horloge / Sessions / Timers'
timedatectl status; who; systemctl list-timers --all --no-pager
section 'I/O'
if command -v iostat >/dev/null; then iostat -xz 1 3; else echo 'iostat absent'; fi
section 'Conteneurs'
if command -v docker >/dev/null; then docker ps -a; docker stats --no-stream; fi
if command -v podman >/dev/null; then podman ps -a; podman stats --no-stream; fi
section 'Fin'; date
```
```bash
vi health-check.sh
chmod 750 health-check.sh
./health-check.sh
umask 077
./health-check.sh 2>&1 | tee "healthcheck-$(hostname)-$(date +%F_%H%M).txt"
```
Limites : pas de validation fonctionnelle, ni de restauration, ni de contrôle exhaustif des certificats et dépendances. Les échecs de commandes doivent être traités comme contrôles non conclusifs.
# Fréquence et checklist
| Contrôle | Fréquence indicative |
|---|---|
| Disponibilité, CPU, RAM, swap, disques, inodes | Quotidienne |
| Services, réseau, logs, backups | Quotidienne |
| I/O, rotation, certificats, tendances, mises à jour | Hebdomadaire ou selon anomalie |
| Audit sécurité | Selon politique / mensuelle |
| Tests restauration | Selon politique |
Checklist : identifier serveur, uptime, horloge, CPU, RAM, swap, partitions, inodes, montages, services, réseau, ports, erreurs, OOM, sauvegardes, tâches, preuves, actions validées, rapport. Une case cochée signifie contrôlé, non forcément OK.
# Classification et escalade
| Statut | Action |
|---|---|
| OK | Surveillance habituelle |
| WARNING | Surveillance renforcée |
| CRITICAL | Intervention prioritaire |
| DOWN | Procédure incident |
| UNKNOWN | Identifier le blocage |
| N/A | Non applicable |
Détecter → collecter preuves → identifier cause probable → évaluer impact → faire valider → corriger ou escalader N2/N3 → contrôler → documenter. Priorité selon impact métier, urgence et SLA.
| Symptôme | Commandes |
|---|---|
| Serveur lent | uptime, top, vmstat, iostat |
| CPU élevé | top, ps, mpstat |
| RAM saturée | free, ps, vmstat |
| Disque plein | df, du, find |
| Inodes | df -i, du --inodes |
| Espace non libéré | lsof +L1 |
| Service arrêté | systemctl status, journalctl -u |
| Port inaccessible | ss, nc, ip route |
| DNS | getent, dig |
| Permissions | ls, namei, getfacl, ausearch |
| Crash Java | journalctl, jcmd |
| TLS | openssl |
| Reboot inattendu | last -x, journalctl -b -1 |
| Conteneur arrêté | docker ps -a, inspect |
# Rapport Health Check
## Informations générales
Date, heure, serveur, environnement, OS, responsable et statut global.
| Indicateur | Résultat fictif | Statut |
|---|---|---|
| Disponibilité | Accessible | OK |
| Uptime | 125 jours | OK |
| CPU | 25 % | OK |
| Load | 0.85 / 0.90 / 1.02 | OK |
| RAM disponible | 45 % | OK |
| Swap | 5 % | OK |
| / | 45 % | OK |
| /var | 92 % | CRITICAL |
| Inodes | 15 % | OK |
| Services, réseau | Opérationnels | OK |
| Logs | Aucune erreur critique récente | OK |
| Backup, NTP | Vérifiés | OK |
## Anomalies et diagnostic
Partition /var à 92 %, accumulation de logs archivés. Analyse des tailles et de la rétention.
## Actions réalisées
Vérification des partitions, analyse des fichiers, identification des archives, validation du périmètre, nettoyage ciblé autorisé, mesure après intervention, contrôle des services.
| Indicateur | Avant | Après |
|---|---|---|
| /var | 92 % | 65 % |
| Services | Actifs | Actifs |
## Recommandations et conclusion
Contrôler logrotate, suivre croissance et respecter la rétention. Statut final OK si tous les contrôles attendus sont concluants. Exemple fictif, à adapter aux observations réelles.
# Cheatsheet et routine quotidienne
```bash
hostname; date; uptime
top -bn1 | head -n 15
free -h
df -hT; df -i
systemctl --failed
ip -br addr; ss -s
journalctl -p err --since '24 hours ago' -n 50
journalctl --disk-usage
timedatectl status
```
| Commande | Usage |
|---|---|
| nproc, mpstat, sar | CPU et historique |
| vmstat, free | Mémoire et swap |
| df, du, find, lsblk, findmnt | Stockage |
| lsof +L1, iostat | Fichiers ouverts, I/O |
| ps, systemctl, journalctl | Processus, services, logs |
| tail, grep, zgrep | Analyse des journaux |
| ip, ss, ping, nc, curl, dig | Réseau |
| crontab, timedatectl | Tâches, temps |
| getenforce, openssl | Sécurité |
| docker stats, gzip, tar | Conteneurs, archives |
# Références et conclusion
```bash
man top
man free
man df
man du
man find
man systemctl
man journalctl
man ss
man lsof
man iostat
man logrotate
man openssl
```
Observer → Analyser → Identifier → Faire valider → Corriger → Vérifier → Documenter. Adapter les chemins, services, seuils et règles aux environnements réels. Ce guide pédagogique n’est pas une procédure officielle validée par Yas.
