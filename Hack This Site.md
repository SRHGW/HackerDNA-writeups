# Writeup — Hack This Site (HackerDNA)

Plateforme : HackerDNA
Lab : Hack This Site
Lien : https://hackerdna.com/labs/hack-this-site
Catégorie : Reverse Engineering & Binary Exploitation / Client-Side Security
Difficulté : Intermédiaire
Objectif : Contourner l'authentification côté client et récupérer le flag

---

## 1. Contexte

Le lab met en scène une application web dont les développeurs ont tenté de protéger
les secrets en obscurcissant le code JavaScript. L'objectif est de percer ces
défenses en analysant le code côté client, d'extraire des identifiants codés en dur,
de contourner l'authentification et de capturer le flag [citation:2][citation:6].

---

## 2. Analyse

Le site cible est accessible via l'IP fournie par le lab. En ouvrant les DevTools
du navigateur (F12), on peut inspecter le code source, les fichiers JavaScript
chargés et les requêtes réseau.

L'obscurcissement JavaScript (renommage de variables, encodage de chaînes,
injection de code mort) n'offre pas de sécurité réelle : le navigateur doit
exécuter la logique originale, donc un analyste peut toujours la reconstituer [citation:2].

---

## 3. Exploitation

### Étape 1 — Repérer le code JavaScript obscurci

Dans l'onglet Sources (ou Debugger), on identifie les fichiers JS contenant
la logique d'authentification. Un formateur de code (beautifier) permet de
rendre le code lisible.

### Étape 2 — Rechercher des identifiants ou tokens

On recherche dans le code des chaînes suspectes : mots de passe, tokens,
conditions de validation. Les DevTools permettent de faire une recherche
globale (Ctrl+Shift+F) dans tous les fichiers [citation:2].

### Étape 3 — Contourner l'authentification

Si la vérification est purement côté client (comparaison de mot de passe en JS,
token stocké en local storage), on peut :
- Modifier la condition dans la console pour la rendre toujours vraie
- Extraire le mot de passe/token codé en dur et l'utiliser
- Modifier la requête réseau dans l'onglet Network (Edit and Resend)

### Étape 4 — Capturer le flag

Une fois l'authentification contournée, la page protégée devient accessible
et affiche le flag.

---

## 4. Cause racine

La sécurité repose entièrement sur du code côté client (JavaScript), qui est
par nature transparent pour l'utilisateur. Tout contrôle d'accès, validation
d'identifiants ou token stocké côté client peut être inspecté, modifié ou
contourné [citation:2].

---

## 5. Remédiation

- Ne jamais implémenter de vérification d'authentification côté client seul.
- Valider les identifiants et les autorisations côté serveur.
- Ne pas stocker de secrets (mots de passe, tokens) dans le code JavaScript.
- Considérer l'obscurcissement comme une simple gêne, pas une sécurité.

---

## 6. Leçons retenues

- L'obscurcissement JavaScript ne protège rien : le code doit être exécuté par
  le navigateur, donc il est toujours réversible.
- Les DevTools sont l'outil principal pour l'analyse de sécurité côté client.
- Toute décision de sécurité prise dans le navigateur peut être contournée.

---
