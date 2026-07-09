---
title: "Architecture Matérielle: SPU & ASIC"
tags:
  - study/architecture
  - security/spu
  - lang/french
---

# ⚡ Accélération Matérielle: Les ASIC de sécurité (SPU)

L'avantage technologique majeur de FortiGate repose sur l'intégration de processeurs dédiés appelés **Security Processing Units (SPU)**. Ces puces électroniques conçues sur mesure (ASIC) déchargent le processeur général (CPU) des tâches réseau et de sécurité les plus lourdes, garantissant des débits élevés et une latence minimale.

---

## 🛠️ Types de Processeurs SPU

### 1. Network Processor (NP - ex: NP6, NP7)
Le **Network Processor** opère directement au niveau de la couche réseau (L3/L4) et gère le flux de données à très haute vitesse (Fast Path).
*   **Fonction principale** : Accélérer le traitement des paquets réseau sans solliciter le CPU principal.
*   **Tâches déchargées** :
    *   Routage IPv4 et IPv6.
    *   Translation d'adresses réseau (NAT / PAT).
    *   Chiffrement et déchiffrement des tunnels VPN IPSec.
    *   Distribution de charge matérielle (Load Balancing).
*   **Impact** : Le trafic correspondant à des sessions déjà établies est directement traité par le NP (Fast Path), offrant une latence proche de zéro microseconde.

### 2. Content Processor (CP - ex: CP9, CP10)
Le **Content Processor** intervient au niveau applicatif (L7) pour accélérer le traitement du contenu des paquets.
*   **Fonction principale** : Fournir des services de cryptographie et d'inspection de contenu hors bande (co-processeur).
*   **Tâches déchargées** :
    *   Chiffrement/Déchiffrement SSL/TLS à haute vitesse (essentiel pour le *Deep SSL Inspection*).
    *   Analyse de signatures pour les moteurs de prévention des intrusions (IPS) et Antivirus.
    *   Compression et décompression de données à la volée.

### 3. System-on-a-Chip (SoC - ex: SoC4)
Le **SoC** intègre sur une seule et même puce électronique un CPU RISC standard, un processeur réseau (NP) et un processeur de contenu (CP).
*   **Cas d'utilisation** : Conçu spécifiquement pour les modèles d'entrée de gamme (*Entry-Level*, e.g., FortiGate 40F, 60F, 80F).
*   **Avantage** : Réduit le coût de fabrication et la consommation d'énergie tout en conservant d'excellentes performances d'accélération matérielle.

---

## 📊 Tableau Récapitulatif de la Répartition des Rôles

| Composant | Niveau OSI | Fonctions Clés | Impact sur les Performances |
| :--- | :--- | :--- | :--- |
| **CPU Général** | Couches 1 - 7 | Gestion du système (FortiOS), console, GUI, routage dynamique complexe, initialisation des sessions. | Goulot d'étranglement potentiel si trop de trafic non accéléré lui est envoyé. |
| **NP (Network)** | Couches 3 - 4 | Routage IPv4/IPv6, NAT, VPN IPSec, commutation de paquets. | Latence ultra-faible, débit proche de la vitesse de la ligne (wire-speed). |
| **CP (Content)** | Couches 4 - 7 | Déchiffrement SSL/TLS, analyse de signatures de virus/IPS, compression. | Évite l'effondrement des performances lors de l'inspection de flux chiffrés. |
| **SoC (System-on-Chip)** | Couches 1 - 7 | Consolidation (CPU + NP + CP) sur un seul composant pour modèles d'entrée de gamme. | Performance équilibrée pour les petits environnements (TPE/Agences). |