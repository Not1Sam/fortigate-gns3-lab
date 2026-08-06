# FortiGate GNS3 Lab

Laboratoire de sécurité réseau complet avec FortiGate, GNS3 et Docker.

## Contenu

- **03_GNS3_Labs/** — Guides de configuration, topologie, références nodes
- **Rapport-de-Stage/** — Rapport de stage LaTeX (compile avec `xelatex`)
- **auto-tests.sh** — Script de tests automatisés pour le container Traffic-Gen
- **GNS3-Images/** — Images QEMU pour FortiGate (à télécharger séparément)

## Prérequis

- [GNS3](https://www.gns3.com/) (GUI + Server)
- QEMU/KVM
- Docker
- XeLaTeX (pour compiler le rapport)

## Installation des images FortiGate

Les images QCOW2 sont trop volumineuses pour GitHub (>100MB). Téléchargez-les manuellement :

1. **FortiGate v7.4.12** (eval license) — `fgt-v7.4.12.qcow2`
   - Disponible sur le site Fortinet ou via votre compte d'évaluation

2. **FortiGate v7.0.9** (pré-licencié) — `fortios.qcow2`
   - Image pré-configurée avec licence

Placez les fichiers dans `GNS3-Images/` ou importez-les directement dans GNS3 via Edit > Preferences > QEMU > Qemu Templates.

## Démarrage rapide

1. Ouvrir le projet GNS3 (`d311a72f-2416-4426-9138-96ccd23fe8fd`)
2. Démarrer tous les nodes
3. Lancer les tests automatisés :
   ```bash
   docker exec GNS3.Traffic-Gen-1.<uuid> sh /tmp/auto-tests.sh
   ```

## Compiler le rapport

```bash
cd Rapport-de-Stage/
xelatex -interaction=nonstopmode main.tex
xelatex -interaction=nonstopmode main.tex  # 2e passe pour les références
```

## Topologie

```
                    ┌─────────────┐
                    │   NAT1      │
                    │  (virbr0)   │
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │   Switch1   │
                    │  (WAN)      │
                    └──┬───┬───┬──┘
                       │   │   │
              ┌────────┘   │   └────────┐
              │            │            │
        ┌─────┴─────┐ ┌───┴────┐ ┌─────┴─────┐
        │FGT-Primary│ │FGT-Sec │ │FGT-Pri-HA │
        │  (LAN1)   │ │ (LAN2) │ │  (STBY)   │
        └─────┬─────┘ └───┬────┘ └───────────┘
              │            │
        ┌─────┴─────┐ ┌───┴────┐
        │ OVS-LAN1  │ │OVS-LAN2│
        └──┬──┬──┬──┘ └──┬──┬──┘
           │  │  │       │  │
          PC1 │  │      Ubuntu
              │  │       │
         appServer │   Grafana
              │    │   Prometheus
         PostgreSQL │  Traffic-Gen
                    │
              ┌─────┴─────┐
              │Docker Router│
              │ (cross-LAN) │
              └─────────────┘
```

## Licence

Projet de stage — EMSI Rabat, 2026
