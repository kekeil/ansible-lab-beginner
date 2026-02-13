# Lab Ansible Debutant sur GCP (Bastion + 3 noeuds)

Ce guide est un mini-cours pratique pour apprendre Ansible a petite echelle.
Objectif: comprendre le cycle complet, du SSH au deploiement d'une app tres simple.

## 1) Ce que tu vas construire

- 4 VM Ubuntu:
  - 1 bastion (machine de controle Ansible)
  - 3 noeuds cibles (`node-1`, `node-2`, `node-3`)
- Un canal SSH propre: bastion -> noeuds
- Un deploiement Ansible d'un script "hello world + nom du serveur"
- Un depot GitHub comme source unique de verite

## 2) Architecture du lab

```text
Google Cloud Shell
   |
   | (creation VM + config)
   v
Bastion (Ansible + Git)
   |------ SSH ------> node-1
   |------ SSH ------> node-2
   |------ SSH ------> node-3

GitHub (repo du lab) --git clone--> Bastion
```

## 3) Prerequis

- Un projet Google Cloud actif
- **Compte de facturation lie au projet** (obligatoire pour Compute Engine, voir section 4.2)
- Cloud Shell accessible
- Une paire de cle SSH de ton compte GCP (generee automatiquement par gcloud au besoin)
- Budget minimum (4x e2-micro, souvent couvert par le niveau gratuit)

## 4) Etape infra GCP (simple)

Les commandes suivantes sont lancees depuis Cloud Shell.

### 4.1 Variables d'environnement

Ces variables evitent de repeter les memes valeurs dans chaque commande `gcloud`.

| Variable | Role | Valeur par defaut |
|----------|------|-------------------|
| `PROJECT_ID` | Projet GCP cible (toutes les ressources y seront creees) | Lu depuis `gcloud config` |
| `REGION` | Region geographique (ex: europe-west1) | europe-west1 |
| `ZONE` | Zone precise dans la region (ex: europe-west1-b) | europe-west1-b |

#### Cas 1 : Projet deja configure

Si tu as deja fait `gcloud config set project MON_PROJET` :

```bash
export PROJECT_ID="$(gcloud config get-value project)"
export REGION="europe-west1"
export ZONE="europe-west1-b"
echo "Projet actif : $PROJECT_ID"
```

Tu dois voir l'ID du projet s'afficher.

#### Cas 2 : Projet non configure (unset)

Si `gcloud config get-value project` affiche `(unset)` ou rien :

1. Lister les projets disponibles :
   ```bash
   gcloud projects list
   ```

2. Choisir un projet et le definir :
   ```bash
   gcloud config set project VOTRE_PROJECT_ID
   ```

3. Re-exporter les variables :
   ```bash
   export PROJECT_ID="$(gcloud config get-value project)"
   export REGION="europe-west1"
   export ZONE="europe-west1-b"
   echo $PROJECT_ID
   ```

#### Cas 3 : Creer un nouveau projet pour le lab

Si tu veux isoler le lab dans un projet dedie :

```bash
gcloud projects create mon-lab-ansible --name="Lab Ansible"
gcloud config set project mon-lab-ansible
# Lier la facturation (voir section 4.2 si erreur)
gcloud billing projects link mon-lab-ansible --billing-account=ACCOUNT_ID
gcloud services enable compute.googleapis.com
export PROJECT_ID="$(gcloud config get-value project)"
export REGION="europe-west1"
export ZONE="europe-west1-b"
```

#### Cas 4 : Changer de region/zone

Si tu es loin de l'Europe ou veux reduire la latence :

```bash
# Lister les regions disponibles
gcloud compute regions list

# Exemples : us-central1, asia-southeast1, europe-west1
export REGION="us-central1"
export ZONE="us-central1-a"
```

#### Verification

Avant de creer les VM, verifie que tout est coherent :

```bash
echo "PROJECT_ID=$PROJECT_ID"
echo "REGION=$REGION"
echo "ZONE=$ZONE"
gcloud config get-value project
```

Si `PROJECT_ID` est vide, les commandes `gcloud compute instances create` echoueront avec une erreur de projet.

### 4.2 Facturation et API Compute (obligatoire avant les VM)

**Pourquoi** : L'API Compute Engine et la creation de VM necessitent un compte de facturation lie au projet. Meme en mode gratuit, GCP exige une carte bancaire enregistree.

**Ordre recommande** : Faire cette etape avant 4.3 (creation des VM). Si tu obtiens une erreur lors de la creation des VM, reviens ici.

#### Cas 1 : Erreur "API not enabled"

Si tu obtiens "API [compute.googleapis.com] not enabled" :

```bash
gcloud services enable compute.googleapis.com
```

Attends 1 a 2 minutes. Si une erreur apparaît (ex. timeout), relance la commande.

#### Cas 2 : Erreur "Billing must be enabled"

Si tu obtiens "Billing account for project ... is not found" ou "Billing must be enabled" :

1. **Lister les comptes de facturation** :
   ```bash
   gcloud billing accounts list
   ```

2. **Verifier qu'un compte est actif** :

   | OPEN | Signification |
   |------|---------------|
   | `True` | Compte actif, utilisable |
   | `False` | Compte ferme/suspendu, inutilisable |

   Si tous les comptes sont `OPEN: False` :
   - Va sur [console.cloud.google.com/billing](https://console.cloud.google.com/billing)
   - Cree un nouveau compte de facturation et associe une carte bancaire
   - Relance `gcloud billing accounts list` et note l'`ACCOUNT_ID` du compte avec `OPEN: True`

3. **Lier le compte au projet** :
   ```bash
   gcloud billing projects link $PROJECT_ID --billing-account=ACCOUNT_ID
   ```
   Remplace `ACCOUNT_ID` par la valeur affichee (ex. `01A749-1257B2-7BB394`).

4. **Activer l'API Compute** :
   ```bash
   gcloud services enable compute.googleapis.com
   ```

#### Cas 3 : Deja configure

Si tu as deja lie la facturation et active l'API sur ce projet, passe directement a la verification ci-dessous.

#### Verification avant de continuer

Execute ces commandes et assure-toi que tout est OK :

```bash
# 1. Facturation liee au projet
gcloud billing projects describe $PROJECT_ID
# Doit afficher : billingAccountName: ... (pas vide)

# 2. API Compute activee
gcloud services list --enabled --filter="name:compute.googleapis.com"
# Doit au moins lister : compute.googleapis.com

# 3. Variables toujours definies
echo "PROJECT_ID=$PROJECT_ID"
echo "ZONE=$ZONE"
```

Si tout est OK, tu peux passer a la creation des VM.
Si une etape echoue, reviens a la section correspondante ci-dessus.

### 4.3 Creation des VM

```bash
gcloud compute instances create ansible-bastion \
  --zone="$ZONE" \
  --machine-type=e2-micro \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --tags=ansible-bastion

gcloud compute instances create node-1 node-2 node-3 \
  --zone="$ZONE" \
  --machine-type=e2-micro \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --tags=ansible-node
```

### 4.4 Firewall minimal

Option pedagogique simple:
- SSH vers bastion depuis Internet
- SSH vers noeuds depuis le reseau interne (ou depuis bastion tagge)

```bash
gcloud compute firewall-rules create allow-ssh-bastion \
  --allow=tcp:22 \
  --target-tags=ansible-bastion \
  --source-ranges=0.0.0.0/0

gcloud compute firewall-rules create allow-ssh-between-lab-nodes \
  --allow=tcp:22 \
  --target-tags=ansible-node \
  --source-tags=ansible-bastion
```

Note: en production, limite `source-ranges` a ton IP publique.

### 4.5 Recupere les IP

```bash
gcloud compute instances list --filter="name~'ansible-bastion|node-'" \
  --format="table(name,zone,networkInterfaces[0].networkIP,networkInterfaces[0].accessConfigs[0].natIP)"
```

Conserve:
- IP publique du bastion
- IP internes des noeuds

## 5) Connexion au bastion et installation outils

### 5.1 Connexion

```bash
gcloud compute ssh ansible-bastion --zone="$ZONE"
```

### 5.2 Installation sur bastion

```bash
sudo apt update
sudo apt install -y git python3 python3-pip ansible sshpass vim

ansible --version
python3 --version
ssh -V
```

## 6) Canal SSH bastion -> noeuds (point cle)

### 6.1 Generer une cle SSH sur le bastion

```bash
ssh-keygen -t ed25519 -C "ansible-lab" -f ~/.ssh/id_ed25519 -N ""
```

### 6.2 Copier la cle publique vers chaque noeud

**Cas qui se mord la queue** : `ssh-copy-id` exige deja une connexion aux noeuds. Or les noeuds n'ont pas encore la cle du bastion, donc `ssh-copy-id` echoue avec "Permission denied (publickey)". Il faut d'abord injecter la cle via Cloud Shell (qui, lui, peut se connecter aux noeuds via `gcloud`).

**Methode** : Depuis Cloud Shell (pas depuis le bastion), ajouter la cle du bastion sur chaque noeud.

1. **Sur le bastion** : recuperer la cle publique, puis quitter (`exit`) :
   ```bash
   cat ~/.ssh/id_ed25519.pub
   exit
   ```

2. **Depuis Cloud Shell** : remplacer `TA_CLE_PUBLIQUE` par la sortie de l'etape 1 (toute la ligne). Utilise les **NETWORK_IP** de ta section 4.5.
   ```bash
   export ZONE="europe-west1-b"
   # Exemple avec node-1: 10.132.0.3, node-2: 10.132.0.5, node-3: 10.132.0.4
   gcloud compute ssh node-1 --zone="$ZONE" --command="mkdir -p ~/.ssh && chmod 700 ~/.ssh && echo 'TA_CLE_PUBLIQUE' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
   gcloud compute ssh node-2 --zone="$ZONE" --command="mkdir -p ~/.ssh && chmod 700 ~/.ssh && echo 'TA_CLE_PUBLIQUE' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
   gcloud compute ssh node-3 --zone="$ZONE" --command="mkdir -p ~/.ssh && chmod 700 ~/.ssh && echo 'TA_CLE_PUBLIQUE' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
   ```

3. **Reconnecte-toi au bastion** : `gcloud compute ssh ansible-bastion --zone="$ZONE"`

### 6.3 Permissions (deja fait)

Les commandes ci-dessus creent `~/.ssh` et `authorized_keys` avec les bonnes permissions. Rien a faire de plus.

### 6.4 Test SSH direct

```bash
ssh "$USER@10.132.0.3" "hostname"
ssh "$USER@10.132.0.5" "hostname"
ssh "$USER@10.132.0.4" "hostname"
```

Si ces 3 commandes passent sans mot de passe, Ansible pourra fonctionner.

## 7) GitHub: structure du dossier du lab

Tu peux creer un repo dedie, par exemple `ansible-lab-beginner`.

Structure recommandee:

```text
ansible-lab-beginner/
  README.md
  ansible.cfg
  inventory/
    hosts.ini
  group_vars/
    all.yml
  playbooks/
    ping.yml
    deploy-hello.yml
    verify.yml
  templates/
    hello.sh.j2
```

Dans ce repo actuel, un exemple pret a l'emploi est deja fourni ici:
`ansible/labs/gcp-beginner/`.

## 8) Cloner le repo sur bastion

```bash
sudo mkdir -p /opt/ansible-lab
sudo chown -R "$USER:$USER" /opt/ansible-lab
cd /opt/ansible-lab
git clone <URL_GITHUB_DU_REPO> .
```

## 9) Executer le lab Ansible

Depuis le bastion:

```bash
cd /opt/ansible-lab/ansible/labs/gcp-beginner
```

### 9.1 Remplir l'inventaire

Edite `inventory/hosts.ini` et remplace les IP d'exemple par tes IP internes GCP.

### 9.2 Verifier la connectivite Ansible

```bash
ansible all -i inventory/hosts.ini -m ping
```

### 9.3 Premier playbook de test

```bash
ansible-playbook -i inventory/hosts.ini playbooks/ping.yml -v
```

### 9.4 Deploiement de l'app simplifiee

```bash
ansible-playbook -i inventory/hosts.ini playbooks/deploy-hello.yml
```

### 9.5 Verification

```bash
ansible-playbook -i inventory/hosts.ini playbooks/verify.yml
```

## 10) Idempotence (concept central Ansible)

Relance 2 fois:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/deploy-hello.yml
ansible-playbook -i inventory/hosts.ini playbooks/deploy-hello.yml
```

Tu dois observer:
- premier run: `changed` > 0
- second run: `changed` proche de 0

## 11) Workflow quotidien ultra simple

1. Modifier les fichiers en local
2. `git add . && git commit -m "..." && git push`
3. Sur bastion: `git pull`
4. Relancer les playbooks

## 12) Depannage rapide

- `UNREACHABLE`:
  - verifier IP dans `inventory/hosts.ini`
  - verifier cle privee sur bastion
  - tester `ssh user@node-ip "hostname"`
- `Permission denied (publickey)`:
  - recopie la cle avec `ssh-copy-id`
  - verifie `chmod 700 ~/.ssh` et `chmod 600 ~/.ssh/authorized_keys`
- Python manquant sur cible:
  - `sudo apt install -y python3` sur le noeud

## 13) Prochaine etape apres ce lab

- Passer au deploiement Docker avec Ansible (niveau 2)
- Structurer en roles (`roles/common`, `roles/app`)
- Ajouter CI lint Ansible dans GitHub Actions

## 14) Nettoyage - Supprimer les VM

**Important** : Les VM consomment des ressources et peuvent generer des couts. Pense a les supprimer quand tu as termine le lab.

Depuis Cloud Shell (ou ton terminal avec `gcloud` configure) :

```bash
export ZONE="europe-west1-b"

gcloud compute instances delete ansible-bastion node-1 node-2 node-3 --zone="$ZONE"
```

Confirme avec `y` quand demande. Les VM et leurs disques boot sont supprimes definitivement.

Pour recreer le lab plus tard, reprendre depuis la section 4.3 (Creation des VM).
