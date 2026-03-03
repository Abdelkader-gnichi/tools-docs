# Certificat Let’s Encrypt pour demo-usp.pivasoftware.com

Ce document explique pas à pas comment obtenir et installer un certificat Let’s Encrypt pour le sous-domaine `demo-usp.pivasoftware.com`.

---

## 1️⃣ Prérequis
- Accès à la console ou au terminal de votre serveur.
- Accès à l'interface DNS de `pivasoftware.com` (hébergé chez IONOS / 1&1).
- Certbot installé sur le serveur :
```bash
sudo apt update
sudo apt install certbot
```

---

## 2️⃣ Commande Certbot

Exécuter Certbot en mode manuel avec challenge DNS :
```bash
sudo certbot certonly --manual --preferred-challenges dns -d demo-usp.pivasoftware.com
```

- `certonly` : génère uniquement le certificat, sans configurer le serveur.
- `--manual` : validation manuelle.
- `--preferred-challenges dns` : utilisation d’un challenge DNS.
- `-d demo-usp.pivasoftware.com` : le sous-domaine à sécuriser.

---

## 3️⃣ Créer l’enregistrement DNS TXT

Certbot va générer une valeur unique à ajouter dans le DNS, par exemple :

- **Nom / Host** : `_acme-challenge.demo-usp`
- **Valeur / Content** : `oegStI4UviCRk-ZnSKonRWTGw0ECXqYS-fzf8PxRt0k`
- **TTL** : 300 (5 minutes)

> ⚠️ Important : ne pas ajouter `.pivasoftware.com` dans le nom si IONOS l’ajoute automatiquement.

### Instructions IONOS / 1&1 :
1. Connectez-vous à votre interface DNS.
2. Sélectionnez la zone `pivasoftware.com`.
3. Ajoutez un enregistrement **TXT** avec les informations ci-dessus.
4. Sauvegardez.

---

## 4️⃣ Vérifier la propagation

Avant de continuer avec Certbot, vérifiez que le TXT est visible publiquement :

```bash
dig +short TXT _acme-challenge.demo-usp.pivasoftware.com
```

- La sortie doit être exactement la valeur générée par Certbot :

```
"oegStI4UviCRk-ZnSKonRWTGw0ECXqYS-fzf8PxRt0k"
```

- Si vous voyez `NXDOMAIN`, attendez quelques minutes et testez de nouveau.

---

## 5️⃣ Valider le challenge avec Certbot

- Une fois que le TXT est visible, retournez dans le terminal Certbot et appuyez sur **Enter**.
- Certbot vérifiera le challenge et générera le certificat si tout est correct.

---

## 6️⃣ Emplacement des fichiers de certificat

Après génération, Certbot indique l’emplacement des fichiers :

```
/etc/letsencrypt/live/demo-usp.pivasoftware.com/fullchain.pem
/etc/letsencrypt/live/demo-usp.pivasoftware.com/privkey.pem
```

- `fullchain.pem` → certificat complet
- `privkey.pem` → clé privée

---

## 7️⃣ Déploiement sur le serveur (APISIX, Nginx, etc.)

- Configurez votre serveur pour utiliser les fichiers ci-dessus.
- Assurez-vous que le port **443** est exposé et que le service écoute en HTTPS.

---

## 8️⃣ Astuces et recommandations

- Pour plusieurs sous-domaines, utiliser un **wildcard** :  
```bash
sudo certbot certonly --manual --preferred-challenges dns -d "*.pivasoftware.com"
```
- Pour éviter la saisie manuelle à chaque renouvellement, utiliser un **plugin DNS** si disponible pour votre provider.

---

Document préparé pour sécuriser `demo-usp.pivasoftware.com` avec Let’s Encrypt.

