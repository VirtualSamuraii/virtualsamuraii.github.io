---
layout: post
categories: redteam
title: "Anatomie des EDR pt.1 : Architecture générale"
permalink: "/:categories/:title/"
---

# Anatomie des EDR Pt.1

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-1/surgeon.png" style="max-height:500px;display:block;margin:auto">


Cet article est le premier d’une collection de notes personnelles et d’expérimentations. Il a pour objectif de dresser un portrait générique des solutions de sécurité appelées **EDR**.

Pour commencer, je décrirai l’architecture générale qu’on peut retrouver avec la plupart des solutions du marché.

Ensuite, je présenterai les composants logiciels (exécutables, services, DLL, drivers…) qui sont installés sur les machines surveillées (appelées aussi **endpoints**).

Pour illustrer mes propos, j’utilise la solution **SentinelOne** en exemple car il s’agit d’un des leaders du marché.

# Architecture générale

## Définition

**EDR** pour **Endpoint Detection and Response** est un terme qui définit une solution de sécurité qui combine plusieurs éléments, couches et techniques pour détecter des menaces et y répondre, sur un poste de travail ou un serveur.

Quelle est la différence entre un **AV** (Antivirus) et un **EDR** ? 

## Un peu d’histoire

À l’époque où les malwares étaient écrits juste pour le fun par des ~~cinglés~~ passionnés, en slip devant leur écran, il existait très peu voire pas du tout de moyens de défense. Puis, un jour, ces mecs là sont passés de “juste pour le fun” à “pourquoi pas un peu de profit hehe”. 

Sans oublier les puissances militaires, qui avaient déjà compris l’intérêt de ces nouvelles armes pouvant parfois renverser une nation entière (par exemple Stuxnet dans le conflit Israël / Iran pour l’armement nucléaire). 

Historiquement, les AV étaient les premières solutions de sécurité développées pour réagir face à l’émergence de ces nouvelles menaces.

Au début, les AV reproduisaient une analyse statique et les détections étaient principalement basées sur les signatures des fichiers malveillants. Puis, avec le temps, ils ont commencé à reproduire une analyse dynamique, par exemple, en émulant le programme malveillant dans un bac à sable (sandbox).

En face, les **maldev*** ripostaient avec des **nouvelles techniques de contournement de bac à sable (sandbox)** et implémentaient des techniques d’**anti rétro-ingénierie et anti-débogage.**

Avec les années, les éditeurs ont dû s’adapter à l’évolution de ces malwares et développer de nouvelles solutions plus robustes et plus intelligentes. Il a fallu créer des nomenclatures pour le marketing et la vente de ces produits (EDR/XDR/MDR). 

Finalement, on peut dire qu’**un EDR est un AV mais avec plus de fonctionnalités et de moyens de détection**. Nous verrons par la suite pourquoi. 

Certains diront même qu’un AV/EDR est un malware. Ce fut notamment le cas de Kaspersky durant l’histoire du leak des outils de la NSA par le groupe TheShadowBrokers en 2016.

***maldev** : développeur de malwares

## Fonctionnement global

Voici un exemple générique d’architecture utilisée par les éditeurs de solutions de sécurité.

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-1/edr_archi.png" style="display:block;margin:auto">

**L’EDR collecte des données** sur les systèmes qu’il surveille en temps réel. Les données peuvent regrouper des fichiers, adresses IP, URLs, clés de registre, des processus. Bref, tout ce qui peut se produire sur un système est collecté.

Ces données sont ensuite transférées, avec amour et respect de la RGPD, sur l’infrastructure de l’éditeur de la solution et peuvent faire l’objet d’analyses approfondies. 

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-1/sentinelone_ui.png" style="display:block;margin:auto">

Sur l’image ci-dessus, on peut observer deux URLs. La première correspond à l’interface de gestion et la seconde au “**cloud**” de l’éditeur de la solution.

Les analystes et utilisateurs de la solution peuvent se connecter à une interface, exposée sur internet qui leur permet de gérer le parc des machines surveillées, les alertes, les configurations et politiques de sécurité.

Ci-dessous, un exemple de l’interface de gestion de la solution **SentinelOne** :

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-1/sentinelone_ui_home.png">

Lors de la détection d’une menace, l’**EDR** **peut également y répondre** à l’aide d’actions pré-configurées (arrêter un processus, éteindre l’hôte, l’isoler sur le réseau, etc). 

Il est aussi possible d’administrer les machines à distance et d’exécuter tout un panel d’actions sur les agents installés.

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-1/sentinelone_actions.png" style="display:block;margin:auto">

# Composants logiciels

## Paquet d’installation

Lorsqu’un agent est installé sur une machine, il se passe tout un tas de choses qu’on va essayer de voir ensemble dans cette partie.

Les agents sont distribués par les éditeurs sous la forme de paquets pour plusieurs plateformes (Linux, Windows…). 

Ces agents sont mis à jour plus ou moins régulièrement selon les éditeurs et les vulnérabilités découvertes, qu’elles soient applicatives ou logiques (par exemple, la possibilité de contourner une règle de détection).

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-1/sentinelone_package.png">

Il faut bien avoir en tête que ces solutions de sécurité sont étudiées par des milliers de chercheurs (qu’ils soient professionnels, criminels ou les deux). 

## Répertoire d’installation

Lorsque l’agent est installé, un répertoire est créé généralement ici :

```powershell
C:\Program Files\Nom_de_EDR\Version_EDR\
```

Par exemple pour SentinelOne.

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-1/sentinelone_install_folder.png">

Ce répertoire contient tous les exécutables, bibliothèques, fichiers de configurations et ressources utilisées par l’agent.

 

## Drivers

S1 enregistre 3 nouveaux pilotes lors de l’installation de l’agent sur le système :

- SentinelDeviceControl
- SentinelELAM
- SentinelMonitor

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-1/sentinelone_drivers.png" style="display:block;margin:auto">

Il existe plusieurs types de pilotes :

- Pilotes en mode utilisateur
- Pilotes en mode noyau

La documentation de Microsoft fait un excellent travail pour les décrire :

[https://learn.microsoft.com/fr-fr/windows-hardware/drivers/kernel/types-of-windows-drivers](https://learn.microsoft.com/fr-fr/windows-hardware/drivers/kernel/types-of-windows-drivers) 

Parmi ces pilotes, **SentinelDeviceControl** sert à surveiller le système. Pour être plus précis, il écoute les évènements qui se produisent et fournit une fonction au système, qu’on appelle une **callback routine**, qui sera exécutée lorsqu’un évènement qui l’intéresse se produit (création d’un processus, d’un thread, etc). 

Microsoft documente ce mécanisme ici : [https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/callback-objects](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/callback-objects)

Quel est l’intérêt de faire ça ? En surveillant chaque nouveau processus ou thread, le driver peut ensuite injecter une DLL qui se chargera de surveiller les appels aux fonctions de l’API Windows, effectués par le processus. C’est ce qu’on appelle l’**API Hooking**.

Pour le moment, tout ce qu’on doit retenir c’est que ces pilotes ont chacun un rôle bien spécifique et un fonctionnement plus ou moins publiquement documenté.

Nous verrons plus en détail le fonctionnement de ces drivers et des kernel callbacks dans un prochain article.

## Services et utilisateurs

En général, après avoir installé l’agent, on remarque la création de plusieurs services. Dans le cas de SentinelOne, il y en a 4 :

- Sentinel Agent (gère l’agent SentinelOne.exe)
- SentinelHelperService
- SentinelOne Agent Log Processing Service (gère les journaux d’évènements)
- SentinelOne Static Service (gère les moteurs statiques pour SentinelOne Endpoint Protection)

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-1/sentinelone_services.png" style="display:block;margin:auto">

Ces services orchestrent le tout et forment cette douce symphonie qui rythme les journées des blue teamers.

## Exécutables, bibliothèques et utilitaires

Comme on l’a vu plus haut, dans le répertoire d’installation se trouvent plusieurs dossiers et fichiers indispensables ou complémentaires à l’utilisation de l’agent EDR.

Parmi ces fichiers, il existe des exécutables qui sont régis par les services qu’on a vu précédemment, qui permettent de scanner le système de fichiers et la mémoire vive (c’est par exemple le cas de **SentinelMemoryScanner.exe** ou **SentinelStaticEngineScanner.exe**) et d’autres qui permettent d’administrer le poste de travail (c’est le cas de **SentinelCtl.exe**).

Ces exécutables vont charger des bibliothèques dynamiques (**DLL**) développées par l’éditeur et qui exposent des fonctions.

Par exemple, si on lance notepad sur la machine, on peut apercevoir la présence de deux DLL (**kern3l32.dll** et **ntd1l.dll**) injectées dans l’espace mémoire virtuel du processus.

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-1/sentinelone_dll.png">

Comme on l’a vu plus haut avec les drivers, les DLL qui sont injectées servent à surveiller les appels aux fonctions de l’API Windows (qui sont dans **kernel32.dll** et **ntdll.dll**) effectués par le processus notepad. 

En conclusion :

- L’architecture d’un EDR est composée de plusieurs élements et couches. 
- Un agent est installé sur chaque machine et collecte les données.
- Les services gèrent les exécutables et permettent le bon fonctionnement de la solution.
- Un pilote (driver) est installé et surveille la création de nouveaux processus, threads, etc.
- Chaque nouveau process se voit injecter une ou plusieurs DLL qui vont surveiller les appels aux fonctions de l’API Windows (API Hooking).
- Si des actions malveillantes sont détectées, elles sont analysées et une réponse a lieu.

Dans les prochains articles, nous verrons comment fonctionne en détail chaque brique du système de détection ainsi que les méthodes de contournement découvertes par les chercheurs.