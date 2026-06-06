# Lab 16 — Inspection HTTPS Android : Désactivation du SSL Pinning avec Objection + Proxy (Burp Suite)

**Étudiante :** Chaimaa Elgadaoui  
**Cours :** Sécurité des applications mobiles  
**Date :** 06 juin 2026  

---

> ⚠️ **Avertissement éthique** : Ces techniques sont utilisées uniquement dans un cadre légal (formation, audit autorisé, tests sur appareils personnels). Le contournement du SSL pinning sert à analyser la sécurité réseau d'une application, pas à compromettre des systèmes en production.

---

## Objectifs

- Installer et utiliser Objection (surcouche Frida) pour désactiver le SSL pinning d'une app Android.
- Mettre en place un proxy (Burp Suite) et installer le certificat CA sur l'appareil.
- Lancer l'app avec Objection et exécuter `android sslpinning disable` efficacement.
- Valider la capture du trafic HTTPS en clair dans Burp Suite.

---

## Partie 1 — Environnement & Outils

J'ai vérifié que tous les outils nécessaires étaient installés et fonctionnels sur mon PC Windows avant de démarrer le lab.

| Outil        | Version détectée  | Statut |
|--------------|-------------------|--------|
| Python       | 3.11.9            | ✅     |
| pip          | 24.0              | ✅     |
| ADB          | 1.0.41 (v37.0.0)  | ✅     |
| frida (CLI)  | 17.9.11           | ✅     |
| objection    | 1.12.5            | ✅     |

- **OS PC :** Windows 10.0.26200
- **Appareil :** Émulateur Android (`emulator-5554`) — statut : `device` 
- **Architecture CPU :** `x86_64`

Commandes exécutées :
```
python --version
pip --version
adb version
frida --version
pip install --upgrade objection
objection version
```

> **Note :** Objection a été mis à jour de la version 1.12.4 vers 1.12.5. La commande correcte pour afficher la version est `objection version` (et non `objection --version`).

> 📸 Screenshot : images/SS-01.png

---

## Partie 2 — Appareil & frida-server

J'ai réutilisé le même émulateur Android x86_64 du Lab 15. Le frida-server était déjà actif sur le port 27042.

```
adb devices
frida-ps -Uai
```

| Action                  | Résultat                          | Statut |
|-------------------------|-----------------------------------|--------|
| adb devices             | emulator-5554 — device            | ✅     |
| frida-ps -Uai           | Liste des apps visible            | ✅     |
| frida-server            | Actif sur port 27042              | ✅     |

- **App cible :** `jakhar.aseem.diva` (Diva) — PID 6989 ✅

> 📸 Screenshot : images/SS-02.png

---

## Partie 3 — Configuration Proxy & Installation du Certificat CA

J'ai configuré Burp Suite comme proxy TLS et installé son certificat CA sur l'émulateur.

**Étapes réalisées :**

**1) Lancement de Burp Suite :**
- Proxy listener : `127.0.0.1:8080` ✅

**2) Redirection ADB :**
```
adb reverse tcp:8080 tcp:8080
```

**3) Configuration du proxy Wi-Fi sur l'émulateur :**
- Paramètres → Wi-Fi → AndroidWifi → Modifier
- Proxy : Manual, hostname `127.0.0.1`, port `8080`

**4) Téléchargement du certificat CA Burp :**
- Navigateur → `http://127.0.0.1:8080` → CA Certificate → `cacert.der`

**5) Installation du certificat CA :**

Le fichier `.der` n'étant pas reconnu directement par le gestionnaire de fichiers Android, j'ai renommé le certificat en `.crt` via ADB :
```
adb shell "cp /sdcard/Download/cacert.der /sdcard/Download/cacert.crt"
```

Puis installé manuellement via :
**Paramètres → Sécurité → Chiffrement et informations d'identification → Installer un certificat → Certificat CA → cacert.crt**

Confirmation reçue : **"CA certificate installed"** ✅

| Action                          | Résultat                            | Statut |
|---------------------------------|-------------------------------------|--------|
| Burp Suite listener             | 127.0.0.1:8080 actif               | ✅     |
| adb reverse tcp:8080            | Trafic redirigé vers Burp           | ✅     |
| Proxy Wi-Fi émulateur           | 127.0.0.1:8080 configuré           | ✅     |
| Téléchargement cacert.der       | 986 bytes depuis Burp               | ✅     |
| Renommage en cacert.crt         | Reconnu par Android                 | ✅     |
| Installation CA utilisateur     | "CA certificate installed"          | ✅     |

> 📸 Screenshot : images/SS-03.png

---

## Partie 4 — Lancement d'Objection et Désactivation du SSL Pinning

J'ai attaché Objection à l'application DIVA déjà ouverte sur l'émulateur, puis désactivé le SSL pinning.

**Commande d'attachement :**
```
objection -g jakhar.aseem.diva explore
```

**Commande dans la console Objection :**
```
android sslpinning disable
```

**Logs obtenus :**
```
(agent) Custom TrustManager ready, overriding SSLContext.init()
(agent) Found com.android.org.conscrypt.TrustManagerImpl, overriding TrustManagerImpl.verifyChain()
(agent) Found com.android.org.conscrypt.TrustManagerImpl, overriding TrustManagerImpl.checkTrustedRecursive()
(agent) Registering job 594229. Name: android-sslpinning-disable
```

| Hook activé                              | Résultat   | Statut |
|------------------------------------------|------------|--------|
| SSLContext.init — Custom TrustManager    | Overridé   | ✅     |
| TrustManagerImpl.verifyChain()           | Overridé   | ✅     |
| TrustManagerImpl.checkTrustedRecursive() | Overridé   | ✅     |
| Job android-sslpinning-disable           | Enregistré | ✅     |

> 📸 Screenshot : images/SS-04.png

---

## Partie 5 — Validation du trafic HTTPS intercepté

Après installation du certificat CA et activation du bypass Objection, j'ai navigué vers `https://example.com` depuis **Chrome sur l'émulateur**. Burp Suite a intercepté et déchiffré le trafic HTTPS avec succès.

**Proxy :** Burp Suite Community v2026.4.3 — `127.0.0.1:8080`

| Requête interceptée       | Méthode | Status | TLS | IP              | Title          |
|---------------------------|---------|--------|-----|-----------------|----------------|
| https://example.com       | GET     | 200    | ✅  | 172.66.147.243  | Example Domain |
| http://play.googleapis.com| GET     | 204    | —   | 216.239.36.223  | —              |
| http://connectivitycheck  | GET     | 204    | —   | 142.251.140.227 | —              |

> **Observation clé :** La requête `https://example.com` apparaît dans Burp HTTP History avec la colonne **TLS cochée (✅)**, le status **200**, et le titre **"Example Domain"** visible en clair. Cela confirme que le trafic HTTPS est bien intercepté et déchiffré par Burp Suite grâce à la combinaison :
> - Certificat CA Burp installé comme CA utilisateur sur l'émulateur
> - Bypass SSL pinning activé via Objection (`android sslpinning disable`)
> - Proxy Wi-Fi configuré sur `127.0.0.1:8080` avec `adb reverse`

> 📸 Screenshot : images/SS-05.png

---

