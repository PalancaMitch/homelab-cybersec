# 🛡️ Homelab Cybersécurité – Honeypot, Supervision & Accès Distant

> Déploiement d'une infrastructure de cybersécurité sur hyperviseur Hyper-V, combinant détection d'intrusion, collecte de logs centralisée et accès distant sécurisé.

---

## 📐 Architecture générale

```
┌─────────────────────────────────────────────────────┐
│                  Hyper-V (Windows)                  │
│                                                     │
│  ┌─────────────────┐   ┌──────────────────────────┐ │
│  │   VM Honeypot   │   │     VM Grafana / Loki    │ │
│  │                 │──▶│                          │ │
│  │ OpenCanary      │   │  Grafana  (dashboards)   │ │
│  │ Apache (leurre) │   │  Loki     (agrégation)   │ │
│  │ Nagios          │   │  Promtail (collecte)     │ │
│  └─────────────────┘   └──────────────────────────┘ │
│                                                     │
│  ┌──────────────────────────────────────────────┐   │
│  │           VM Apache Guacamole                │   │
│  │                                              │   │
│  │  Guacamole (portail web RDP/SSH/VNC)         │   │
│  │  Tomcat 9 + guacd                            │   │
│  │  MySQL (authentification)                    │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
         │
         ▼
   Alertes Discord (webhook)
```

---

## 🎯 Objectifs du projet

- Simuler un environnement exposé et détecter des comportements malveillants (scans, tentatives de connexion, requêtes HTTP suspectes)
- Centraliser et visualiser les logs en temps réel via une stack Grafana + Loki
- Sécuriser l'accès aux VMs via un portail d'accès distant authentifié (Guacamole)
- Mettre en place des alertes automatiques sur événements de sécurité

---

## 🖥️ Stack technique

| Composant | Rôle | Version |
|---|---|---|
| Hyper-V | Hyperviseur | Windows Server / Win 10-11 |
| OpenCanary | Honeypot multi-protocoles | 0.6.x |
| Apache HTTP | Service leurre (honeypot web) | 2.4 |
| Nagios | Supervision d'infrastructure | 4.x |
| Grafana | Visualisation des logs et métriques | 10.x |
| Loki | Agrégateur de logs | 3.x |
| Promtail | Agent de collecte de logs | 3.x |
| Apache Guacamole | Portail d'accès distant (RDP/SSH/VNC) | 1.5.x |
| Tomcat | Serveur d'application Java (Guacamole) | 9.x |
| MySQL | Base de données (auth Guacamole) | 8.x |

---

## ⚙️ Déploiement

### Prérequis
- Hyper-V activé sur Windows 10/11 Pro ou Windows Server
- Au moins 8 Go de RAM disponibles pour les 3 VMs
- Réseau interne Hyper-V configuré avec NAT pour l'accès outbound

### VM 1 – Honeypot (OpenCanary + Apache + Nagios)

```bash
# Installation OpenCanary
pip install opencanary

# Générer la config par défaut
opencanaryd --copyconfig

# Lancer le service
opencanaryd --start
```

> **Note :** La règle NAT Windows suivante est nécessaire pour que la VM ait accès à Internet (webhook Discord) :
> ```powershell
> New-NetNat -Name "HyperV-NAT" -InternalIPInterfaceAddressPrefix "192.168.x.0/24"
> ```

### VM 2 – Stack Grafana + Loki

```bash
# Lancer la stack via Docker Compose
docker compose up -d
```

> Voir [`grafana-loki/docker-compose.yml`](./grafana-loki/docker-compose.yml) pour la configuration complète.

**Point d'attention Loki v3 :** la syntaxe de configuration a changé par rapport aux versions antérieures. Le champ `schema_config` doit utiliser `tsdb` comme store :

```yaml
schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h
```

### VM 3 – Apache Guacamole

```bash
# Initialiser le schéma MySQL
cat initdb.sql | docker exec -i mysql_container mysql -u root -p guacamole_db

# Lancer Guacamole
docker compose up -d
```

> Accès au portail : `http://[IP-VM]:8080/guacamole`  
> Authentification MySQL activée (remplacement du provider par défaut).

---

## 📊 Dashboards Grafana

Le dossier [`grafana-loki/dashboards/`](./grafana-loki/dashboards/) contient les exports JSON des dashboards :

- **Honeypot Activity** : visualisation des tentatives de connexion par protocole (SSH, HTTP, FTP…)
- **System Logs** : logs système agrégés depuis les 3 VMs
- **Nagios Alerts** : état des services supervisés

Pour les importer : *Grafana → Dashboards → Import → Upload JSON*

---

## 🔔 Alertes Discord

OpenCanary est configuré pour envoyer des alertes en temps réel via webhook Discord à chaque événement détecté (scan de port, tentative SSH, requête HTTP suspecte).

```json
{
  "handler": "SocketHandler",
  "host": "127.0.0.1",
  "port": 1514
}
```

> ⚠️ **Note de sécurité :** en environnement de production, les webhooks Discord ne sont pas recommandés comme canal d'alerte principal (pas d'authentification mutuelle, dépendance à un service tiers). Préférer un SIEM ou une solution d'alerting interne.

---

## 🧠 Problèmes résolus / apprentissages

| Problème | Solution |
|---|---|
| Loki v3 : erreur de configuration sur le champ `schema_config` | Migration vers le store `tsdb` et schéma `v13` |
| VMs Hyper-V sans accès Internet (webhook Discord bloqué) | Création d'une règle NAT Windows via `New-NetNat` |
| Corruption Unicode dans les terminaux SSH Guacamole | Forçage de l'encodage UTF-8 côté connexion Guacamole |
| Guacamole : authentification par défaut insuffisante | Remplacement par l'extension `guacamole-auth-jdbc-mysql` |

---

## 🔒 Sécurité & avertissement

Ce projet est réalisé dans un environnement isolé à des fins **éducatives uniquement**.  
Aucune adresse IP, credential ou information d'infrastructure réelle n'est publiée dans ce dépôt.  
Les fichiers de configuration utilisent des variables d'environnement pour les valeurs sensibles.

---

## 📁 Structure du dépôt

```
homelab-cybersec/
├── README.md
├── honeypot/
│   ├── opencanary.conf.example
│   └── nagios/
│       └── services.cfg.example
├── grafana-loki/
│   ├── docker-compose.yml
│   ├── loki-config.yml
│   ├── promtail-config.yml
│   └── dashboards/
│       ├── honeypot-activity.json
│       └── system-logs.json
└── guacamole/
    ├── docker-compose.yml
    └── initdb.sql.example
```

---

## 👤 Auteur

**Glodi Kisoka** – Étudiant BUT Réseaux & Télécommunications (cybersécurité), IUT de Saint-Malo  
Certifié Stormshield Network Security Administrator (CSNA)  

---

## 📄 Licence

Ce projet est publié sous licence [MIT](LICENSE) – libre d'utilisation à des fins éducatives.
