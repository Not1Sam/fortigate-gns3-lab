# Demo Script — Superviseur (8 août 2026)

## Commandes utiles

**Afficher l'IP de l'appServer et le traceroute :**
```
docker exec GNS3.Traffic-Gen-1.d311a72f-2416-4426-9138-96ccd23fe8fd sh -c "echo '--- IP appServer ---' && ip route get 192.168.10.103 && echo '--- API Status ---' && curl -s http://192.168.10.103:5000/api/status && echo '' && echo '--- Traceroute vers appServer ---' && traceroute -n -m 3 192.168.10.103"
```

**Lancer les tests automatisés :**
```
docker exec GNS3.Traffic-Gen-1.d311a72f-2416-4426-9138-96ccd23fe8fd sh /tmp/auto-tests.sh
```

## 1. Montrer la topologie (2 min)
- Ouvrir GNS3, afficher la topologie complète
- Pointer : 4 FortiGate VMs, 2 paires HA, Docker Router, OVS switches, clients
- Expliquer : la limite de 3 ports a forcé le design avec Docker Router

## 2. Vérifier tous les nodes (2 min)
- Dashboard GNS3 → tous les nodes en vert
- Montrer l'usage RAM/CPU

## 3. Tests automatisés — Traffic-Gen (5 min)
- Depuis le terminal de la machine hôte, lancer :
```
docker exec GNS3.Traffic-Gen-1.d311a72f-2416-4426-9138-96ccd23fe8fd sh /tmp/auto-tests.sh
```
- 18 tests couvrent tout le chapitre 7 du rapport :
  - **Connectivité** : ping WAN, cross-LAN, appServer, DNS
  - **Routage** : traceroute, Docker Router
  - **Services** : appServer HTTP, API, users DB, headers NAT
  - **Sécurité** : EICAR (AV), SQLi (IPS), XSS (IPS), Web Filter, DNS Filter, SSL
  - **Syslog** : envoi de logs vers FortiGate
- Expliquer les résultats :

| Badge | Signification |
|---|---|
| `[OK]` | Test réussi, service accessible |
| `[BLOCKED]` | FortiGate a bloqué le trafic (l'effet voulu) |
| `[WARNING]` | Pas détecté (signatures factory limitées) |
| `[INFO]` | Info technique |

| # | Test | Résultat | Signification |
|---|---|---|---|
| 1.1 | Ping WAN | OK | Gateway LAN2 accessible |
| 1.2 | Ping Cross-LAN | OK | FGT-Primary via Docker Router |
| 1.3 | Ping appServer | OK | appServer depuis LAN2 |
| 1.4 | DNS externe | OK | Résolution DNS Internet |
| 2.1 | Traceroute | OK | 5 sauts vers 8.8.8.8 |
| 2.2 | Docker Router | OK | Routeur accessible |
| 2.3 | Routage cross-LAN | OK | Confirmé via appServer |
| 3.1 | HTTP appServer | OK | HTTP 200 |
| 3.2 | API status | OK | API active, DB connectée |
| 3.3 | Users endpoint | OK | 14 users en base |
| 3.4 | Inspect headers | OK | NAT vérifié |
| 4.1 | EICAR (AV) | OK | Endpoint accessible |
| 4.2 | SQL Injection | BLOCKED | IPS fonctionne |
| 4.3 | XSS | WARNING | Signatures factory limitées |
| 4.4 | Web Filter | BLOCKED | URL bloquée |
| 4.5 | DNS Filter | BLOCKED | Domaine bloqué |
| 4.6 | SSL Inspection | INFO | Non appliqué (limite 3 politiques) |
| 5.1 | Syslog | OK | Message envoyé |

## 4. Politiques firewall (3 min)
- Ouvrir le WebUI de FGT-Primary (`https://192.168.122.x`)
- Montrer les 3 politiques : LAN1→WAN, LAN2→WAN, cross-LAN
- Expliquer la limite de 3 politiques

## 5. Failover HA (5 min) — la partie impressionnante
- Ouvrir la console de FGT-Primary
- Lancer `execute shutdown`
- Observer FGT-Primary-HA prendre le relais (`get system ha status` sur le standby)
- Ping depuis PC1 continue avec une interruption brève (~3-5s)
- Redémarrer FGT-Primary, montrer qu'il revient en mode standby

## 6. Services Docker (3 min)
- Montrer l'appServer : ouvrir `http://192.168.10.103:5000` dans le navigateur
- Montrer les endpoints : `/users`, `/api/status`, `/eicar`, `/inspect`
- Montrer la page d'accueil avec les stats DB

## 7. Dashboard Grafana (2 min)
- Ouvrir Grafana à `192.168.20.11:3000`
- Montrer le tableau de bord de supervision

## 8. Profils de sécurité (2 min)
- Dans le WebUI de FGT-Primary → Security Profiles
- Montrer AV, IPS, Web Filter, DNS Filter configurés
- Expliquer : signatures factory uniquement, pas de FortiGuard

## 9. Conclusion (1 min)
- Résumer : 7/16 objectifs réalisés, 5 partiels, 4 non réalisables
- Mentionner les contraintes de licence comme principale limitation
- Montrer le rapport

---

**Durée totale : ~28 min.** Imprimer le rapport ou l'avoir sur USB. Lancer GNS3 avant l'arrivée du superviseur.
