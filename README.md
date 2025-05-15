# Guide d'installation et d'utilisation de l'infrastructure IoT pour MagicMirror

Ce guide explique comment installer et configurer une infrastructure IoT complète pour votre MagicMirror de Rivière-du-Loup, comprenant trois composants principaux : InfluxDB, Node-RED et Grafana. Cette infrastructure vous permettra de collecter, stocker et visualiser des données de votre MagicMirror.

## Table des matières

1. [Vue d'ensemble](#vue-densemble)
2. [Prérequis](#prérequis)
3. [Installation avec Docker](#installation-avec-docker)
4. [Configuration de Node-RED](#configuration-de-node-red)
5. [Configuration d'InfluxDB](#configuration-dinfluxdb)
6. [Configuration de Grafana](#configuration-de-grafana)
7. [Intégration avec MQTT](#intégration-avec-mqtt)
8. [Dépannage](#dépannage)

## Vue d'ensemble

Voici les rôles de chaque composant dans notre infrastructure :

- **InfluxDB** : Base de données temporelle optimisée pour stocker des mesures horodatées comme la température, l'utilisation de la mémoire, etc.
- **Node-RED** : Outil de programmation visuelle pour connecter des appareils et des services. Il servira d'intermédiaire entre MQTT et InfluxDB.
- **Grafana** : Plateforme de visualisation pour créer des tableaux de bord élégants à partir des données stockées dans InfluxDB.
- **MQTT** : Protocole de messagerie légère utilisé pour la communication entre votre MagicMirror et l'infrastructure IoT.

## Prérequis

- Un serveur Linux Debian pour héberger l'infrastructure
- Docker et Docker Compose installés
- Accès à votre broker MQTT (mirrormqtt.jeremielapointe.ca)
- Ports requis : 3001 (Grafana), 8086 (InfluxDB), 1880 (Node-RED)

## Installation avec Docker

1. Créez un fichier nommé `install-iot-stack-simple.sh` avec le contenu suivant :

```bash
#!/bin/bash
# Script d'installation simple pour Node-RED, InfluxDB et Grafana

set -e

echo "===== Installation de l'infrastructure IoT de base ====="
echo "Ce script va installer Node-RED, InfluxDB et Grafana via Docker."

# Mise à jour du système
echo "===== Mise à jour du système ====="
sudo apt update && sudo apt upgrade -y

# Installation de Docker si nécessaire
if ! command -v docker &> /dev/null; then
    echo "===== Installation de Docker ====="
    curl -fsSL https://get.docker.com -o get-docker.sh
    sudo sh get-docker.sh
    sudo usermod -aG docker $USER
    rm get-docker.sh
fi

# Installation de Docker Compose si nécessaire
if ! command -v docker-compose &> /dev/null; then
    echo "===== Installation de Docker Compose ====="
    sudo apt install -y docker-compose
fi

# Créer un répertoire pour le projet
echo "===== Création des répertoires ====="
mkdir -p ~/iot-stack
cd ~/iot-stack

# Création du fichier docker-compose.yml
echo "===== Création du fichier docker-compose.yml ====="
cat > docker-compose.yml << 'EOL'
version: '3'

services:
  influxdb:
    image: influxdb:2.7
    container_name: influxdb
    restart: unless-stopped
    ports:
      - "8086:8086"
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=admin
      - DOCKER_INFLUXDB_INIT_PASSWORD=Patate123
      - DOCKER_INFLUXDB_INIT_ORG=MagicMirror
      - DOCKER_INFLUXDB_INIT_BUCKET=mirror_data
      - DOCKER_INFLUXDB_INIT_ADMIN_TOKEN=my-super-secret-auth-token
    volumes:
      - ./influxdb:/var/lib/influxdb2

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=Patate123
    volumes:
      - ./grafana:/var/lib/grafana

  node-red:
    image: nodered/node-red:latest
    container_name: node-red
    restart: unless-stopped
    ports:
      - "1880:1880"
    volumes:
      - ./node-red:/data
EOL

# Démarrer les services
echo "===== Démarrage des services ====="
docker-compose up -d

echo ""
echo "===== Installation terminée! ====="
echo ""
echo "Les services sont maintenant disponibles aux adresses suivantes :"
echo "- InfluxDB: http://$(hostname -I | awk '{print $1}'):8086"
echo "- Grafana: http://$(hostname -I | awk '{print $1}'):3001"
echo "- Node-RED: http://$(hostname -I | awk '{print $1}'):1880"
echo ""
echo "Identifiants par défaut :"
echo "- InfluxDB: admin / Patate123"
echo "- Grafana: admin / Patate123"
echo ""
echo "Étape suivante : Installation des nœuds MQTT dans Node-RED"
echo "1. Ouvrez Node-RED dans votre navigateur"
echo "2. Cliquez sur le menu (en haut à droite) > Manage palette"
echo "3. Onglet 'Install', cherchez et installez 'node-red-contrib-mqtt-broker'"
echo "4. Cherchez et installez 'node-red-contrib-influxdb'"
```

2. Rendez le script exécutable et lancez-le :

```bash
chmod +x install-iot-stack-simple.sh
./install-iot-stack-simple.sh
```

3. Si Docker n'est pas encore en cours d'exécution, démarrez-le :

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

4. Vérifiez que les conteneurs fonctionnent correctement :

```bash
docker ps
```

## Configuration de Node-RED

Node-RED est votre interface visuelle de programmation qui va connecter le broker MQTT à la base de données InfluxDB.

### Installation des nœuds supplémentaires

1. Ouvrez Node-RED dans votre navigateur à l'adresse `http://<IP-DU-SERVEUR>:1880`
2. Cliquez sur le menu (en haut à droite) > **Manage palette**
3. Dans l'onglet **Install**, recherchez et installez :
   - `node-red-contrib-mqtt-broker`
   - `node-red-contrib-influxdb`

### Création d'un flux pour les données système

1. Créez un nouveau flux en important le code JSON suivant (menu hamburger > Import) :

```json
[
  {
    "id": "a1b2c3d4.e5f6g7",
    "type": "tab",
    "label": "MagicMirror to InfluxDB",
    "disabled": false,
    "info": ""
  },
  {
    "id": "h8i9j0k1.l2m3n4",
    "type": "mqtt in",
    "z": "a1b2c3d4.e5f6g7",
    "name": "MagicMirror MQTT",
    "topic": "magicmirror/system",
    "qos": "0",
    "datatype": "json",
    "broker": "o5p6q7r8.s9t0u1",
    "nl": false,
    "rap": true,
    "rh": 0,
    "x": 180,
    "y": 120,
    "wires": [["v2w3x4y5.z6a7b8", "c9d0e1f2.g3h4i5"]]
  },
  {
    "id": "v2w3x4y5.z6a7b8",
    "type": "debug",
    "z": "a1b2c3d4.e5f6g7",
    "name": "Debug",
    "active": true,
    "tosidebar": true,
    "console": false,
    "tostatus": false,
    "complete": "payload",
    "targetType": "msg",
    "statusVal": "",
    "statusType": "auto",
    "x": 410,
    "y": 80,
    "wires": []
  },
  {
    "id": "c9d0e1f2.g3h4i5",
    "type": "function",
    "z": "a1b2c3d4.e5f6g7",
    "name": "Format pour InfluxDB",
    "func": "// Créer un point pour InfluxDB\nconst data = msg.payload;\n\nlet point = {\n    measurement: \"system\",\n    tags: {\n        host: data.hostname || \"magicmirror\"\n    },\n    fields: {\n        memory_used_percent: data.memory ? data.memory.usage : 0,\n        cpu_temp: data.cpuTemperature || 0,\n        uptime: data.uptime || 0\n    },\n    timestamp: new Date().getTime() * 1000000 // en nanoseconds\n};\n\nreturn { payload: [point] };",
    "outputs": 1,
    "noerr": 0,
    "initialize": "",
    "finalize": "",
    "libs": [],
    "x": 420,
    "y": 120,
    "wires": [["j6k7l8m9.n0o1p2"]]
  },
  {
    "id": "j6k7l8m9.n0o1p2",
    "type": "influxdb out",
    "z": "a1b2c3d4.e5f6g7",
    "influxdb": "q3r4s5t6.u7v8w9",
    "name": "Enregistrer dans InfluxDB",
    "measurement": "",
    "precision": "",
    "retentionPolicy": "",
    "database": "database",
    "precisionV18FluxV20": "ns",
    "retentionPolicyV18Flux": "",
    "org": "MagicMirror",
    "bucket": "mirror_data",
    "x": 650,
    "y": 120,
    "wires": []
  },
  {
    "id": "o5p6q7r8.s9t0u1",
    "type": "mqtt-broker",
    "name": "MagicMirror MQTT Broker",
    "broker": "mirrormqtt.jeremielapointe.ca",
    "port": "8883",
    "clientid": "node-red-client",
    "autoConnect": true,
    "usetls": true,
    "protocolVersion": "4",
    "keepalive": "60",
    "cleansession": true,
    "birthTopic": "",
    "birthQos": "0",
    "birthPayload": "",
    "birthMsg": {},
    "closeTopic": "",
    "closeQos": "0",
    "closePayload": "",
    "closeMsg": {},
    "willTopic": "",
    "willQos": "0",
    "willPayload": "",
    "willMsg": {},
    "sessionExpiry": "",
    "credentials": {
      "user": "MirrorMQTT",
      "password": "Patate123"
    }
  },
  {
    "id": "q3r4s5t6.u7v8w9",
    "type": "influxdb",
    "hostname": "influxdb",
    "port": "8086",
    "protocol": "http",
    "database": "database",
    "name": "InfluxDB",
    "usetls": false,
    "tls": "",
    "influxdbVersion": "2.0",
    "url": "http://influxdb:8086",
    "rejectUnauthorized": true,
    "credentials": {
      "username": "admin",
      "password": "Patate123",
      "token": "my-super-secret-auth-token"
    }
  }
]
```

2. Après l'importation, vous devrez configurer deux éléments :

   - Double-cliquez sur le nœud **MagicMirror MQTT Broker** pour configurer la connexion MQTT :

     - Serveur : mirrormqtt.jeremielapointe.ca
     - Port : 8883
     - Connexion sécurisée : cochez la case
     - Nom d'utilisateur : MirrorMQTT
     - Mot de passe : Patate123
     - ID client : node-red-client

   - Double-cliquez sur le nœud **Enregistrer dans InfluxDB** pour configurer la connexion InfluxDB :
     - Version : 2.0
     - URL : http://influxdb:8086
     - Token : my-super-secret-auth-token
     - Organisation : MagicMirror
     - Bucket : mirror_data

3. Cliquez sur le bouton **Deploy** pour déployer le flux.

## Configuration d'InfluxDB

InfluxDB est configuré automatiquement via les variables d'environnement dans le fichier Docker Compose. Cependant, vous pouvez vous connecter à l'interface Web pour vérifier que tout fonctionne correctement.

1. Ouvrez InfluxDB dans votre navigateur à l'adresse `http://<IP-DU-SERVEUR>:8086`
2. Connectez-vous avec les identifiants suivants :

   - Nom d'utilisateur : admin
   - Mot de passe : Patate123

3. Dans le menu latéral, allez dans **Data Explorer** pour vérifier que les données arrivent dans le bucket `mirror_data`.

## Configuration de Grafana

Grafana est l'outil qui vous permettra de visualiser les données stockées dans InfluxDB de manière élégante.

1. Ouvrez Grafana dans votre navigateur à l'adresse `http://<IP-DU-SERVEUR>:3001`
2. Connectez-vous avec les identifiants suivants :
   - Nom d'utilisateur : admin
   - Mot de passe : Patate123

### Ajout de la source de données InfluxDB

1. Dans le menu latéral, cliquez sur l'icône ⚙️ (Configuration) puis **Data sources**
2. Cliquez sur **Add data source**
3. Sélectionnez **InfluxDB**
4. Configurez la source de données comme suit :
   - Name : InfluxDB
   - URL : http://influxdb:8086
   - Accès : Server (default)
   - Auth : désactivé
   - Version InfluxDB : Flux
   - Organisation : MagicMirror
   - Token : my-super-secret-auth-token
   - Default Bucket : mirror_data
5. Cliquez sur **Save & Test** pour vérifier la connexion

### Création d'un tableau de bord

1. Dans le menu latéral, cliquez sur **+ Create** > **Dashboard**
2. Cliquez sur **Add new panel**
3. Configurez le panneau pour afficher les données système :
   - Dans l'éditeur de requête, choisissez **InfluxDB** comme source de données
   - Sélectionnez **Flux** comme langage de requête
   - Entrez la requête suivante :

```flux
from(bucket: "mirror_data")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "system")
  |> filter(fn: (r) => r._field == "cpu_temp")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
  |> yield(name: "mean")
```

4. Dans l'onglet **Panel options**, donnez un titre au panneau, par exemple "Température CPU"
5. Cliquez sur **Apply** pour ajouter le panneau au tableau de bord

6. Répétez les étapes 2 à 5 pour créer d'autres panneaux, par exemple :

   - Utilisation de la mémoire (remplacez `cpu_temp` par `memory_used_percent` dans la requête)
   - Temps de fonctionnement (remplacez `cpu_temp` par `uptime` dans la requête)

7. N'oubliez pas de sauvegarder votre tableau de bord en cliquant sur l'icône de disquette en haut à droite et en lui donnant un nom comme "MagicMirror System Dashboard"

## Intégration avec MQTT

Pour que cette infrastructure reçoive des données du MagicMirror, vous avez besoin d'un script qui envoie des données système au broker MQTT. Voici un exemple de script pour le Raspberry Pi :

```javascript
// mqtt-test-client.js
const mqtt = require("mqtt");
const os = require("os");
const fs = require("fs");

// Configuration MQTT
const mqttConfig = {
  server: "mqtts://mirrormqtt.jeremielapointe.ca",
  port: 8883,
  username: "MirrorMQTT",
  password: "Patate123",
  clientId: "mqtx_2de14cc3",
  topic: "magicmirror/system",
};

// Options de connexion MQTT
const options = {
  port: mqttConfig.port,
  clientId: mqttConfig.clientId,
  username: mqttConfig.username,
  password: mqttConfig.password,
  protocol: "mqtts",
  rejectUnauthorized: false, // Pour le développement uniquement
};

// Connexion au serveur MQTT
console.log(`Connecting to MQTT server: ${mqttConfig.server}`);
const client = mqtt.connect(mqttConfig.server, options);

client.on("connect", function () {
  console.log("Connected to MQTT server");
  sendSystemData();

  // Envoyer des données toutes les 60 secondes
  setInterval(sendSystemData, 60000);
});

client.on("error", function (error) {
  console.error("MQTT Error:", error);
});

// Fonction pour envoyer les données système
function sendSystemData() {
  try {
    // Collecter les données système
    const systemData = {
      timestamp: new Date().toISOString(),
      hostname: os.hostname(),
      platform: os.platform(),
      arch: os.arch(),
      uptime: os.uptime(),
      memory: {
        total: os.totalmem(),
        free: os.freemem(),
        usage: Math.round((1 - os.freemem() / os.totalmem()) * 100),
      },
      loadavg: os.loadavg(),
    };

    // Lire la température CPU sur Raspberry Pi
    if (fs.existsSync("/sys/class/thermal/thermal_zone0/temp")) {
      const temp = fs
        .readFileSync("/sys/class/thermal/thermal_zone0/temp", "utf8")
        .trim();
      systemData.cpuTemperature = parseFloat(temp) / 1000;
    }

    // Publier les données sur MQTT
    const message = JSON.stringify(systemData);
    client.publish(
      mqttConfig.topic,
      message,
      { qos: 0, retain: false },
      (error) => {
        if (error) {
          console.error("Publish error:", error);
        } else {
          console.log(`Data sent to topic: ${mqttConfig.topic}`);
        }
      }
    );
  } catch (e) {
    console.error("Error sending system data:", e);
  }
}

// Gestion de la déconnexion
process.on("SIGINT", function () {
  console.log("Disconnecting from MQTT...");
  client.end(true, () => {
    console.log("Disconnected from MQTT");
    process.exit(0);
  });
});
```

Pour exécuter ce script sur votre Raspberry Pi :

1. Installez les dépendances nécessaires :

```bash
npm install mqtt --save
```

2. Créez un fichier `mqtt-test-client.js` avec le contenu ci-dessus
3. Exécutez le script :

```bash
node mqtt-test-client.js
```

Vous pouvez également l'ajouter à votre script de démarrage `start.sh` pour qu'il s'exécute automatiquement lors du démarrage de MagicMirror.

## Dépannage

### Problèmes de connexion MQTT

Si vous rencontrez des problèmes de connexion MQTT :

1. Vérifiez que les identifiants MQTT sont corrects
2. Assurez-vous que le port 8883 est ouvert sur le broker MQTT
3. Vérifiez les logs Node-RED pour voir les erreurs de connexion
4. Essayez de tester la connexion avec un client MQTT comme `mosquitto_pub/sub`

### Pas de données dans InfluxDB

Si aucune donnée n'apparaît dans Grafana :

1. Vérifiez que Node-RED reçoit des données depuis MQTT (utilisez le nœud Debug)
2. Vérifiez que le nœud InfluxDB est correctement configuré
3. Vérifiez les logs du conteneur InfluxDB : `docker logs influxdb`
4. Assurez-vous que la requête Flux dans Grafana est correcte

### Problèmes avec Grafana

Si le tableau de bord ne s'affiche pas correctement :

1. Vérifiez que la source de données InfluxDB est correctement configurée
2. Vérifiez que les requêtes dans les panneaux pointent vers les bons champs et mesures
3. Ajustez l'intervalle de temps en haut à droite de Grafana pour voir différentes périodes

---

Ce guide d'installation et de configuration devrait vous permettre de mettre en place une infrastructure IoT complète pour surveiller votre MagicMirror. Avec ces outils, vous pourrez collecter, stocker et visualiser diverses métriques système et, potentiellement, étendre cette infrastructure pour surveiller d'autres aspects de votre environnement domestique.
