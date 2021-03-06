# ansible-proxmoxve

## But du projet

Faciliter le déploiement de cluster ProxmoxVE via Scaleway et Ansible

## Déployer les VMs avec Ansible

[https://docs.ansible.com/ansible/latest/modules/scaleway_image_facts_module.html#scaleway-image-facts-module]

### Prérequis

* Avoir un compte sur Scaleway
* Avoir une clé SSH, avoir la clé publique à la racine du projet, et l'appeler admin.pub
* Installer les package pip

```bash
pip install  jinja2 PyYAML paramiko cryptography packaging
```

* Installer Ansible depuis les sources (>= 2.8devel)
* Installer jq

### Token

Créer un token sur le site de Scaleway pour les accès distants et le stocker dans un fichier scaleway\_token

```bash
export SCW_API_KEY='aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa'
```

Sourcer le fichier

```bash
source scaleway_token
```

### Générer les machines

Utiliser le playbook create\_proxmox\_vms.yaml pour :

* récupérer l'ID d'organisation du compte Scaleway
* récupérer un ID d'image compatible debian Stretch
* ajouter si nécessaire la clé SSH de l'admin
* créer autant de machines que nécessaire

```bash
ansible-playbook create_proxmox_vms.yml
```

### Mise en place de l'inventaire dynamique (inventory.yml)

```YAML
plugin: scaleway
regions:
  - par1
tags:
  - proxmoxve
```

Vérifier le retour de la commande ansible-inventory

```JSON
ansible-inventory --list -i inventory.yml
{
    "_meta": {
        "hostvars": {
            "x.x.x.x": {
                "arch": "x86_64",
                "commercial_type": "START1-S",
                "hostname": "proxmox3",
[...]
    "proxmoxve": {
        "hosts": [
            "x.x.x.x",
            "y.y.y.y",
            "z.z.z.z"
        ]
    }

```

Récupérer les empreintes de nos nouveaux serveurs pour éviter une erreur lors de la première connexion

```bash
ansible-inventory --list -i inventory.yml | jq -r '.proxmoxve.hosts | .[]' | xargs ssh-keyscan >> ~/.ssh/known_hosts
```

Ajouter sa clé SSH dans l'agent, puis vérifier qu'on peut se connecter à tous les serveurs via Ansible

```bash
eval `ssh-agent`
ssh-add myprivate.key
```

```JSON
ansible proxmoxve -i inventory.yml -u root -m ping
x.x.x.x | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
y.y.y.y | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
z.z.z.z | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

## Préparation du serveur

```bash
ansible-playbook -i inventory.yml -u root proxmox_prerequisites.yml
```

A l'issue de l'installation, rebooter les serveurs (manuellement ou via ansible)

## Installation et configuration de tinc

```bash
ansible-playbook -i inventory.yml -u root tinc_installation.yml
```

## Etapes additionnelles

Se connecter manuellemement sur tous les serveurs et configurer un mot de passe pour l'utilisateur root (absolument nécessaire pour la création du cluster).

### Créer un cluster

* Se connecter à l'UI de Proxmox avec un des noeuds
* Dans Datacenter (à gauche), sélectionner Cluster puis cliquer sur "Create Cluster"
* Donner un nom au cluster, **mais surtout, donner comme adresse ring0 l'adresse VPN** (10.10.10.1 si on est sur le noeud 1)
* Une fois l'opération terminée, cliquer sur "Join Information", dans le même menu, et copier les informations en cliquant sur "Copy Information"

### Joindre le cluster

* Se connecter sur l'UI de la 2ème machine
* Dans Datacenter, sélectionner Cluster puis cliquer sur "Join Cluster"
* Dans le champ "Information", coller les données récupérée lors de l'étape précédente. Les informations de connexion devraient automatiquement être remplies, **SAUF le Corosync ring0, qu'il faut positionner à 10.10.10.2"**, et le mot de passe root du noeud 1. Cliquer sur "Join".
* Si l'opération réussie, un message doit s'afficher pour dire de recharger la page

Répéter l'opération pour les noeuds suivants (3 voire ++)

## Autre

Réinitialiser le fichier known\_hosts

```bash
sed -ri '/^([0-9]+\.){3}[0-9]+ /d' ~/.ssh/known_hosts
```
