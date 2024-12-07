---
layout: post
categories: redteam
title: "Personnalisation d'outils et contournement d'antivirus"
permalink: "/:categories/:title/"
---

# Personnaliser un outil pour contourner les antivirus

<img src="../../assets/images/posts/redteam/mythic-custom-agent/cover.png" style="display:block;margin:auto;max-height:400px;width:auto">

# Introduction

De nos jours, de nombreux pentesters et opérateurs Red Team utilisent des outils publics (à défaut d'avoir des outils privés) dans le cadre de leur travail. Ces outils sont, pour la plupart, scrutés par les éditeurs de solutions antivirales qui publient régulièrement des signatures permettant à leur produit de détecter leur présence sur une machine.

Cet article a pour objectif d'introduire les lecteurs à la personnalisation de ces outils afin de limiter le risque d'être identifié durant un audit technique.

Dans un premier temps, nous présenterons rapidement les outils d'exploitation en red teaming (les C2 ou Command and Control).

Ensuite, nous verrons comment il est possible de contourner les détections d'un antivirus comme Windows Defender simplement en modifiant des chaînes de caractères qui se trouvent dans le code source d'un outil.

Pour illustrer les propos, nous modifierons un outil publique : l'agent **Apollo** pour le C2 Mythic. 

# Les C2 (Command and Control)

Historiquement, les opérateurs Red Team n'avaient que très peu d'outils à leur disposition pour mener à bien leur mission. Très vite, de nombreuses solutions publiques et privées ont vu le jour. 

Un C2 n'est rien d'autre qu'une suite logicielle constituée de deux éléments principaux :

* un serveur (server.exe) qui gère des clients et leur donne des ordres (commandes)
* un ou plusieurs clients (client.exe) qui reçoivent les ordres, les éxécutent et renvoient les résultats au serveur.

Le serveur est généralement installé sur une machine d'attaque et les clients sont installés sur les machines cibles compromises par les attaquants. Le serveur est accessible à travers une interface graphique qui offre la possibilité aux opérateurs de générer des agents (clients) et de gérer leurs opérations.

Voici un exemple montrant l'interface utilisateur de Mythic C2 :

<img src="../../assets/images/posts/redteam/mythic-custom-agent/apollo_final_2.png" style="display:block;margin:auto;max-height:400px;width:auto">

## C2 publics

Les C2 publics sont des outils développés et publiés gratuitement par des groupes d'individus ou des organisations. Ils sont généralement disponibles sur des dépôts publics comme Github.

Voici quelques exemples d'outils publics populaires :

* [Mythic](https://github.com/its-a-feature/Mythic)
* [Sliver](https://github.com/BishopFox/sliver)
* [Havoc](https://github.com/HavocFramework/Havoc)
* [Covenant](https://github.com/cobbr/Covenant)
* [Empire](https://github.com/BC-SECURITY/Empire)

## C2 privés/commerciaux

Les C2 privés sont généralement développés au sein des organisations matures et principalement pour permettre à leurs opérateurs/consultants de l'utiliser durant leurs missions. Les C2 commerciaux, sont quant à eux, développés par des entreprises désirant les commercialiser sous forme de licence à vendre aux autres entreprises et notamment les prestataires de cybersécurité.

Voici quelques outils commerciaux :

* [Cobalt Strike](https://www.cobaltstrike.com/)
* [Brute Ratel](https://bruteratel.com/)
* [Nighthawk](https://nighthawkc2.io/)

La différence principale entre les C2 publics et privés/commerciaux réside dans les fonctionnalités proposées ainsi que la fréquence des mises à jour.

Les outils commerciaux mettent en avant leur capacité à rester à jour faces aux moyens de détection des solutions antivirales même si dans la réalité on constate que c'est difficilement le cas.

En effet, à l'instar des attaquants, les éditeurs d'antivirus peuvent tout simplement acheter des licences et suivre les dernières fonctionnalités proposées pour implémenter des règles de détection. 

Une liste plus complète des C2 existants se trouve [ici](https://docs.google.com/spreadsheets/d/1b4mUxa6cDQuTV2BPC6aA-GR4zGZi0ooPYtBe4IgPsSc/edit?gid=0#gid=0).

# Personnalisation d'un agent de C2

Jusque là, on a seulement survolé le fonctionnement des C2 sans rentrer dans les détails car ce n'est pas l'objectif de l'article. 

Même si chaque C2 a ses spécificités, ces outils proposent généralement les mêmes fonctionnalités et notamment la possibilité de générer des clients personnalisés.

Pourquoi ? Comme on l'a vu plus haut, les éditeurs de solutions antivirales ont créé des bases de données de signatures (qui peuvent être publiques ou privées) correspondant aux malwares identifiés auparavant. Parmi ces malwares on retrouve entre autres, les clients de C2 publics ou privés.

Dans cette partie, nous allons voir comment personnaliser l'agent Apollo de l'outil open-source **Mythic C2**.

## Apollo

Apollo est un agent pour machines Windows écrit en C# et qui s'interface parfaitement avec le serveur Mythic C2. Il dispose de la plupart des fonctionnalités qu'on pourrait retrouver dans des C2 commerciaux et est régulièrement mis à jour par une communauté de développeurs.

Après avoir installé l'agent sur le serveur Mythic, nous allons générer une première version dite "nature" (ou "vanilla" en anglais) pour observer le comportement de l'antivirus Windows Defender. Cette version correspond au code source par défaut et qui est disponible publiquement [ici](https://github.com/MythicAgents/Apollo).

<img src="../../assets/images/posts/redteam/mythic-custom-agent/generate_payload_1.png" style="display:block;margin:auto;max-height:400px;width:auto">

Lorsque l'agent est exécuté sur une machine Windows avec Defender activé, une alerte est déclenchée et l'antivirus procède à une mise en quarantaine puis supprime le fichier.

<img src="../../assets/images/posts/redteam/mythic-custom-agent/generate_payload_3.png" style="display:block;margin:auto;max-height:400px;width:auto">

À l'aide de l'outil [ThreatCheck](https://github.com/rasta-mouse/ThreatCheck), il est possible d'identifier quelle portion du code contenu dans l'agent **apollo.exe** déclenche une alerte sur l'antivirus.

<img src="../../assets/images/posts/redteam/mythic-custom-agent/cover.png" style="display:block;margin:auto;max-height:400px;width:auto">

Généralement, les octets considérés comme malveillants se trouvent à la dernière ligne. Ici, on voit que la chaîne de caractères "Apollo.Peers.TCP" semble déclencher une alerte.

## Modification de l'agent Apollo

Mythic C2 est un ensemble de conteneurs Docker. Il offre la possiblilité de personnaliser chacune des briques qui le composent et notamment le code source des agents. Un article rédigé par Cody Thomas, le développeur à l'origine de l'outil, explique comment procéder : [https://medium.com/@its_a_feature_/agent-customization-in-mythic-tailoring-tools-for-red-team-needs-1746fd02177f](https://medium.com/@its_a_feature_/agent-customization-in-mythic-tailoring-tools-for-red-team-needs-1746fd02177f)

Lorsqu'on installe un agent dans Mythic, de nouvelles variables d'environnement sont créées dans le fichier .env.

<img src="../../assets/images/posts/redteam/mythic-custom-agent/mythic_conf.png" style="display:block;margin:auto;max-height:400px;width:auto">


Si un opérateur souhaite modifier le code source d'un agent pour générer des versions personnalisées à travers l'interface graphique, il doit d'abord éditer la configuration du C2 (fichier .env) avec les commandes suivantes :

```
sudo ./mythic-cli config set APOLLO_USE_BUILD_CONTEXT true
sudo ./mythic-cli config set APOLLO_USE_VOLUME false
```

Avant de toucher au code source de l'agent installé sur le C2, nous allons essayer de modifier le binaire généré (apollo.exe) directement en utilisant l'outil [dnspy](https://github.com/dnSpy/dnSpy) :

<img src="../../assets/images/posts/redteam/mythic-custom-agent/dnspy_0.png" style="display:block;margin:auto">

Comme nous l'avons vu plus haut, après un scan de l'agent Apollo avec ThreatCheck, nous avons pu identifier les "bad bytes" qui déclenchent une alerte. Il est possible de rechercher la portion exacte en saisissant les derniers octets dans l'éditeur hexadécimal de dnspy :

<img src="../../assets/images/posts/redteam/mythic-custom-agent/dnspy_1.png" style="display:block;margin:auto">

Ici il suffit simplement de renommer toutes les occurences de la chaîne de caractères "Apollo.Peers.TCP".

<img src="../../assets/images/posts/redteam/mythic-custom-agent/dnspy_3.png" style="display:block;margin:auto">

Un nouveau scan montre que l'outil est encore détecté par l'antivirus.

<img src="../../assets/images/posts/redteam/mythic-custom-agent/dnspy_4.png" style="display:block;margin:auto">

Il faut donc recommencer le même processus jusqu'à ce qu'il n'y ait plus aucune chaîne de caractères malveillante détectée. 

Après modification de la seconde chaîne de caractères, un dernier scan de l'agent montre qu'il n'est plus identifié comme étant une menace par Windows Defender :

<img src="../../assets/images/posts/redteam/mythic-custom-agent/apollo_final.png" style="display:block;margin:auto">

Nous pouvons désormais modifier directement le code source de l'agent installé sur notre instance Mythic et modifier les deux chaînes de caractères qui posaient problème. Les commandes suivantes permettent de remplacer toutes les occurerences de deux chaînes de caractères dans tous les fichiers du répertoire contenant le code source de l'agent Apollo

```
cd votre_repertoire_mythic

grep -RiIl "Apollo.Peers.TCP" InstalledServices/apollo/apollo/agent_code/ | sudo xargs sed -i "s/Apollo.Peers.TCP/Apollo.Pairs.TCP/g"
grep -RiIl "mythicFileId" InstalledServices/apollo/apollo/agent_code/ | sudo xargs sed -i "s/mythicFileId/mthcFileId/g"

sudo ./mythic-cli build apollo
```

À partir de maintenant, il est possible de générer des clients qui ne sont pas identifiés par Windows Defender directement depuis l'interface graphique de Mythic.

Nous essayons désormais d'exécuter l'agent avec la surveillance en temps réel activée et les dernières définitions téléchargées par Windows Defender :

<img src="../../assets/images/posts/redteam/mythic-custom-agent/apollo_final_1.png" style="display:block;margin:auto">

Et voilà, nous obtenons un accès distant à la machine via notre serveur Mythic :

<img src="../../assets/images/posts/redteam/mythic-custom-agent/apollo_final_2.png" style="display:block;margin:auto">

# Conclusion

* Les outils publiés finissent par être rapidement signés, identifiés et détectés au fil du temps.
* Les solutions antivirales basent leurs détections sur des signatures et des comportements enregistrés dans des bases de données.
* Les attaquants peuvent modifier le code source, parfois une seule ligne de code, des outils publiques pour contourner la logique de détection des antivirus.
* Les EDR étant des antivirus évolués, utilisent des moyens de détections supplémentaires pour identifier la présence de malwares sur une machine.

# Références :

- *Bypassing Defender with ThreatCheck & Ghidra* *: [https://offensivedefence.co.uk/posts/threatcheck-ghidra/](https://offensivedefence.co.uk/posts/threatcheck-ghidra/)*
- *Agent Customization in Mythic: Tailoring Tools for Red Team Needs :* [https://medium.com/@its_a_feature_/agent-customization-in-mythic-tailoring-tools-for-red-team-needs-1746fd02177f](https://medium.com/@its_a_feature_/agent-customization-in-mythic-tailoring-tools-for-red-team-needs-1746fd02177f)

