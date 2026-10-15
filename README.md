# 🛡️ Mini-SOC : Wazuh SIEM & Endpoint Monitoring Lab

Ce projet documente la mise en place d'un **SOC (Security Operations Center)** miniature en environnement de laboratoire. L'objectif est d'assurer la collecte centralisée de logs, la surveillance en temps réel d'endpoints (**Linux** & **Windows**) et la détection d'événements de sécurité suspects (ex: attaques par force brute) à l'aide de la suite **Wazuh (SIEM & XDR)**.

## 📐 Architecture de la Solution

```
┌────────────────────────────────────────────────────────────────────────┐
│                          KALI LINUX (Hôte)                             │
│  IP : 172.20.10.7                                                      │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    Docker Single-Node Stack                      │  │
│  │  - Wazuh Manager   (Port 1514/1515 TCP, 514 UDP)                 │  │
│  │  - Wazuh Indexer   (Port 9200 TCP)                             │  │
│  │  - Wazuh Dashboard (Port 443/5601 TCP)                           │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                  ▲                                     │
│  ┌───────────────────────────────┴──────────────────────────────┐  │
│  │ Local Wazuh Agent (ID: 001 - "kali")                         │  │
│  └──────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────▲─────────────────────────────────────┘
                                   │
                                   │ Logs (1514/TCP)
                                   │ Registration (1515/TCP)
                                   │
┌──────────────────────────────────┴─────────────────────────────────────┐
│                          WINDOWS 10 (VM)                               │
│  IP : Dynamic / Nat                                                    │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Remote Wazuh Agent (ID: 002 - "Windows10")                       │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
```

### Composants utilisés :
* **Serveur SIEM** : Wazuh 4.7.2 déployé via Docker Compose sur Kali Linux.
* **Agent Hôte (Linux)** : Agent local Wazuh installé sur Kali (ID `001`).
* **Agent Distant (Windows)** : Agent MSI Wazuh sur Windows 10 Enterprise/Education (ID `002`).

---

## 🚀 Étape 1 : Déploiement de la Stack Wazuh Manager (Docker)

Le serveur Wazuh a été déployé en mode *single-node* à l'aide des conteneurs Docker officiels :

```bash
cd ~/Mini-Soc/wazuh-docker/single-node
sudo docker compose up -d
```

Vérification de l'état des conteneurs :
```bash
sudo docker compose ps
```
*Résultat : `wazuh.manager`, `wazuh.indexer` et `wazuh.dashboard` sont tous à l'état `Up`.*

---

## 🔑 Étape 2 : Enregistrement et Authentification des Agents

L'enregistrement des agents s'est appuyé sur l'utilitaire `manage_agents` du Manager pour générer les clés de chiffrement symétrique et assurer la liaison sécurisée via le port `1514/TCP`.

### 1. Agent Kali Linux (`ID 001`)
* Enregistrement de l'agent local dans le Manager Wazuh.
* Configuration du serveur cible (`172.20.10.7`) dans `/var/ossec/etc/ossec.conf`.
* Démarrage du service : `sudo systemctl restart wazuh-agent`.

### 2. Agent Windows 10 (`ID 002`)
* Export de la clé encodée en Base64 depuis le Manager :
  ```bash
  sudo docker exec -it single-node-wazuh.manager-1 /var/ossec/bin/manage_agents
  ```
* Importation de la clé sur Windows via PowerShell Administrateur :
  ```powershell
  & 'C:\Program Files (x86)\ossec-agent\manage_agents.exe' -i "MDAyIFdpbmRvd3MxMCBhbnkgN2M0NzI5ZDE5YmE3MjIxYzc0NDQyMzU4MDI4NjdmN2I5MDIzNDgyM2FlYzIyODIxNzY0ZTFlOTBkOTc4Yzk5Yw=="
  ```
* Configuration de l'adresse IP du serveur Manager (`172.20.10.7`) dans `C:\Program Files (x86)\ossec-agent\ossec.conf`.
* Redémarrage du service : `Restart-Service Wazuh`.

---

## 📊 Étape 3 : Validation de la Connectivité des Agents

Une fois la poignée de main cryptographique établie, le Dashboard Wazuh confirme l'état opérationnel et la bonne réception des signaux *keepalive*.

### 📸 Vue d'ensemble des Agents Connectés

![Wazuh Agents Active](data:image/png;base64,iVBORw0KGgoAAAANSU6AAA)
*Figure 1 : Tableau de bord Wazuh affichant 100% d'agents actifs (`kali` ID 001 et `Windows10` ID 002).*

---

## ⚡ Étape 4 : Simulation d'Attaque & Détection en Temps Réel

Afin de valider la capacité du SIEM à corréler et lever des alertes sur des événements de sécurité, nous avons simulé une attaque par force brute / échec d'authentification sur la machine Windows 10.

### Simulation sur Windows 10 (PowerShell) :
```powershell
1..5 | ForEach-Object { net use \\localhost /user:fakeuser fakepassword }
```

### 📸 Remontée des Alertes dans le Dashboard (Security Events)

![Wazuh Security Events](data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQABAAD)
*Figure 2 : Ingestion immédiate des événements. On observe la levée de 5 alertes de niveau 5 pour la règle `60122` (Logon failure - Unknown user or bad password) ainsi qu'une alerte de changement de port réseau sur Kali (Règle `533`).*

---

## 🛠️ Résolution des Problèmes Encountered (Troubleshooting)

1. **Changement d'adresse IP dynamique (DHCP/Hotspot)** :
   * *Problème* : L'agent Windows n'arrivait plus à joindre le serveur (`ERROR: Unable to connect / Connection refused`) suite au passage de Kali de `172.20.10.3` à `172.20.10.7`.
   * *Solution* : Mise à jour à chaud de la balise `<address>` dans le fichier `ossec.conf` sur Windows et ré-importation de la clé symétrique via `manage_agents.exe`.

2. **Initialisation des démons internes Docker** :
   * *Problème* : Port `1514` fermé immédiatement après le démarrage du conteneur Manager.
   * *Solution* : Vérification via `wazuh-control status` pour s'assurer du démarrage complet de `wazuh-remoted` avant l'initiation de la connexion agent.

---

## 🎯 Compétences Démontrées

* **SIEM & XDR** : Administration et supervision Wazuh.
* **Containers** : Déploiement et gestion de microservices avec Docker Compose.
* **Windows / Linux Security** : Analyse des Event Logs Windows (Event ID 4625), gestion des deamons Linux (`systemd`).
* **Réseau & Sécurité** : Analyse de flux TCP/UDP (ports 1514/1515), dépannage de connectivité en environnement virtuel.