# 09 – Interfaces personnalisées

## Objectifs pédagogiques

À la fin de ce chapitre, vous saurez :

- écrire des fichiers `.msg`, `.srv` et `.action` en respectant leur syntaxe et leurs règles de nommage ;
- créer un package d'interfaces (`ament_cmake`), le compiler et utiliser ses types depuis un package Python ;
- décider s'il faut créer une nouvelle interface ou réutiliser une interface standard.

**Prérequis** : [08 – Launch files](08_launch.md).

---

## Sommaire

- [1. Avant de créer : réutiliser](#1-avant-de-créer--réutiliser)
- [2. Syntaxe des interfaces](#2-syntaxe-des-interfaces)
- [3. Le package `turtle_interfaces`](#3-le-package-turtle_interfaces)
- [4. Compiler et vérifier](#4-compiler-et-vérifier)
- [5. Utiliser les interfaces depuis Python](#5-utiliser-les-interfaces-depuis-python)
- [6. Bonnes pratiques](#6-bonnes-pratiques)
- [7. Exercice fil rouge](#7-exercice-fil-rouge)
- [8. Questions d'évaluation](#8-questions-dévaluation)

---

## 1. Avant de créer : réutiliser

Une interface personnalisée a un coût : les autres outils et packages ne la comprennent pas (rviz2, Nav2, `rqt_plot` pour les champs non numériques…). **Cherchez d'abord** dans les interfaces standard :

| Package | Exemples |
| --- | --- |
| `std_msgs`, `std_srvs` | `Header`, `Bool`, `Float64` ; `Empty`, `SetBool`, `Trigger` |
| `geometry_msgs` | `Pose`, `PoseStamped`, `Twist`, `TwistStamped`, `TransformStamped`, `Vector3` |
| `sensor_msgs` | `LaserScan`, `Image`, `Imu`, `PointCloud2`, `JointState`, `NavSatFix` |
| `nav_msgs` | `Odometry`, `Path`, `OccupancyGrid` |
| `nav2_msgs` | Actions `NavigateToPose`, `FollowWaypoints`, `ComputePathToPose` |

```bash
ros2 interface list | grep -i pose
ros2 interface show geometry_msgs/msg/PoseStamped
```

Créez une interface lorsque la sémantique est propre à votre application (état interne d'un contrôleur, requête métier), ou lorsqu'une interface standard obligerait à détourner des champs de leur sens.

---

## 2. Syntaxe des interfaces

### 2.1 Types de champs

| Catégorie | Types |
| --- | --- |
| Booléen | `bool` |
| Entiers | `int8`, `uint8`, `int16`, `uint16`, `int32`, `uint32`, `int64`, `uint64` |
| Réels | `float32`, `float64` |
| Texte | `string`, `string<=N` (borné) |
| Tableaux | `float64[]` (dynamique), `float64[3]` (fixe), `float64[<=10]` (borné) |
| Composés | tout autre message : `geometry_msgs/Pose`, `builtin_interfaces/Time` |

### 2.2 Conventions

- **Nom de fichier** en `CamelCase` : `ControllerState.msg`, `SetTarget.srv`, `GoToPose.action`.
- **Nom de champ** en `snake_case` minuscule : `heading_error`.
- **Constantes** en MAJUSCULES : `uint8 PHASE_HEADING=2`.
- **Valeur par défaut** possible : `float64 tolerance 0.05`.
- **Commentaires** avec `#` : documentez unités et repères.

### 2.3 Les trois formes

```text
# msg : une structure
float64 x
float64 y
```

```text
# srv : requête --- réponse
float64 x
---
bool success
```

```text
# action : goal --- result --- feedback
float64 x
---
bool success
---
float64 progress
```

---

## 3. Le package `turtle_interfaces`

Les interfaces doivent être générées par CMake : le package est donc **toujours** de type `ament_cmake`, même si tout le code est en Python. On le sépare des packages de code pour que d'autres packages (et d'autres robots) puissent en dépendre sans tirer le code du contrôleur.

```bash
cd /workspaces/ros2_tutorial/turtle_ws/src
ros2 pkg create turtle_interfaces --build-type ament_cmake --license Apache-2.0
cd turtle_interfaces
rm -rf include src            # pas de code C++ dans ce package
mkdir msg srv action
```

### 3.1 `msg/ControllerState.msg`

```text
# État interne du régulateur, publié à chaque cycle (débogage, rqt_plot)
uint8 PHASE_IDLE=0               # pas de cible active, commande nulle
uint8 PHASE_MOVING=1             # rho > tolérance : déplacement vers la cible
uint8 PHASE_DONE=2               # rho <= tolérance : cible atteinte, commande nulle

std_msgs/Header header
uint8 phase
float64 rho                      # distance à la cible (m)
float64 alpha                    # erreur de cap vers la cible (rad, dans ]-pi, pi])
geometry_msgs/Twist command      # commande publiée
```

### 3.2 `srv/SetTarget.srv`

```text
# Requête : nouvelle position cible, dans le repère du simulateur
float64 x                        # m, dans [0, 11]
float64 y                        # m, dans [0, 11]
---
bool success
string message
```

### 3.3 `action/GoToPose.action`

```text
# Goal : position à atteindre
float64 x                        # m, dans [0, 11]
float64 y                        # m, dans [0, 11]
---
# Result
bool success
float64 final_distance           # m
builtin_interfaces/Duration elapsed
---
# Feedback
uint8 phase                      # mêmes valeurs que ControllerState.PHASE_*
float64 distance_remaining       # m
```

### 3.4 `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.8)
project(turtle_interfaces)

find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)
find_package(std_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(builtin_interfaces REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/ControllerState.msg"
  "srv/SetTarget.srv"
  "action/GoToPose.action"
  DEPENDENCIES std_msgs geometry_msgs builtin_interfaces
)

ament_export_dependencies(rosidl_default_runtime)
ament_package()
```

> `DEPENDENCIES` liste les packages dont les interfaces sont **utilisées dans vos champs**. L'oublier provoque une erreur de génération (*« Unknown type »*).

### 3.5 `package.xml`

```xml
<?xml version="1.0"?>
<package format="3">
  <name>turtle_interfaces</name>
  <version>0.0.1</version>
  <description>Interfaces du fil rouge turtle_controller</description>
  <maintainer email="etudiant@example.com">Étudiant</maintainer>
  <license>Apache-2.0</license>

  <buildtool_depend>ament_cmake</buildtool_depend>
  <buildtool_depend>rosidl_default_generators</buildtool_depend>

  <depend>std_msgs</depend>
  <depend>geometry_msgs</depend>
  <depend>builtin_interfaces</depend>
  <depend>action_msgs</depend>          <!-- obligatoire dès qu'il y a un .action -->

  <exec_depend>rosidl_default_runtime</exec_depend>
  <member_of_group>rosidl_interface_packages</member_of_group>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

---

## 4. Compiler et vérifier

```bash
cd /workspaces/ros2_tutorial/turtle_ws
colcon build --packages-select turtle_interfaces
source install/setup.bash                       # indispensable pour voir les nouveaux types

ros2 interface package turtle_interfaces
ros2 interface show turtle_interfaces/msg/ControllerState
ros2 interface show turtle_interfaces/srv/SetTarget
ros2 interface show turtle_interfaces/action/GoToPose
```

Erreurs fréquentes :

| Symptôme | Cause probable |
| --- | --- |
| `Unknown type 'geometry_msgs/Twist'` | Package absent de `DEPENDENCIES` ou de `find_package` |
| Erreur `action_msgs` à la compilation | `<depend>action_msgs</depend>` manquant |
| `ModuleNotFoundError: turtle_interfaces` | Workspace non sourcé après compilation |
| Le nom d'un champ est refusé | Majuscule ou chiffre en tête : les champs sont en `snake_case` |

---

## 5. Utiliser les interfaces depuis Python

Dans `turtle_controller/package.xml` :

```xml
<depend>turtle_interfaces</depend>
```

`colcon` compile alors `turtle_interfaces` **avant** `turtle_controller`.

```python
from turtle_interfaces.msg import ControllerState
from turtle_interfaces.srv import SetTarget
from turtle_interfaces.action import GoToPose

state = ControllerState()
state.header.stamp = self.get_clock().now().to_msg()
state.phase = ControllerState.PHASE_MOVING
state.rho = rho

req = SetTarget.Request(x=8.0, y=8.0)
goal = GoToPose.Goal(x=8.0, y=8.0)
```

Une interface de service génère `SetTarget.Request` et `SetTarget.Response`. Une interface d'action génère `GoToPose.Goal`, `GoToPose.Result` et `GoToPose.Feedback`.

---

## 6. Bonnes pratiques

1. **Réutiliser** les interfaces standard dès que la sémantique correspond.
2. **Un package d'interfaces dédié**, sans code, dont dépendent les packages de code.
3. **Documenter unités et repères** dans les commentaires de chaque champ.
4. **Ajouter un `std_msgs/Header`** (horodatage, repère) aux messages qui décrivent une mesure ou un état.
5. **Constantes nommées** plutôt que des entiers magiques.
6. **Penser compatibilité** : modifier une interface casse tous les nœuds compilés avec l'ancienne version. Ajoutez des champs plutôt que de les renommer.

---

## 7. Exercice fil rouge

1. Créez `turtle_interfaces` avec les trois fichiers de la section 3, compilez-le et vérifiez-le avec `ros2 interface show`.
2. Ajoutez la dépendance dans `turtle_controller/package.xml`.
3. Dans `pid_regulator`, représentez l'état du régulateur (au repos, en déplacement, arrivé : tableau de synthèse du chapitre 06, § 7.5) par les constantes `ControllerState.PHASE_*` (le chapitre 11 en dépend), puis publiez un `ControllerState` à chaque cycle sur le topic privé `~/state`.
4. Tracez `rho` et `alpha` :
   ```bash
   ros2 run rqt_plot rqt_plot /pid_regulator/state/rho /pid_regulator/state/alpha
   ```
5. Affichez uniquement la phase en cours :
   ```bash
   ros2 topic echo /pid_regulator/state --field phase
   ```

**Critère de validation** : `colcon build` depuis un workspace propre (`rm -rf build install log`) compile les deux packages dans le bon ordre, sans intervention.

---

## 8. Questions d'évaluation

1. Pourquoi un package d'interfaces doit-il être de type `ament_cmake` ?
2. Quel est le rôle de `<member_of_group>rosidl_interface_packages</member_of_group>` ?
3. Vous ajoutez un champ `geometry_msgs/Point target` à `SetTarget.srv`. Quelles lignes de `CMakeLists.txt` et de `package.xml` vérifiez-vous ?
4. Pourquoi est-il préférable d'utiliser `geometry_msgs/msg/PoseStamped` plutôt qu'un message personnalisé contenant `x` et `y`, si l'on veut afficher la cible dans rviz2 ?
5. Quel est l'ordre des trois sections d'un fichier `.action` ?

## Références

- Créer des interfaces (Jazzy) : <https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Custom-ROS2-Interfaces.html>
- Interfaces standard : <https://github.com/ros2/common_interfaces>
- Conventions d'interfaces : <https://docs.ros.org/en/jazzy/Concepts/Basic/About-Interfaces.html>
