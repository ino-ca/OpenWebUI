## 1. Installation de Python 3.11 et Node.js 22.10+

### 1.1 Installer Python 3.11 via Deadsnakes

Vous devez installer la version 3.11 de Python, car Open WebUI est testé et recommandé pour Python 3.11. Sur Ubuntu, exécutez les commandes suivantes :

```bash
sudo apt update
sudo apt install -y software-properties-common curl git
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt install -y python3.11 python3.11-venv python3.11-dev
```

- **`sudo add-apt-repository ppa:deadsnakes/ppa -y`** ajoute le PPA deadsnakes qui fournit des versions récentes de Python.
- **`python3.11`** sera utilisé pour Open WebUI sans remplacer le Python système.

Vérifiez l’installation avec :

```bash
python3.11 --version
```

### 1.2 Installer Node.js (version ≥ 22.10)

Open WebUI comprend un frontend nécessitant Node.js. Vous devez installer au minimum la version 22.10. Exécutez les commandes suivantes :

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

Vérifiez l’installation avec :

```bash
node -v
npm -v
```

---

## 2. Cloner le dépôt Open WebUI dans /opt

Pour une installation globale, vous devez cloner le dépôt dans **/opt**. Exécutez :

```bash
ssh-keygen -t ed25519 -C "eric@ericvaillancourt.ca"
cat ~/.ssh/id_ed25519.pub

eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

ssh -T git@github.com

sudo mkdir -p /opt

cd ~

sudo git clone git@github.com:ino-ca/OpenWebUI.git
sudo mv OpenWebUI /opt/
cd /opt
```

Ensuite, assurez-vous que les droits sont correctement définis afin que le service puisse y accéder (par exemple, en changeant le propriétaire vers un utilisateur dédié ou en ajustant les permissions) :

```bash
sudo chown -R root:root /opt/OpenWebUI
```

> **Remarque :** Vous pouvez créer un utilisateur système dédié (par exemple, `openwebui`) et changer le propriétaire en conséquence pour plus de sécurité.

Placez-vous dans le dossier cloné :

```bash
cd /opt/OpenWebUI
```

---

## 3. Configurer le fichier `.env`

### 3.1 Créer le fichier `.env`

Copiez le fichier d’exemple en exécutant :

```bash
sudo cp -RPp .env.example .env
```

- L’option **`-RPp`** permet de préserver les attributs et permissions.

### 3.2 Modifier les variables d’environnement

Éditez le fichier **.env** (avec `sudo nano .env` par exemple) et ajoutez ou modifiez les lignes suivantes :

```bash
OPENAI_API_BASE_URL='https://api.openai.com/v1'
OPENAI_API_KEY='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'
DATABASE_URL='postgresql://postgres:Machiavel69@192.168.2.9/talsom_webui'
WEBUI_NAME="Talsom AI Hub"
ENABLE_OAUTH_SIGNUP="true"
MICROSOFT_CLIENT_ID="635ea318-c712-4c2b-8029-71e277cb7180"
MICROSOFT_CLIENT_SECRET="mDF8Q~kencoekKq1W9E3Nzyjo~X2meVwZ5cizbx6"
MICROSOFT_CLIENT_TENANT_ID="ceaaca2d-67b2-45c0-a714-18c3225b6c1e"
MICROSOFT_OAUTH_SCOPE="openid email profile"
MICROSOFT_REDIRECT_URI="https://chat.talsom.tech/oauth/microsoft/callback"
```

> **Astuce :** Veillez à adapter ces valeurs selon vos propres identifiants et configurations.

---

## 4. Installer et construire le frontend

Dans le répertoire **/opt/open-webui**, vous devez installer les dépendances et construire le frontend :

```bash
sudo npm install
sudo npm run build
```

- **`npm install`** installe toutes les dépendances définies dans le fichier `package.json`.
- **`npm run build`** génère le build de production du frontend.

> **Conseil :** Si vous préférez éviter d’utiliser sudo avec npm, vous pouvez adapter les permissions sur /opt ou exécuter les commandes en tant qu’utilisateur dédié.

---

## 5. Préparer et configurer le backend

### 5.1 Créer un environnement virtuel Python 3.11

Placez-vous dans le dossier backend :

```bash
cd /opt/OpenWebUI/backend
```

Créez un environnement virtuel en utilisant Python 3.11 :

```bash
sudo python3.11 -m venv .venv
```

Puis activez-le :

```bash
source .venv/bin/activate
```

Votre invite devrait être préfixée par `(.venv)`.

### 5.2 Installer les dépendances Python du backend

Installez les dépendances nécessaires avec :

```bash
sudo chown -R eric:eric /opt/OpenWebUI/backend/.venv
pip install -r requirements.txt -U
```

Cette commande installe et met à jour les bibliothèques requises (FastAPI, Uvicorn, etc.).

---

## 6. Installer et configurer le certificat SSL/TLS

### 6.1 Créer le dossier `cert` et copier les certificats

Dans le répertoire **/opt/open-webui/backend**, créez le dossier **cert** :

```bash
sudo mkdir -p cert
sudo chmod 700 cert
```

Ensuite, copiez vos fichiers de certificat wildcard dans ce dossier. Par exemple, si vous disposez de :

- `wildcard.talsom.tech.key` (clé privée)
- `wildcard.talsom.tech.crt` (certificat)

Exécutez :

```bash
sudo cp /chemin/vers/wildcard.talsom.tech.key cert/
sudo cp /chemin/vers/wildcard.talsom.tech.crt cert/
```

Vérifiez et, si nécessaire, ajustez les permissions :

```bash
sudo chmod 600 cert/wildcard.talsom.tech.key
```

### 6.2 Modifier le script `start.sh` pour utiliser le port 443 en HTTPS

Ouvrez le fichier **/opt/open-webui/backend/start.sh** avec un éditeur (par exemple, `sudo nano start.sh`) et modifiez la fin du script pour utiliser le port 443 et charger le certificat SSL. Par exemple :

```bash
#!/usr/bin/env bash

SCRIPT_DIR=$( cd -- "$( dirname -- "${BASH_SOURCE[0]}" )" &> /dev/null && pwd )
cd "$SCRIPT_DIR" || exit

# (Contenu existant du script, incluant la génération de WEBUI_SECRET_KEY, etc.)

PORT="${PORT:-443}"
HOST="${HOST:-0.0.0.0}"

# Lancement d’Uvicorn en HTTPS avec les certificats
WEBUI_SECRET_KEY="$WEBUI_SECRET_KEY" exec uvicorn open_webui.main:app \
  --host "$HOST" \
  --port "$PORT" \
  --forwarded-allow-ips '*' \
  --ssl-keyfile "$SCRIPT_DIR/cert/wildcard.talsom.tech.key" \
  --ssl-certfile "$SCRIPT_DIR/cert/wildcard.talsom.tech.crt"
```

> **Important :** Le port 443 est un port privilégié. Vous devez donc disposer des droits nécessaires pour lier ce port. Vous pouvez, par exemple, attribuer la capacité `cap_net_bind_service` à Python (ou Uvicorn) pour éviter d’exécuter le service en root :
>
> ```bash
> sudo setcap 'cap_net_bind_service=+ep' $(which python3.11)
> ```

### 6.3 Rendre le script exécutable

Exécutez :

```bash
sudo chmod +x start.sh
```

---

## 7. Créer un service systemd pour Open WebUI

Pour que l’application démarre automatiquement, vous devez créer un service systemd.

### 7.1 Créer et éditer le fichier de service

Créez le fichier **/etc/systemd/system/openwebui.service** :

```bash
sudo nano /etc/systemd/system/openwebui.service
```

Ajoutez-y le contenu suivant :

```ini
[Unit]
Description=Open WebUI Service (Talsom AI Hub)
After=network.target

[Service]
Type=simple
# Pour plus de sécurité, vous pouvez créer et utiliser un utilisateur dédié (par ex. openwebui)
User=root
WorkingDirectory=/opt/OpenWebUI/backend
ExecStart=/bin/bash -c 'source /opt/OpenWebUI/backend/.venv/bin/activate && exec /opt/OpenWebUI/backend/start.sh'
Restart=always

[Install]
WantedBy=multi-user.target
```

> **Remarque :**  
> - Ici, le service est exécuté en tant que **root** pour faciliter l’accès au port 443. Pour une solution plus sécurisée, envisagez de créer un utilisateur système dédié et d’attribuer les capacités nécessaires.
> - Vérifiez que le chemin **WorkingDirectory** pointe vers **/opt/open-webui/backend**.

### 7.2 Activer et démarrer le service

Rechargez la configuration systemd, activez et démarrez le service avec :

```bash
sudo systemctl daemon-reload
sudo systemctl enable openwebui.service
sudo systemctl start openwebui.service
```

Vérifiez le statut du service :

```bash
sudo systemctl status openwebui.service
```

Le service doit apparaître comme **active (running)**. En cas de problème, consultez les journaux avec :

```bash
sudo journalctl -u openwebui.service -f
```

---

## 8. Accéder à votre Open WebUI

Si tout s’est bien passé, votre application est désormais accessible en HTTPS via le port 443. Vous devez pouvoir y accéder en ouvrant votre navigateur sur :

```
https://<adresse-du-serveur>
```

Assurez-vous que le DNS du domaine (par exemple, `chat.talsom.tech`) pointe vers l’IP de votre serveur.

---

## Références

- **Open WebUI** – [GitHub du projet](https://github.com/open-webui/open-webui)
- **NodeSource** – [Guide d’installation de Node.js 22.x](https://github.com/nodesource/distributions/blob/master/README.md)
- **Deadsnakes PPA** – [Python sur Ubuntu](https://launchpad.net/~deadsnakes/+archive/ubuntu/ppa)
- **Systemd** – [Documentation officielle](https://www.freedesktop.org/software/systemd/man/systemd.service.html)

---

En suivant ces étapes, vous installez Open WebUI dans **/opt/open-webui**, avec un environnement Python 3.11, Node.js ≥ 22.10 et un certificat wildcard pour servir l’application en HTTPS sur le port 443. Vous disposez ainsi d’une solution centralisée et non dépendante d’un répertoire utilisateur spécifique.