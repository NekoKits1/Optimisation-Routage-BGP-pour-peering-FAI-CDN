# Optimisation du Routage BGP pour le Peering FAI-CDN : Architecture Dual-Upstream

**[Consulter le rapport d'ingénierie complet (PDF)](./Rapport_Audrey_Kitio.pdf)**

## Présentation du Projet
Ce projet présente l'audit, la conception et l'implémentation d'une architecture réseau optimisée basée sur le protocole BGP. Il simule un environnement de télécommunications réel où un opérateur (AS 37462) est interconnecté à Internet via deux liaisons physiques asymétriques : 
* Un lien principal à haut débit (AS 328594 - 600 Mbps).
* Un lien de secours (AS 15964 - 155 Mbps).

L'objectif de cette étude est de corriger les défaillances structurelles d'une topologie initiale (routage asymétrique, temps de convergence élevé, congestion lors du basculement) en déployant des stratégies avancées d'ingénierie de trafic BGP (Inbound et Outbound Traffic Engineering).

---

## Topologie et Architecture (Simulation sous EVE-NG)

![Topologie de l'architecture sous EVE-NG](./topologie_EVE-NG.png)

L'architecture s'articule autour de :
* **Routeurs de cœur de réseau :** ASR-CORE1 et ASR-CORE3 assurant les annonces BGP externes (eBGP) et internes (iBGP).
* **Réseau de distribution et d'accès :** Commutateurs de distribution et routeurs de passerelle (R-VIP, R-BRAS) segmentés par VLANs.
* **Simulateurs FAI et CDN :** Routeurs interconnectés représentant l'Internet mondial et les réseaux de diffusion de contenu (CDN) majeurs.

---

## Problématiques Initiales Identifiées (Baseline)
L'audit de l'architecture par défaut a mis en évidence plusieurs défaillances critiques :
* **Défaut de propagation BGP :** Blocage des annonces des préfixes fragmentés de l'entreprise sur Internet.
* **Next-Hop iBGP Inaccessible :** Neutralisation de la redondance interne suite à l'impossibilité de joindre les adresses de saut suivant entre les routeurs de cœur.
* **Temps de convergence critique :** Temps de basculement avoisinant les 170 secondes, provoquant des pertes de paquets massives lors de la coupure de la liaison principale.
* **Saturation du lien de secours :** Absence de politique de Qualité de Service (QoS), entraînant la dégradation globale des flux standards et VIP lors d'un failover.
* **Routage Sortant Asymétrique :** Acheminement par défaut des téléchargements massifs vers le lien de secours, laissant le lien principal sous-utilisé.
* **Routage Entrant Aléatoire :** Perception topologique identique de l'opérateur depuis l'extérieur, induisant un risque d'engorgement du lien de secours par le trafic descendant.

---

## Solutions et Implémentations Techniques

### Phase 1 : Fiabilisation de l'Infrastructure de Base
* **Accessibilité des Préfixes (Route Null0) :** Création d'une route statique pointant vers l'interface virtuelle Null0 pour forcer l'annonce du bloc global de l'entreprise, préservant ainsi le découpage interne.
* **Correction du Routage Interne :** Application de la commande `next-hop-self` sur les sessions iBGP pour assurer la résolution correcte des chemins en interne.
* **Optimisation de la Convergence :** Déploiement du protocole BFD (Bidirectional Forwarding Detection) couplé aux sessions BGP et ajustement des minuteurs (Keepalive à 10s, Hold Timer à 30s) pour une détection de panne en moins de 300 ms.
* **Qualité de Service Hiérarchique (HQoS) :** Implémentation d'une politique de lissage de débit (Traffic Shaping à 155 Mbps) couplée à une file d'attente à faible latence (LLQ) pour garantir une bande passante dédiée aux flux VIP.

### Phase 2 : Ingénierie de Trafic BGP (Peering CDN)
* **Ingénierie Sortante (Outbound TE) via Local Preference :** Utilisation de Route-Maps et de Prefix-Lists pour identifier les réseaux CDN majeurs et forcer l'application d'un attribut `Local Preference` élevé (300). Cette politique contraint le trafic lourd à emprunter systématiquement la liaison principale.
* **Ingénierie Entrante (Inbound TE) via AS-Path Prepending :** Allongement artificiel du chemin BGP sur les annonces envoyées au lien de secours. Les routeurs de l'Internet mondial dévient ainsi le trafic descendant vers le lien principal, perçu comme topologiquement plus court.
* **Exceptions de Proximité Géographique :** Déploiement de règles de contournement ciblées pour les serveurs locaux, éliminant les détours mondiaux et minimisant la latence RTT.

---

## Résultats et Performances
L'implémentation de cette ingénierie de trafic a permis d'atteindre l'ensemble des objectifs fixés :
* **Haute Disponibilité (Failover) :** Réduction du temps d'interruption de près de 170 secondes à moins de 30 secondes lors d'une coupure du lien principal.
* **Latence (RTT) :** Latence moyenne vers les CDN mondiaux stabilisée à environ 51 ms lors des tests de stress.
* **Garantie de Service VIP :** Maintien d'une bande passante garantie et continuité de service sans faille pour les abonnés VIP lors d'événements de congestion.

---

## Instructions de Simulation
1. Disposer d'un émulateur réseau (EVE-NG ou GNS3) avec des images Cisco IOS compatibles.
2. Reproduire l'architecture physique conformément à l'image de topologie.
3. Importer les configurations vers les routeurs correspondants.
4. Vérifier la convergence via la commande `show ip bgp summary` et tester les mécanismes de failover en simulant une coupure d'interface sur l'un des liens externes.
