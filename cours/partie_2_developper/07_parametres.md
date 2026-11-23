# 07 – Paramètres

## Objectifs pédagogiques

| Niveau (Bloom) | Objectif |
| --- | --- |
| Comprendre | Décrire le cycle de vie d'un paramètre : déclaration, surcharge au lancement, modification à l'exécution. |
| Appliquer | Déclarer des paramètres typés avec descripteurs (description, bornes, lecture seule). |
| Appliquer | Charger un fichier YAML et réagir aux modifications avec des callbacks. |
| Analyser | Distinguer ce qui relève d'un paramètre, d'un topic ou d'un service. |

**Prérequis** : [06 – Topics et QoS](06_topics_et_qos.md).

---

## Sommaire

- [1. Notions](#1-notions)
- [2. Déclarer et lire](#2-déclarer-et-lire)
- [3. Descripteurs : documenter et contraindre](#3-descripteurs--documenter-et-contraindre)
- [4. Fichiers YAML](#4-fichiers-yaml)
- [5. Réagir aux modifications](#5-réagir-aux-modifications)
- [6. CLI](#6-cli)
- [7. Bonnes pratiques](#7-bonnes-pratiques)
- [8. Exercice fil rouge : gains et cible paramétrables](#8-exercice-fil-rouge--gains-et-cible-paramétrables)
- [9. Questions d'évaluation](#9-questions-dévaluation)

---

## 1. Notions

Un paramètre est une **valeur nommée, typée, attachée à un nœud**. Chaque nœud héberge son propre serveur de paramètres : il n'existe pas de serveur global comme dans ROS 1.

| Aspect | Règle |
| --- | --- |
| Portée | Locale au nœud : `/pid_regulator` et `/robot2/pid_regulator` ont des paramètres distincts. |
| Déclaration | **Obligatoire** avant usage (sauf option `allow_undeclared_parameters`, à éviter). |
| Types | `bool`, `int` (64 bits), `double`, `string`, `byte[]`, `bool[]`, `int[]`, `double[]`, `string[]`. |
| Tableaux | **Homogènes** : `[1.0, 0.5]` est un `double[]` ; `[1, 0.5]` est refusé. |
| Structures | **Pas de dictionnaire.** L'imbrication YAML devient des noms à points : `pid.linear.kp`. |
| Typage | Fixé par la valeur par défaut ; `2` (int) n'est pas `2.0` (double). |

**Ce qu'un paramètre n'est pas** : un canal de données. Une consigne qui change à chaque cycle passe par un **topic** ; une demande ponctuelle avec réponse passe par un **service** ou une **action** ; un **paramètre** règle le comportement du nœud (gains, seuils, fréquences, chemins, noms de repères).

---

## 2. Déclarer et lire

```python
class ParamDemo(Node):
    def __init__(self) -> None:
        super().__init__('param_demo')
        self.declare_parameter('rate_hz', 50.0)
        self.declare_parameter('frame_id', 'base_link')
        self.declare_parameter('waypoints', [1.0, 1.0, 8.0, 8.0])   # double[]

        rate = self.get_parameter('rate_hz').value                  # float
        frame = self.get_parameter('frame_id').get_parameter_value().string_value
        self.get_logger().info(f'rate={rate} Hz, frame={frame}')
```

Déclaration groupée avec un préfixe :

```python
self.declare_parameters(
    namespace='pid.linear',
    parameters=[('kp', 1.0), ('ki', 0.0), ('kd', 0.0)],
)
kp = self.get_parameter('pid.linear.kp').value
```

> **Piège de typage.** `self.declare_parameter('kp', 1)` crée un paramètre **entier**. Un `ros2 param set ... kp 1.5` sera alors refusé. Écrivez toujours `1.0` pour un réel.

---

## 3. Descripteurs : documenter et contraindre

```python
from rcl_interfaces.msg import FloatingPointRange, ParameterDescriptor

self.declare_parameter(
    'pid.linear.kp', 1.0,
    ParameterDescriptor(
        description='Gain proportionnel sur la distance (s⁻¹)',
        floating_point_range=[FloatingPointRange(from_value=0.0, to_value=20.0, step=0.0)],
    ),
)
self.declare_parameter(
    'rate_hz', 50.0,
    ParameterDescriptor(description='Fréquence de la boucle de commande', read_only=True),
)
```

| Champ | Effet |
| --- | --- |
| `description` | Affiché par `ros2 param describe` et `rqt_reconfigure` |
| `floating_point_range` / `integer_range` | Bornes vérifiées automatiquement ; curseur dans `rqt_reconfigure` |
| `read_only=True` | Modifiable seulement au lancement (YAML, `-p`), pas à l'exécution |
| `additional_constraints` | Texte libre décrivant d'autres contraintes |
| `dynamic_typing=True` | Autorise un changement de type (rarement souhaitable) |

> `rate_hz` en lecture seule : modifier la période d'un timer en cours d'exécution est possible, mais c'est rarement ce que l'on veut. Le figer évite une classe de bugs.

---

## 4. Fichiers YAML

Fichier `turtle_controller/config/pid_params.yaml` :

```yaml
/**:                       # s'applique à tout nœud chargeant ce fichier, quel que soit son nom ou son namespace
  ros__parameters:
    rate_hz: 50.0
    pid:
      linear:  {kp: 1.0, ki: 0.0, kd: 0.0, u_max: 2.0}
      angular: {kp: 4.0, ki: 0.0, kd: 0.0, u_max: 2.0}
    target: [8.0, 8.0]
    tolerance:
      position: 0.05
```

- La clé de premier niveau est le **nom complet du nœud** (`/robot1/pid_regulator`) ou le joker `/**`.
- `ros__parameters` (deux tirets bas) est obligatoire.
- Les nombres doivent porter le bon type : `1.0`, et non `1`, pour un `double`.

Chargement :

```bash
ros2 run turtle_controller pid_regulator --ros-args --params-file config/pid_params.yaml
ros2 run turtle_controller pid_regulator --ros-args -p pid.linear.kp:=2.0      # surcharge ponctuelle
```

**Ordre de priorité** (du plus faible au plus fort) : valeur par défaut du code < fichier YAML < `-p` en ligne de commande < `ros2 param set` à l'exécution.

Installation du dossier `config/` (dans `setup.py`) :

```python
import os
from glob import glob
...
data_files=[
    ('share/ament_index/resource_index/packages', ['resource/' + package_name]),
    ('share/' + package_name, ['package.xml']),
    (os.path.join('share', package_name, 'config'), glob('config/*.yaml')),
],
```

Après `colcon build`, le fichier est installé dans `install/turtle_controller/share/turtle_controller/config/`, où le launch file le retrouvera (chapitre 08).

---

## 5. Réagir aux modifications

Depuis Iron, `rclpy` offre trois points d'accroche :

| Callback | Moment | Usage |
| --- | --- | --- |
| `add_pre_set_parameters_callback` | Avant validation | Modifier ou compléter la liste demandée (rare) |
| `add_on_set_parameters_callback` | Validation | **Accepter ou refuser** ; ne pas modifier l'état du nœud ici |
| `add_post_set_parameters_callback` | Après acceptation | **Appliquer** les nouvelles valeurs à l'état du nœud |

```python
from rcl_interfaces.msg import SetParametersResult
from rclpy.parameter import Parameter


class PidRegulator(Node):
    def __init__(self):
        ...
        self.add_on_set_parameters_callback(self.validate_params)
        self.add_post_set_parameters_callback(self.apply_params)

    def validate_params(self, params: list[Parameter]) -> SetParametersResult:
        for p in params:
            if p.name == 'target':
                v = p.value
                if len(v) != 2 or not (0.0 <= v[0] <= 11.0 and 0.0 <= v[1] <= 11.0):
                    return SetParametersResult(successful=False,
                                               reason='target = [x, y], x et y dans [0, 11]')
        return SetParametersResult(successful=True)

    def apply_params(self, params: list[Parameter]) -> None:
        for p in params:
            if p.name.startswith('pid.'):
                self.reload_gains(p)          # recopie les gains dans les objets Pid et les remet à zéro
            elif p.name == 'target':
                self.set_target(*p.value)
```

> Le callback de validation reçoit **la liste** des paramètres d'une même requête (`ros2 param load` en modifie plusieurs à la fois). La requête est atomique : soit tout est accepté, soit rien.

---

## 6. CLI

```bash
ros2 param list /pid_regulator
ros2 param describe /pid_regulator pid.linear.kp
ros2 param get /pid_regulator pid.linear.kp
ros2 param set /pid_regulator pid.linear.kp 2.0
ros2 param set /pid_regulator target "[5.0, 5.0]"
ros2 param dump /pid_regulator > reglage_final.yaml
ros2 param load /pid_regulator reglage_final.yaml
ros2 run rqt_reconfigure rqt_reconfigure        # curseurs pour les paramètres bornés
```

---

## 7. Bonnes pratiques

1. **Déclarer tous les paramètres dans `__init__`**, avec une valeur par défaut sûre et un descripteur.
2. **Noms hiérarchiques en `snake_case`** (`pid.linear.kp`) plutôt que `Kp_lin`.
3. **Unités dans la description** (m, rad, s⁻¹).
4. **Valider dans `on_set`, appliquer dans `post_set`.**
5. **Remettre à zéro** l'état interne dépendant d'un gain (intégrale) lorsqu'il change.
6. **Un fichier YAML par configuration** (simulation, robot réel) plutôt que des valeurs codées en dur.

---

## 8. Exercice fil rouge : gains et cible paramétrables

**Objectif** : rendre `pid_regulator` entièrement configurable sans recompilation.

1. Déclarez avec descripteurs :
   - `pid.linear.{kp,ki,kd,u_max}`, `pid.angular.{…}`: réels, bornés à $[0, 20]$ ;
   - `target` : `double[]` de longueur 2, valeur par défaut `[8.0, 8.0]` ;
   - `tolerance.position` : réels positifs ;
   - `rate_hz` : en lecture seule.
2. Créez `config/pid_params.yaml` et installez-le via `setup.py`.
3. Implémentez `validate_params` (longueur et domaine de `target`).
4. Testez :
   ```bash
   ros2 run turtle_controller pid_regulator --ros-args \
     --params-file install/turtle_controller/share/turtle_controller/config/pid_params.yaml
   ros2 param set /pid_regulator target "[2.0, 9.0]"       # la tortue repart
   ros2 param set /pid_regulator target "[20.0, 9.0]"      # doit être refusé
   ros2 param set /pid_regulator pid.linear.kp 25.0             # doit être refusé (borne)
   ros2 param set /pid_regulator rate_hz 10.0                   # doit être refusé (lecture seule)
   ```
5. Réglez les gains en direct avec `rqt_reconfigure` tout en observant `rqt_plot`. Sauvegardez le réglage obtenu avec `ros2 param dump`.


**Question d'analyse** : modifier `target` par `ros2 param set` fonctionne, mais le client ne sait pas quand la tortue est arrivée. Quel mécanisme de communication serait plus adapté ? (Réponse aux chapitres 10 et 11.)

---

## 9. Questions d'évaluation

1. Pourquoi faut-il déclarer un paramètre avant de l'utiliser ? 
2. Comment représenter un ensemble de gains structuré sachant qu'il n'existe pas de paramètre de type dictionnaire ?
3. Classez par priorité croissante : valeur par défaut, `-p`, fichier YAML, `ros2 param set`.
4. Pourquoi séparer validation (`on_set`) et application (`post_set`) ?
5. Une consigne de vitesse change à 50 Hz. Paramètre ou topic ? Justifiez.

## Références

- Documentation des paramètres (Jazzy) : <https://docs.ros.org/en/jazzy/Concepts/Basic/About-Parameters.html>
- Tutoriel « Using parameters in a class (Python) » : <https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Using-Parameters-In-A-Class-Python.html>
