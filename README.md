# TP Infrastructure Haute Disponibilité – Pacemaker & Corosync

## 🎯 Objectif

Dans ce TP, nous allons construire ensemble une infrastructure de haute disponibilité en mode *active/passive*. L'idée est simple : si un serveur tombe en panne, le service continue de fonctionner automatiquement sur un autre serveur, sans interruption pour les utilisateurs.

On va utiliser **Pacemaker** et **Corosync** pour gérer cette bascule automatique, et **Nginx** comme service web de démonstration.

---

## 🏗️ Architecture de notre lab

Voici comment sont organisées nos machines :

| Machine | Rôle | Adresse IP |
|---------|------|------------|
| admin | Machine cliente pour les tests | 192.168.36.139 |
| node1 | Premier nœud du cluster | 192.168.36.140 |
| node2 | Second nœud du cluster | 192.168.36.141 |
| VIP | IP flottante (virtuelle) | 192.168.36.150 |

Le principe : les deux nœuds communiquent entre eux et se partagent une IP virtuelle (VIP). C'est cette IP qui héberge le service, et elle "saute" d'un nœud à l'autre en cas de problème.

---

## 📋 Ce dont vous avez besoin

- Ubuntu Server 20.04 ou 22.04
- Accès root ou sudo sur les machines
- Les trois VM doivent pouvoir communiquer sur le même réseau
- Une connexion internet pour installer les paquets

---

## 🔍 Les outils qu'on va utiliser

**Corosync** s'occupe de la communication entre les nœuds. C'est lui qui détecte si un serveur est en panne grâce au système de "heartbeat" (battement de cœur).

**Pacemaker** gère les ressources : il décide quel serveur doit héberger le service et déclenche les bascules quand c'est nécessaire.

**PCS** nous permet de configurer tout ça facilement en ligne de commande.

**Nginx** est le service web qu'on va rendre hautement disponible.

---

## 🛠️ Mise en place étape par étape

### Étape 1 : Préparer les serveurs

Sur **node1** et **node2**, commencez par éditer le fichier `/etc/hosts` pour que les machines puissent se trouver par leur nom :

```bash
sudo nano /etc/hosts
```

Ajoutez ces lignes :
```
192.168.36.139 admin
192.168.36.140 node1
192.168.36.141 node2
```

Ensuite, installez Nginx mais ne le démarrez pas (c'est Pacemaker qui le fera) :

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl stop nginx
sudo systemctl disable nginx
```

Pour pouvoir différencier les deux serveurs lors des tests, personnalisez la page d'accueil :

Sur **node1** :
```bash
echo "<h1>Je suis NODE 1</h1>" | sudo tee /var/www/html/index.html
```

Sur **node2** :
```bash
echo "<h1>Je suis NODE 2</h1>" | sudo tee /var/www/html/index.html
```

---

### Étape 2 : Installer les composants du cluster

Sur **les deux nœuds**, installez Pacemaker, Corosync et PCS :

```bash
sudo apt install pacemaker corosync pcs -y
```

Démarrez le service PCS :
```bash
sudo systemctl start pcsd
sudo systemctl enable pcsd
```

Définissez un mot de passe pour l'utilisateur système `hacluster` (utilisez le même sur les deux nœuds) :
```bash
sudo passwd hacluster
```

---

### Étape 3 : Créer le cluster

Cette partie se fait **uniquement depuis node1**.

Authentifiez les deux nœuds :
```bash
sudo pcs host auth node1 node2 -u hacluster
```
Le système vous demandera le mot de passe que vous venez de définir.

Créez le cluster :
```bash
sudo pcs cluster setup web_cluster node1 node2 --force
```

Démarrez le cluster sur tous les nœuds :
```bash
sudo pcs cluster start --all
sudo pcs cluster enable --all
```

Pour ce TP avec seulement 2 nœuds, on désactive certaines protections qui nécessiteraient 3 nœuds minimum :
```bash
sudo pcs property set stonith-enabled=false
sudo pcs property set no-quorum-policy=ignore
```

Vérifiez que tout fonctionne :
```bash
sudo pcs status
```
Vous devriez voir les deux nœuds "Online".

---

### Étape 4 : Configurer les ressources

Maintenant qu'on a un cluster qui fonctionne, on va lui dire quoi gérer.

**Créer l'IP virtuelle** :
```bash
sudo pcs resource create Virtual_IP ocf:heartbeat:IPaddr2 \
  ip=192.168.36.150 cidr_netmask=24 op monitor interval=30s
```

Cette commande crée une IP flottante qui sera surveillée toutes les 30 secondes.

**Créer la ressource Nginx** :
```bash
sudo pcs resource create Web_Server ocf:heartbeat:nginx \
  configfile=/etc/nginx/nginx.conf op monitor interval=30s
```

**Définir les règles de fonctionnement** :

On veut que Nginx et l'IP virtuelle soient toujours sur le même nœud :
```bash
sudo pcs constraint colocation add Web_Server with Virtual_IP INFINITY
```

Et on veut que l'IP virtuelle démarre avant Nginx :
```bash
sudo pcs constraint order Virtual_IP then Web_Server
```

Vérifiez la configuration :
```bash
sudo pcs status resources
```

---

### Étape 5 : Tester la haute disponibilité

Depuis la machine **admin**, testez l'accès au service :
```bash
curl http://192.168.36.150
```

Vous devriez voir "Je suis NODE 1" ou "Je suis NODE 2" selon le nœud actif.

**Simulons maintenant une panne** en arrêtant le nœud actif :
```bash
sudo pcs cluster stop node1
```

Attendez quelques secondes, puis refaites le test depuis admin :
```bash
curl http://192.168.36.150
```

Le service devrait maintenant répondre depuis node2. La bascule s'est faite automatiquement !

Redémarrez node1 quand vous êtes prêt :
```bash
sudo pcs cluster start node1
```

---

## ✨ Ce que vous devez observer

- Les deux nœuds apparaissent comme "Online" dans `pcs status`
- L'IP virtuelle 192.168.36.150 est active sur un seul nœud à la fois
- Le service Nginx répond toujours, même quand un nœud tombe
- La page affichée change automatiquement quand la ressource bascule d'un nœud à l'autre

---

## 🔧 Problèmes courants et solutions

**Les nœuds ne se voient pas** : vérifiez votre fichier `/etc/hosts` et que les machines peuvent se pinguer.

**Corosync ne démarre pas** : assurez-vous que les firewalls ne bloquent pas les ports (UDP 5405 pour Corosync).

**Les ressources ne démarrent pas** : consultez les logs avec `sudo journalctl -xe` et vérifiez la configuration Nginx.

---

## 🎓 Ce qu'on retient

Ce TP montre comment mettre en place une haute disponibilité au niveau infrastructure. Contrairement à la redondance applicative (comme les réplicas Kubernetes), ici on protège l'infrastructure elle-même.

L'avantage : pas besoin de modifier l'application. L'inconvénient : la bascule prend quelques secondes, là où une solution applicative peut être instantanée.

Pacemaker et Corosync sont particulièrement adaptés pour des services legacy ou des bases de données qui ne supportent pas nativement le clustering.

---

**TP réalisé par Victor**
