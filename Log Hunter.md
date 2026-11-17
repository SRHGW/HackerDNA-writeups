# Writeup — Log Hunter (HackerDNA)

Plateforme : HackerDNA
Auteur : SRH
Lab : Log Hunter
Lien : https://hackerdna.com/labs/log-hunter
Catégorie : Forensique / Analyse de logs web
Difficulté : Intermédiaire (50 XP)
Objectif : Analyser access.log pour trouver le fichier exfiltré par un attaquant et récupérer le flag

---

## 1. Contexte

Le lab fournit un fichier access.log d'un serveur web compromis.
La mission : identifier le fichier que les attaquants ont réussi à accéder
pendant leur scan, puis récupérer le flag caché à l'intérieur.

---

## 2. Téléchargement et inspection

Téléchargement du fichier :
curl -sL 'https://lab.hdna.me/81-log-hunter/access.log' -o access.log
wc -l access.log  # 177 lignes

Le log utilise le format Apache Combined. Chaque ligne contient :
IP client - - [timestamp] "requête HTTP" code_statut taille "referer" "user-agent"

---

## 3. Analyse

Filtrage des accès réussis (200) vers des ressources non statiques,
en cherchant des chemins sensibles :

grep ' 200 ' access.log | grep -vE '\.(png|jpg|gif|css|js|html|xml|txt|ico)' \
  | grep -E '(admin|login|php|upload|env|git|config|backup|shell|cgi|test|api)'

Résultat clé :

172.16.0.88 - - [22/Jul/2025:08:32:44 +0000] "GET /backup.txt HTTP/1.1" 200 42
  "https://attacker.com/tools" "SecScanner/1.0"

Interprétation :
- IP : 172.16.0.88 (interne, attaquant)
- Statut : 200 (accès réussi)
- Taille : 42 octets (correspond à un UUID + newline)
- Referer : https://attacker.com/tools (infrastructure attaquante)
- User-Agent : SecScanner/1.0 (scanner automatisé)

Les autres requêtes de cette IP (/admin/, /.env, /.git/, /config.php, /phpmyadmin/)
renvoient toutes 404. Seul backup.txt a été trouvé et lu avec succès.

---

## 4. Récupération du flag

Le fichier est accessible sur le même serveur :

curl -s 'https://lab.hdna.me/81-log-hunter/backup.txt'

Le contenu contient le flag (UUID).

---

## 5. Cause racine

Un fichier de sauvegarde (backup.txt) contenant des données sensibles a été
laissé dans la racine web (web root), accessible publiquement.
Un scanner automatisé a énuméré les chemins courants et est tombé dessus.

---

## 6. Remédiation

- Ne jamais placer de fichiers sensibles (backups, configs, dumps, .env)
  dans le répertoire accessible par le serveur web.
- Stocker ces fichiers en dehors de la racine web.
- Restreindre l'accès aux fichiers par authentification/autorisation.
- Surveiller les logs pour détecter les scans automatisés (User-Agent suspect,
  rafales de 404, referer externe).

---

## 7. Leçons retenues

- L'analyse de logs commence par les anomalies : un 200 sur un chemin
  inhabituel avec un User-Agent non standard est le premier indicateur.
- Les scanners automatisés laissent des signatures évidentes : progression
  séquentielle de chemins, timing régulier, referer pointant vers l'attaquant.
- Filtrer les 200 sur des chemins non publics en premier : ce sont les
  vraies trouvailles.
- Les codes de statut racontent l'histoire : 404 = échec, 200 = succès.

---
