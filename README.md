# Optimisation du Routage BGP pour le Peering FAI-CDN : Architecture Dual-Upstream

Audit, conception et implémentation d'une architecture réseau basée sur BGP, simulée sous EVE-NG. Un opérateur fictif est interconnecté à Internet via deux liaisons physiques asymétriques, et l'objectif est de corriger les défaillances d'une topologie initiale (routage asymétrique, convergence lente, congestion au basculement) par des techniques avancées d'ingénierie de trafic BGP.

> Projet académique réalisé en stage ingénieur réseau. Les données ci-dessous sont génériques : numéros d'AS de la plage documentaire (RFC 5398), pas de nom de client ni de plan d'adressage réel.

---

## Topologie (simulation EVE-NG)

- **Opérateur** : AS 65001, deux liens upstream asymétriques
  - Lien principal haut débit : AS 65002 (600 Mbps)
  - Lien de secours : AS 65003 (155 Mbps)
- **Cœur de réseau** : deux routeurs (ASR-CORE1, ASR-CORE2) en redondance VRRP, annonces eBGP vers les upstreams et iBGP en interne
- **Distribution/accès** : commutateurs et routeurs de passerelle segmentés par VLAN
- **Simulateurs FAI/CDN** : routeurs représentant Internet et les réseaux de diffusion de contenu


---

## Problèmes identifiés (baseline)

- Blocage des annonces de préfixes fragmentés vers Internet
- Next-hop iBGP inaccessible, neutralisant la redondance interne
- Convergence critique : ~170 s de basculement, pertes de paquets massives
- Absence de QoS : saturation du lien de secours lors d'un failover
- Routage sortant asymétrique : le lien principal reste sous-utilisé
- Routage entrant aléatoire : risque d'engorgement du lien de secours

---

## Solutions implémentées

**Phase 1 — Fiabilisation de l'infrastructure**
- Route statique vers `Null0` pour forcer l'annonce du bloc global sans perdre le découpage interne
- `next-hop-self` sur les sessions iBGP
- BFD couplé à BGP, timers ajustés (Keepalive 10 s / Hold 30 s) → détection de panne < 300 ms
- QoS hiérarchique (shaping + LLQ) pour garantir la bande passante des flux prioritaires

**Phase 2 — Ingénierie de trafic BGP**
- **Sortant (Local Preference)** : route-maps et prefix-lists pour identifier les réseaux CDN majeurs et forcer une Local Preference élevée sur le lien principal
- **Entrant (AS-Path Prepending)** : allongement du chemin BGP annoncé sur le lien de secours, pour dévier le trafic descendant vers le lien principal
- Règles de contournement pour les serveurs à proximité géographique, afin de minimiser la latence

---

## Résultats

- **Failover** : réduction du temps d'interruption de ~170 s à moins de 30 s
- **Latence (RTT)** : stabilisée autour de 51 ms en test de stress (environnement simulé)
- **Service prioritaire** : bande passante garantie, continuité de service sans faille lors de la congestion

---

## Reproduire la simulation

1. Disposer d'un émulateur réseau (EVE-NG ou GNS3) avec images Cisco IOS compatibles
2. Reconstituer la topologie
3. Importer les configurations (`config_BGP.txt`) vers les routeurs correspondants
4. Vérifier la convergence : `show ip bgp summary`, puis simuler une coupure d'interface pour tester le failover

---

## Ce que ce projet m'a appris

- Diagnostiquer une architecture BGP à partir de symptômes observables (pertes, asymétrie, convergence lente)
- La différence entre ingénierie de trafic entrante et sortante, et quand utiliser Local Preference plutôt qu'AS-Path Prepending
- L'importance de `next-hop-self` et du conditionnement par `track` pour éviter les trous noirs de routage
- Que les communities BGP opèrent à la granularité du préfixe, pas de l'adresse individuelle
