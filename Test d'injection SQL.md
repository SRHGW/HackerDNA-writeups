# Writeup — Test d'injection SQL (HackerDNA)

Plateforme : HackerDNA
Lab : Test d'injection SQL
Auteur : SRH
Lien : https://hackerdna.com/labs/sql-injection-test
Catégorie : Sécurité Web & API / Contournement d'authentification
Difficulté : Basique
Objectif : Contourner le formulaire de connexion et récupérer le flag

---

## 1. Contexte

Le lab fournit un portail de connexion réaliste dont la requête SQL sous-jacente est vulnérable. L'objectif est de manipuler la logique d'authentification pour se connecter sans connaître de mot de passe valide.

---

## 2. Détection de la vulnérabilité

Test initial : on saisit un guillemet simple ' dans le champ nom d'utilisateur, avec un mot de passe quelconque.
Observation : la page renvoie une erreur de syntaxe SQL ou un comportement anormal, ce qui indique que l'entrée utilisateur est concaténée directement dans la requête, sans paramétrage.

Confirmation : on teste une condition toujours vraie :
' OR '1'='1
Si la connexion réussit malgré un mot de passe invalide, la faille est confirmée : la clause WHERE de la requête d'authentification est manipulable.

---

## 3. Exploitation

Payload utilisé :
admin' or '1'='1
Champ : Nom d'utilisateur
Mot de passe : n'importe quelle valeur

Principe : la requête d'authentification originale ressemble à :
SELECT * FROM users WHERE username = '<input>' AND password = '<input>'

Avec le payload, elle devient :
SELECT * FROM users WHERE username = 'admin' or '1'='1' AND password = '...'

La condition '1'='1' est toujours vraie, ce qui contourne la vérification du mot de passe et retourne l'utilisateur admin.

Résultat : la connexion est acceptée et le flag est affiché sur le tableau de bord administrateur.

---

## 4. Cause racine

La vulnérabilité provient de la concaténation de chaînes pour construire la requête SQL :

$sql = "SELECT * FROM users WHERE username = '" . $_POST['username'] . "' AND password = '" . $_POST['password'] . "'";

L'entrée utilisateur n'est ni échappée ni paramétrée, permettant à l'attaquant de modifier la structure de la requête.

---

## 5. Remédiation

Solution principale : requêtes paramétrées. La seule défense fiable est l'utilisation de requêtes préparées (prepared statements), qui séparent le code SQL des données utilisateur :

$stmt = $pdo->prepare("SELECT * FROM users WHERE username = ? AND password = ?");
$stmt->execute([$username, $password]);

Avec cette approche, le payload admin' or '1'='1 est traité comme une chaîne littérale, pas comme du code SQL.

Mesures complémentaires :
- Validation des entrées : limiter les caractères autorisés (défense en profondeur)
- Comptes DB à privilèges minimaux : réduire l'impact en cas de compromission
- Messages d'erreur génériques : ne pas exposer la structure SQL au client
- WAF : bloquer les payloads connus (ne remplace pas le paramétrage)

---

## 6. Leçons retenues

- Un simple ' qui casse la syntaxe SQL est déjà un signal d'alarme.
- Le payload ' OR '1'='1 exploite la logique booléenne pour rendre la clause WHERE toujours vraie.
- Les requêtes paramétrées sont la seule protection fiable ; le filtrage par liste noire est contournable.
- Ce lab illustre pourquoi la validation des entrées doit compléter, jamais remplacer, le paramétrage des requêtes.

---
