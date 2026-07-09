---
title: "Cycle de vie d'un paquet dans le pare-feu"
tags:
  - study/architecture
  - network/lifecycle
  - lang/french
---

# 🔄 Cycle de vie d'un paquet (FortiOS Packet Flow)

Comprendre le cheminement exact d'un paquet à travers FortiOS est indispensable pour configurer correctement le pare-feu et diagnostiquer les problèmes de connectivité ou de blocage de trafic.

---

## 🗺️ Schéma du Flux de Traitement (Packet Flow)

```mermaid
graph TD
    %% Nodes
    Ingress["1. Ingress (Entrée Interface)"]
    SessionLookup{"2. Session Lookup"}
    FastPath["NP Fast Path (Accélération L3/L4)"]
    SlowPath["3. Initialisation de Session (CPU)"]
    RouteLookup["Recherche de Route de Sortie"]
    PolicyEval{"4. Évaluation des Politiques"}
    Drop["Trafic Rejeté (Implicit Deny)"]
    UTMEval{"5. Profils UTM actifs ?"}
    SSLInspect["Déchiffrement SSL/TLS (ASIC CP)"]
    UTMEngine["Moteurs UTM (IPS, AV, WebFilter)"]
    Egress["6. Egress (Sortie Interface)"]

    %% Flow links
    Ingress --> SessionLookup
    SessionLookup -->|Session existante| FastPath
    SessionLookup -->|Nouvelle session| SlowPath
    SlowPath --> RouteLookup
    RouteLookup --> PolicyEval
    PolicyEval -->|Rejet (Deny)| Drop
    PolicyEval -->|Autorisation (Accept)| UTMEval
    UTMEval -->|Non| Egress
    UTMEval -->|Oui| SSLInspect
    SSLInspect --> UTMEngine
    UTMEngine --> Egress
    FastPath --> Egress

    %% Styling
    style Ingress fill:#1b3a4b,stroke:#00a8cc,stroke-width:2px,color:#fff
    style Egress fill:#1b3a4b,stroke:#00a8cc,stroke-width:2px,color:#fff
    style Drop fill:#4a121a,stroke:#e63946,stroke-width:2px,color:#fff
    style FastPath fill:#134021,stroke:#2a9d8f,stroke-width:2px,color:#fff
```

---

## 📝 Description Détaillée des Étapes

### 1. Ingress (Interface d'Entrée)
Le paquet physique arrive sur une interface réseau (port physique ou VLAN). FortiOS effectue des vérifications initiales au niveau de la couche 2 (Ethernet) :
*   Validation de la somme de contrôle (checksum).
*   Vérification des en-têtes IP et des options.
*   Si le paquet correspond à une session déjà déchargée au niveau du processeur réseau (NP), il bascule directement sur le **Fast Path** (voir [[ASIC_SPU_Architecture]]).

### 2. Session Lookup (Recherche de Session)
Le pare-feu interroge sa table d'états (state table) pour déterminer si le paquet appartient à une session TCP, UDP ou ICMP active :
*   **Session existante** : Le paquet évite l'évaluation des règles de sécurité et est traité directement via le Fast Path (**[[ASIC_SPU_Architecture#1. Network Processor (NP - ex: NP6, NP7)|NP Acceleration]]**).
*   **Nouvelle session** : Le paquet passe par le **Slow Path** (traitement CPU général) pour l'établissement de la session.

### 3. Route Lookup (Sélection de Route)
Pour une nouvelle session, FortiOS examine sa table de routage pour identifier l'interface de sortie appropriée pour le paquet de retour et le paquet aller. C'est également à cette étape que s'effectue la vérification de l'anti-spoofing (Reverse Path Forwarding - RPF).

### 4. Firewall Policy Evaluation (Évaluation des Politiques)
FortiOS parcourt la liste des politiques de sécurité de haut en bas (voir **[[FortiGate_Admin#🛡️ Firewall Policies & Stateful Inspection|Firewall Policies]]** pour la logique de filtrage) :
*   **Action DENY** : Le trafic est jeté immédiatement. Si aucune règle ne correspond, la règle par défaut **[[FortiGate_Admin#🛡️ Firewall Policies & Stateful Inspection|Implicit Deny]]** s'applique et bloque le trafic.
*   **Action ACCEPT** : La session est créée dans la table d'états, et FortiOS applique le NAT source ou destination si configuré.

### 5. Security Profiles / UTM (Inspection de Contenu L7)
Si la politique de sécurité intègre des profils de sécurité (**[[UTM_Inspection_Modes|Antivirus, Web Filtering, Application Control, IPS]]**) :
*   **Déchiffrement SSL (SSL Inspection)** : Si le flux est chiffré, le processeur **[[ASIC_SPU_Architecture#2. Content Processor (CP - ex: CP9, CP10)|CP]]** procède au déchiffrement temporaire du paquet pour que le moteur UTM puisse l'analyser (voir **[[UTM_Inspection_Modes#🔑 Déchiffrement SSL (SSL Inspection)|SSL Inspection]]**).
*   **Inspection de contenu** : Le trafic est soumis à l'analyse de signatures ou de catégories.
*   **Re-chiffrement** : Le paquet est re-chiffré avant d'être envoyé vers l'interface de sortie.

### 6. Egress (Interface de Sortie)
Le paquet est transmis à l'interface réseau de sortie. Si IPSec ou d'autres encapsulations sont appliquées, le paquet est préparé et encapsulé avant la transmission finale sur le média physique.