# Tutoriel ROS 2

Ce dépôt contient un cours pratique sur **ROS 2 Jazzy Jalisco**, orienté vers les systèmes autonomes. L'environnement de travail est fourni sous forme de conteneur Docker ouvert dans VS Code : aucune installation de ROS 2 sur la machine hôte n'est nécessaire.

---

## Objectifs du cours

- **Découvrir ROS 2 en tant qu'utilisateur** : graphe de calcul, commandes CLI, outils d'introspection et de visualisation (`rqt`, `rviz2`, TF2, `ros2 bag`), écosystème de packages pour les systèmes autonomes.
- **Développer une application ROS 2** : workspace, packages, nœuds, topics, paramètres, launch files, interfaces, services et actions.
- **Réaliser un projet de synthèse** : le logiciel d'une voiture autonome pour le challenge CoVAPSy, en simulation sous Webots.

---

## Prérequis

- **Système d'exploitation** :
  - **Ubuntu 24.04** (recommandé) ;
  - **Windows 10/11** via WSL2.
- **Logiciels** :
  - **Git** ;
  - **Docker** (Docker Engine sous Linux, Docker Desktop sous Windows) ;
  - **Visual Studio Code** avec l'extension **Dev Containers** ;
  - Sous Windows 10 uniquement : un serveur X11 (VcXsrv). Sous Windows 11, WSLg prend en charge l'affichage graphique.

Aucun GPU n'est nécessaire : tous les outils du cours (turtlesim, `rqt`, `rviz2`) fonctionnent avec un rendu logiciel.

---

## Structure du dépôt

| Dossier | Contenu |
| --- | --- |
| 📂 `cours/` | **Cours** : Partie I (découvrir ROS 2) et fiches de référence (Partie II). Point d'entrée : [`cours/README.md`](cours/README.md). |
| 📂 `docker/` | `Dockerfile` de l'image ROS 2 Jazzy. |
| 📂 `.devcontainer/` | Configuration du conteneur pour VS Code. |

---

## Installation

### Sous Ubuntu

1. **Cloner le dépôt** :
   ```bash
   git clone https://github.com/F-KHENFRI/ros2_tutorial.git
   cd ros2_tutorial
   ```
2. **Autoriser l'affichage graphique depuis le conteneur** (à refaire après chaque redémarrage) :
   ```bash
   xhost +local:docker
   ```
   > Cette commande autorise uniquement les clients X locaux lancés par Docker. Évitez `xhost +`, qui ouvre l'affichage à tout le réseau.
3. **Ouvrir le projet dans VS Code** :
   ```bash
   code .
   ```
   Acceptez la notification **Reopen in Container**, ou passez par la *Command Palette* (`Ctrl+Shift+P`) → **Dev Containers: Rebuild and Reopen in Container**.

### Sous Windows

1. **Affichage graphique** :
   - Windows 11 : rien à faire (WSLg).
   - Windows 10 : installez [VcXsrv](https://sourceforge.net/projects/vcxsrv/) et lancez-le en cochant *Disable access control*.
2. **Cloner le dépôt** (branche dédiée à Windows) dans un terminal PowerShell :
   ```bash
   git clone https://github.com/F-KHENFRI/ros2_tutorial.git -b win_env
   cd ros2_tutorial
   ```
3. **Ouvrir le projet dans VS Code**, puis *Command Palette* → **Dev Containers: Rebuild and Reopen in Container**.

### Tester l'environnement

Dans le terminal intégré de VS Code (donc dans le conteneur) :
```bash
ros2 doctor --report | head -20   # vérifie l'installation ROS 2
ros2 run turtlesim turtlesim_node # une fenêtre avec une tortue doit s'ouvrir
```

---

## Résolution des problèmes

### « cannot open display » / « could not connect to display »
- Vérifiez la variable `DISPLAY` dans le conteneur :
  ```bash
  echo $DISPLAY
  ```
  Sous Ubuntu elle vaut en général `:0` ou `:1`.
- Sous Ubuntu, relancez `xhost +local:docker` sur la machine hôte.
- Sous Windows 10, vérifiez que VcXsrv tourne avec *Disable access control*.

### Problèmes de permissions Docker (Ubuntu)
```bash
sudo usermod -aG docker $USER
```
Déconnectez-vous puis reconnectez-vous pour appliquer le changement.

### Les nœuds de deux étudiants se voient sur le réseau
Le conteneur utilise `--network=host` : sur un même réseau, tous les nœuds ROS 2 partageant le même `ROS_DOMAIN_ID` se découvrent mutuellement. Choisissez un identifiant propre à chaque poste (entre 0 et 101) :
```bash
echo "export ROS_DOMAIN_ID=<numéro_de_poste>" >> ~/.bashrc
```
