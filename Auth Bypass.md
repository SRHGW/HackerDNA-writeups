# Writeup — Auth Bypass (HackerDNA)

Plateforme : HackerDNA
Auteur : SRH
Lab : Auth Bypass
Lien : https://hackerdna.com/labs/auth-bypass
Catégorie : Sécurité Web & API / Injection SQL / Contournement d'authentification
Difficulté : Intermédiaire (50 XP)
Objectif : Contourner le formulaire de connexion et récupérer le flag

---

## 1. Contexte

Le lab met en scène une application web dont le formulaire de connexion est vulnérable à l'injection SQL. L'objectif est de manipuler la logique d'authentification pour se connecter sans identifiants valides et accéder au flag [citation:2][citation:4].

---

## 2. Détection de la vulnérabilité

On identifie le formulaire de connexion (champs username/password).

Test initial : on injecte un guillemet simple ' dans le champ username avec un mot de passe quelconque.

Observation : la page renvoie une erreur SQL ou un comportement anormal, confirmant que l'entrée est concaténée directement dans la requête sans paramétrage [citation:2].

---

## 3. Exploitation

### Payload utilisé

Dans le champ username :

' OR 1=1 --

Mot de passe : n'importe quelle valeur (ou vide)

### Principe

La requête d'authentification originale ressemble à :

SELECT * FROM users WHERE username = '<input>' AND password = '<input>'

Avec le payload, elle devient :

SELECT * FROM users WHERE username = '' OR 1=1 --' AND password = '...'

La condition 1=1 est toujours vraie, et -- commente la fin de la requête (la vérification du mot de passe est ignorée).

### Résultat

La connexion est acceptée et le flag est affiché sur la page protégée.

---

## 4. Cause racine

La vulnérabilité provient de la concaténation de chaînes pour construire la requête SQL :

$sql = "SELECT * FROM users WHERE username = '" . $_POST['username'] . "' AND password = '" . $_POST['password'] . "'";

L'entrée utilisateur n'est ni échappée ni paramétrée, permettant à l'attaquant de modifier la structure de la requête [citation:2].

---

## 5. Remédiation

Solution principale : requêtes paramétrées (prepared statements).

$stmt = $pdo->prepare("SELECT * FROM users WHERE username = ? AND password = ?");
$stmt->execute([$username, $password]);

Mesures complémentaires :
- Validation des entrées (whitelist)
- Comptes DB à privilèges minimaux
- Messages d'erreur génériques (ne pas exposer les erreurs SQL)
- WAF en défense en profondeur [citation:2]

---

## 6. Leçons retenues

- Un simple ' qui casse la syntaxe SQL est un signal d'alarme.
- Le payload ' OR 1=1 -- exploite la logique booléenne pour contourner l'authentification.
- Les requêtes paramétrées sont la seule défense fiable contre l'injection SQL.
- Les filtres par liste noire sont contournables ; le paramétrage est indispensable [citation:2].

---
