# Tutoriel ROS 2

Ce dépôt contient un cours pratique sur **ROS 2 Jazzy Jalisco**, orienté vers les systèmes autonomes. L'environnement de travail est un conteneur **Podman** ouvert dans VS Code : il n'y a rien à installer côté ROS 2 sur la machine hôte.

Sur les postes de la salle, **Podman, VS Code et l'affichage graphique sont déjà configurés** : suivez directement la [Partie 1](#partie-1--récupérer-le-cours-et-lancer-le-conteneur). Pour travailler sur votre PC personnel, voir la [Partie 2](#partie-2--utiliser-limage-sur-un-pc-personnel).

---

## Objectifs du cours

- **Découvrir ROS 2 en tant qu'utilisateur** : graphe de calcul, commandes CLI, outils d'introspection et de visualisation (`rqt`, `rviz2`, TF2, `ros2 bag`), écosystème de packages pour les systèmes autonomes.
- **Développer une application ROS 2** : workspace, packages, nœuds, topics, paramètres, launch files, interfaces, services et actions.
- **Réaliser un projet de synthèse** : le logiciel d'une voiture autonome pour le challenge CoVAPSy, en simulation sous Webots.

---

## Structure du dépôt

| Dossier | Contenu |
| --- | --- |
| 📂 `cours/` | **Cours** : Partie I (découvrir ROS 2) et Partie II (développer). Point d'entrée : [`cours/README.md`](cours/README.md). |
| 📂 `docker/` | `Dockerfile` de l'image ROS 2 Jazzy. |
| 📂 `.devcontainer/` | Configuration du conteneur pour VS Code (`devcontainer.json`). |

---

## Partie 1 : récupérer le cours et lancer le conteneur

### 1.1 Cloner le dépôt (première fois uniquement)

```bash
git clone https://github.com/F-KHENFRI/ros2_tutorial.git
cd ros2_tutorial
```

### 1.2 Mettre à jour le dépôt (si vous l'avez déjà cloné)

Ne clonez pas le dépôt une deuxième fois. Le cours évolue au fil des séances : récupérez les nouveautés avec `git pull`, dans un terminal **sur l'hôte** (pas dans le conteneur) :

```bash
cd ros2_tutorial
git pull
```

Les fichiers du dépôt sont montés dans le conteneur : les mises à jour y apparaissent directement, sans reconstruire le conteneur.

**Si `git pull` refuse à cause de vos modifications locales** (message *« Your local changes would be overwritten by merge »*), mettez-les de côté, récupérez la mise à jour, puis réappliquez-les :

```bash
git stash        # met vos modifications de côté
git pull         # récupère la mise à jour
git stash pop    # réapplique vos modifications
```

S'il reste des conflits après `git stash pop`, Git indique les fichiers concernés ; corrigez-les dans VS Code.

### 1.3 Lancer le conteneur

Ouvrez le projet dans VS Code :

```bash
cd ros2_tutorial
code .
```

**La toute première fois : Rebuild (une seule fois)**

`Ctrl+Shift+P` → **Dev Containers: Rebuild and Reopen in Container**.

La construction télécharge l'image ROS 2 et prend plusieurs minutes. Attendez qu'elle se termine.

> ⚠️ **Le Rebuild ne se fait qu'une seule fois.** Il reconstruit l'image et le conteneur depuis zéro : c'est long, et tout ce que vous avez installé à la main dans le conteneur est perdu.

**Ensuite, à chaque séance : Reopen in Container**

Acceptez la notification **Reopen in Container**, ou `Ctrl+Shift+P` → **Dev Containers: Reopen in Container**. Le conteneur déjà construit redémarre en quelques secondes.

> Seule exception : si une mise à jour modifie `docker/Dockerfile` ou `.devcontainer/devcontainer.json` et que l'enseignant vous le demande, refaites **Rebuild and Reopen in Container** une fois.

### 1.4 Choisir son `ROS_DOMAIN_ID`

Le conteneur utilise `--network=host` : sur un même réseau, tous les nœuds ROS 2 qui partagent le même `ROS_DOMAIN_ID` se découvrent entre eux, et les nœuds de deux étudiants se mélangent. Dans le terminal du conteneur, choisissez un identifiant propre à votre poste (entre 0 et 101) :

```bash
echo "export ROS_DOMAIN_ID=<numéro_de_poste>" >> ~/.bashrc
source ~/.bashrc
```

### 1.5 Tester l'environnement

Dans le terminal intégré de VS Code (donc dans le conteneur) :

```bash
ros2 doctor --report | head -20    # vérifie l'installation ROS 2
ros2 run turtlesim turtlesim_node  # une fenêtre avec une tortue doit s'ouvrir
```

---

## Partie 2 : utiliser l'image sur un PC personnel

Cette partie concerne uniquement votre propre machine. Elle suppose **Ubuntu 24.04**. Une fois ces étapes faites, revenez à la [Partie 1](#partie-1--récupérer-le-cours-et-lancer-le-conteneur).

### 2.1 Installer les logiciels

```bash
sudo apt update
sudo apt install -y git podman x11-xserver-utils
```

Installez ensuite [Visual Studio Code](https://code.visualstudio.com/) et son extension **Dev Containers** (`ms-vscode-remote.remote-containers`).

### 2.2 Configurer VS Code pour Podman

`Ctrl+Shift+P` → **Preferences: Open User Settings (JSON)**, puis ajoutez :

```json
"dev.containers.dockerPath": "podman"
```

### 2.3 Autoriser l'affichage graphique

Sur l'hôte, avant d'ouvrir le conteneur (à refaire après chaque redémarrage ou reconnexion) :

```bash
xhost +local:
```

> Cette commande n'autorise que les clients X locaux. N'utilisez pas `xhost +`, qui ouvre l'affichage à tout le réseau.

### 2.4 Désactiver les certificats de l'école

Sur les postes de la salle, l'image ajoute les certificats réseau de l'école, lus dans `/usr/local/share/ca-certificates/estaca`. Ce dossier n'existe pas sur votre PC et la construction échoue. Désactivez cette étape **à deux endroits** avant le Rebuild.

Dans [`.devcontainer/devcontainer.json`](.devcontainer/devcontainer.json), commentez la ligne `options` :

```jsonc
"build": {
    "context": "..",
    "dockerfile": "../docker/Dockerfile",
    //"options": ["--build-context=hostcerts=/usr/local/share/ca-certificates/estaca"]
},
```

Dans [`docker/Dockerfile`](docker/Dockerfile), commentez les deux lignes qui utilisent ces certificats :

```dockerfile
#COPY --from=hostcerts /*.crt /usr/local/share/ca-certificates/
#RUN update-ca-certificates
```

> Commenter seulement le `devcontainer.json` ne suffit pas : sans le contexte `hostcerts`, Podman essaie de télécharger une image nommée `hostcerts` et la construction échoue.

Ces deux fichiers sont suivis par Git. Pour qu'ils ne bloquent pas un `git pull`, utilisez `git stash` / `git pull` / `git stash pop` (voir [1.2](#12-mettre-à-jour-le-dépôt-si-vous-lavez-déjà-cloné)).

### 2.5 (Optionnel) Activer le GPU NVIDIA

Par défaut, l'accès au GPU est **commenté** dans le `devcontainer.json` : tout fonctionne en rendu logiciel. C'est suffisant pour turtlesim, `rqt` et `rviz2`, mais Webots (projet CoVAPSy) est alors très lent. Si votre PC a une carte **NVIDIA**, vous pouvez donner le GPU au conteneur.

**1. Installer le pilote NVIDIA**, puis redémarrer :

```bash
sudo ubuntu-drivers install
sudo reboot
```

Vérifiez que le pilote fonctionne :

```bash
nvidia-smi
```

**2. Installer le NVIDIA Container Toolkit** :

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update
sudo apt install -y nvidia-container-toolkit
```

**3. Générer la description CDI du GPU** (c'est ce que Podman utilise pour exposer le GPU) :

```bash
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
nvidia-ctk cdi list     # doit afficher nvidia.com/gpu=all
```

> Refaites cette commande après chaque mise à jour du pilote NVIDIA.

Vérifiez que Podman voit le GPU :

```bash
podman run --rm --device nvidia.com/gpu=all ubuntu nvidia-smi
```

**4. Décommenter les lignes GPU** dans [`.devcontainer/devcontainer.json`](.devcontainer/devcontainer.json) :

```jsonc
"runArgs": [
    "--env", "DISPLAY=${localEnv:DISPLAY}",
    // GPU NVIDIA via CDI (Podman) : rendu OpenGL matériel pour Webots
    "--device", "nvidia.com/gpu=all",
    "--env", "NVIDIA_VISIBLE_DEVICES=all",
    "--env", "NVIDIA_DRIVER_CAPABILITIES=all",
    "--env", "__GLX_VENDOR_LIBRARY_NAME=nvidia",
    ...
],
```

**5. Reconstruire le conteneur** une fois : **Dev Containers: Rebuild and Reopen in Container**. Ensuite, revenez à **Reopen in Container**.

Dans le conteneur, `nvidia-smi` doit afficher votre carte.

---

## Résolution des problèmes

### « cannot open display » / « could not connect to display »

- Sur un PC personnel, relancez `xhost +local:` sur l'hôte, puis **Reopen in Container**.
- Vérifiez la variable `DISPLAY` dans le conteneur :
  ```bash
  echo $DISPLAY
  ```
  Elle vaut en général `:0` ou `:1`, la même valeur que sur l'hôte.

### VS Code cherche Docker au lieu de Podman

Message du type *« Docker is not installed »* : le paramètre `dev.containers.dockerPath` n'est pas réglé sur `podman` (voir [2.2](#22-configurer-vs-code-pour-podman)).

### Erreur sur `hostcerts` pendant la construction (PC personnel)

Les certificats de l'école n'ont pas été désactivés : voir [2.4](#24-désactiver-les-certificats-de-lécole).

### Erreur `nvidia.com/gpu=all` : « unresolvable CDI devices »

Les lignes GPU sont décommentées mais la description CDI est absente ou périmée. Relancez `sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml`, ou recommentez les lignes GPU.
