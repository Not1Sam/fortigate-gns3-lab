---
title: "Profils UTM & Modes d'Inspection"
tags:
  - study/utm
  - security/profiles
  - lang/french
---

# 🛡️ Profils UTM & Modes d'Inspection

## 🔬 Modes d'Inspection: Flow-based vs. Proxy-based

### 1. Flow-based (Mode Flux)
*   **Fonctionnement**: Analyse les paquets à la volée lorsqu'ils traversent l'équipement sans mettre le fichier entier en cache.
*   **Avantages**: Latence ultra-faible, débit maximum.
*   **Inconvénients**: Moins précis sur les archives compressées complexes ou imbriquées.

### 2. Proxy-based (Mode Proxy)
*   **Fonctionnement**: Intercepte la connexion. Le FortiGate agit comme destinataire temporaire, télécharge l'intégralité du fichier dans sa RAM, l'analyse, puis le transmet au client final.
*   **Avantages**: Sécurité maximale (bloque le fichier malveillant avant le premier octet transmis).
*   **Inconvénients**: Latence plus élevée, consommation accrue de mémoire vive (RAM).

---

## 🔑 Déchiffrement SSL (SSL Inspection)

> [!important]
> Plus de 90% du trafic web utilise HTTPS. Sans déchiffrement SSL, les filtres Antivirus et IPS sont aveugles. C'est l'étape clé de l'inspection (voir le flux sur **[[Packet_Lifecycle|Packet Flow]]**).

*   **SSL Certificate Inspection (Léger)**: Lit uniquement le champ SNI (Server Name Indication) en clair pour bloquer ou autoriser les domaines. Ne déchiffre pas les données et ne sollicite pas l'ASIC CP.
*   **Deep Packet Inspection (DPI - Inspection Profonde)**: Le FortiGate agit en tant que proxy de déchiffrement Man-in-the-Middle (MitM) grâce à l'accélération du **[[ASIC_SPU_Architecture#2. Content Processor (CP - ex: CP9, CP10)|Content Processor CP]]**.
    *   *Requis:* Déployer le certificat CA racine du FortiGate sur tous les clients du LAN pour éviter les alertes de sécurité des navigateurs.\n