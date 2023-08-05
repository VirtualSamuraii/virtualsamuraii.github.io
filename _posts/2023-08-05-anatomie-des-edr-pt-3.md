---
layout: post
categories: redteam
title: "Anatomie des EDR pt.3 : Processus, services et registre"
permalink: "/:categories/:title/"
---


# Anatomie des EDR Pt.3 : Processus, services et registre

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_1.png" style="display:block;margin:auto;max-height:400px;width:auto">

Dans la première partie de cette série, nous avons remarqué la création de nouveaux services lors de l’installation d’un agent EDR. 

Ces services jouent un rôle crucial car ils permettent de démarrer les programmes automatiquement et garantissent le bon fonctionnement de l’ensemble de la solution.

Ici, nous allons voir, avec plus de détails, leur fonctionnement et l’intérêt de leur utilisation.

Ensuite, nous verrons quels mécanismes sont proposés par Windows pour protéger ces services et les processus en général.

Enfin, nous aborderons quelques techniques de contournement qui ont été démontrées par les chercheurs.

# Fondamentaux

Avant de rentrer dans le vif du sujet, il est indispensable d’effectuer quelques rappels sur les notions de processus et services sur Windows.

## Processus

Un processus correspond à une exécution d’un programme sur la machine.

Sur le système d’exploitation Windows, il existe 4 types de processus user-mode :

- **Processus utilisateur** : applications qui peuvent être en 32 ou 64 bits comme notepad par exemple.
- **Processus de service** : hébergent les services Windows comme PrintSpooler ou TaskScheduler et sont gérés par le gestionnaire de services (**Service Control Manager**).
- **Processus système** : natifs au système d’exploitation comme le processus logon ou le gestionnaire de session.
- **Processus d’environnement sous-système** : intermédiaires entre les autres processus et le kernel.

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_2.png" style="display:block;margin:auto">

Il est possible d’observer ces processus à l’aide d’outils comme **Process Explorer** ou **Process Hacker**. Ces derniers mettent en place un code couleur qui permet de différencier les types de processus sur la machine.

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_3.png" style="display:block;margin:auto">

Chaque processus est caractérisé par les éléments suivants :

- **Un espace privé d’adressage virtuel** : plage d’adresses mémoire virtuelle utilisables par le processus.
- **Un programme exécutable** : code et données chargées dans l’espace d’adressage virtuel
- **Une liste de handles ouverts** : pointeurs vers des ressources accessibles par les threads du processus (fichiers, objets, etc).
- **Un contexte de sécurité** : il s’agit d’un [jeton d’accès](https://learn.microsoft.com/fr-fr/windows/win32/secauthz/access-tokens) (**access token**) qui identifie l’utilisateur, les groupes de sécurité, les privilèges et d’autres informations sur le contexte du processus.
- **Un identifiant (PID)** : unique pour chaque processus
- **Au moins un thread d’exécution** : bien que des processus “vides” puissent exister.

Les processus peuvent être créés de manière abstraite, c’est à dire lorsqu’un utilisateur exécute un programme via un double clique sur une icône du bureau, ou bien de manière programmatique via du code en utilisant l’API Windows et notamment la fonction **CreateProcess**. 

De la même façon, **les processus peuvent être terminés par un utilisateur à condition qu’il possède le niveau de privilèges nécessaire.** 

Chaque processus est caractérisé par une structure **[EPROCESS](https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/ntos/ps/eprocess/index.htm)** qui contient des attributs et des pointeurs vers d’autres structures. 

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_4.png" style="display:block;margin:auto">

Un processus a un ou plusieurs threads qui sont eux mêmes représentés par des structures **[ETHREAD](https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/ntos/ps/ethread/index.htm).**

La structure **EPROCESS** et la plupart des autres structures existent dans l’espace d’adresses mémoire du système (**kernel** **mode**) à l’exception d’une seule : **Process Environment Block** (PEB) qui se trouve dans l’espace d’adresse du processus (**user** **mode**).

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_5.png" style="display:block;margin:auto">

## Services

Comme on l’a vu plus haut, un **service** est un **type de processus**, donc un programme, qui peut **démarrer automatiquement** lors du lancement du système d’exploitation, sans que l’utilisateur n’ait à se connecter ou à intervenir.

Chaque service est caractérisé par les éléments suivants :

- **Un nom**
- **Un état** : démarré ou arrêté
- **Un type** : Own / Share process (le service dispose de son propre processus ou bien partage le même processus avec d’autres services)
- **Un type de démarrage** : automatique, manuel ou désactivé

Sur Windows il existe des groupes de services qui partagent le même processus. En voici un exemple :

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_6.png" style="display:block;margin:auto">

On constate sur l’image ci-dessus que le processus, dont le PID est 772, héberge plusieurs services (PlugPlay, Power, etc).

Les groupes de services natifs à Windows sont hébergés par le processus système **svchost.exe.**

Les services peuvent s’exécuter dans le contexte d’un utilisateur local ou du domaine dans un environnement Active Directory.

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_7.png" style="display:block;margin:auto">

Dans cet exemple, on constate que le groupe de service est exécuté dans le contexte de l’utilisateur NT AUTHORITY\SYSTEM qui correspond à l’utilisateur le plus privilégié sur une machine.

**Mais alors pourquoi utiliser des services ?**

Comme on l’a vu dans les précédents articles, un EDR c’est plusieurs programmes qui s’exécutent et qui s’imbriquent pour former un ensemble harmonieux qui espionne une machine. 

Si l’utilisateur doit démarrer manuellement tous les programmes à chaque fois qu’il allume son poste de travail, cela deviendrait très vite un enfer.

Les services permettent de régler ce problème en démarrant les programmes de l’EDR de manière automatique.

Pour illustrer mes propos, je vais rester sur l’exemple de la solution SentinelOne, qui semble être l’une des plus complètes.

Après installation de l’agent S1, on constate la création de 4 services

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_8.png" style="display:block;margin:auto">

Parmi ceux là, ***SentinelAgent*** et ***SentinelHelperService*** sont de type “**own process**” et les deux autres (***SentinelStaticEngine*** et ***LogProcessorService***) sont de type “**share process**”.

Le service **SentinelAgent** permet de lancer le programme **SentinelAgent.exe** qui est en quelques sorte la **colonne vertébrale de l’EDR**. C’est ce programme qui va ensuite faire appel aux autres composants (DLL, exécutables, drivers) pour détecter les éventuelles menaces et réagir sur la machine.

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_9.png" style="display:block;margin:auto">

Comme on le voit sur l’image, **SentinelAgent.exe** fait appel à deux autres programmes : **SentinelAgentWorker.exe** et **SentinelStaticEngineScanner.exe**.

Un “**worker**” est un programme auquel on affecte une tâche spécifique. Lorsque cette tâche est accomplie, le résultat est renvoyé au programme source via une méthode de rappel (callback method).

Le service **SentinelStaticEngine**, gère le démarrage de **SentinelStaticEngine.exe** qui prend en charge le téléchargement des dernières définitions et signatures de virus et qui lui même fait appel à **SentinelStaticEngineScanner.exe.** Ce dernier, comme son nom l’indique, s’occupe de l’analyse statique. Il peut également être appelé par l’agent lorsqu’il en a besoin. 

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_10.png" style="display:block;margin:auto">

Dans un prochain article, nous verrons les stratégies d’analyses (**statique** et **dynamique**) mises en place par les EDR pour distinguer les programmes malveillants des programmes légitimes.

Enfin, le service **LogProcessorService** s’occupe du traitement des journaux d’évènements Windows.

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_11.png" style="display:block;margin:auto">

Ce service est également accompagné d’un autre programme **LogCollector.exe** qui s’occupe de la collecte des journaux.

L’outil [serviceDetector](https://github.com/tothi/serviceDetector) permet de vérifier la présence de certains services sur une machine distante via SMB. C’est pratique pour savoir si une machine du réseau est surveillée par un agent EDR ou pas.

Pour la suite de l’article, on va se mettre dans le contexte d’un attaquant qui dispose d’un accès privilégié (administrateur) à une machine et sur laquelle est installé un agent EDR.

A partir de là, il y a 3 questions que l’on devrait se poser :

- peut-on désactiver l’agent ?
- que se passe-t-il si on décide de terminer les processus ?
- que se passe-t-il si on décide d’arrêter les services ?

# Sécurité des EDR

Au fil du temps, les éditeurs de solutions antivirales ont dû s’adapter à la ruse des attaquants et implémenter des mécanismes de sécurité développés par eux-mêmes (**Anti-tampering**) ou proposés par Microsoft. Le but étant de limiter les interactions des attaquants avec les composants de la solution (processus, services, drivers, fichiers, etc). 

## Anti-tampering

Les éditeurs de solutions de sécurité proposent généralement un programme qui permet d’administrer l’agent via la ligne de commande.

Dans le cas de SentinelOne, il existe un programme appelé **SentinelCtl.exe** qui permet d’effectuer des actions sur l’agent (scan, configuration, activation/désactivation, etc).

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_12.png" style="display:block;margin:auto">

Ainsi, pour répondre à la première question, il est tout à fait possible de **désactiver un agent EDR, à condition de posséder le jeton** qui autorise à déclencher cette action.

D’ailleurs, si on essaye sans jeton ou avec un mauvais jeton, le programme nous envoie balader. 

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_13.png" style="display:block;margin:auto">

Ce jeton est en réalité réservé aux administrateurs système et aux “équipiers bleu” pour interagir avec les agents en cas de besoin. Il est accessible depuis la console de gestion de la solution.

Dans le cas d’une opération red team, cette option n’est pas la plus intuitive pour les attaquants, bien qu’elle puisse être envisageable.

Comme on peut le constater, le programme SentinelCtl.exe est conçu de manière à être lancé avec des arguments, parmi lesquels se trouve le jeton en clair. Si un attaquant dispose d’un accès à la machine, il peut tout simplement lister les processus et les arguments des programmes pour récupérer le jeton au bon moment.

## Protected Process

Dans le modèle de sécurité de Windows, n’importe quel processus exécuté avec un token qui contient le privilège **SeDebugPrivilege** (comme celui d’un compte administrateur) peut demander n’importe quel accès à n’importe quel autre processus sur la machine. 

Cela lui permet de lire ou écrire la mémoire des autres processus, d’y injecter du code, de suspendre les threads, bref d’avoir un contrôle total dessus.

A l’époque où les gens ~~pirataient~~ regardaient des films et écoutaient de la musique sur CD / DVD, ce modèle de fonctionnement agaçait l’industrie des médias qui faisait alors pression sur les éditeurs comme Microsoft afin que leurs systèmes d’exploitation prennent en charge la **lecture sécurisée** des nouveaux contenus Blu-ray.

C’est à partir de Windows Vista et Windows Server 2008 que Microsoft a introduit les processus protégés (**protected** **processes**).

Les processus protégés et leurs cousins PPL (protected process light) ont un **bit spécial** dans leur structure **EPROCESS** qui modifie le comportement des routines de sécurité du gestionnaire de processus pour refuser certaines permissions d’accès qui seraient initialement accordées à des comptes administrateurs.

Un processus protégé peut être créé par n’importe quelle application. Néanmoins, le système d’exploitation n’autorise un processus à être protégé que si son image a été signée par un certificat Windows Media Certificate.

## Protected Process Light (PPL)

A partir de Windows 8.1 et Windows Server 2012, Microsoft a introduit la fonctionnalité PPL qui est une extension du modèle du processus protégé. L’objectif de cette nouvelle extension est d’apporter plus de granularité au niveau du contrôle d’accès sur les processus protégés.

Les processus PPL sont protégés de la même manière que les “protected processes” mais utilisent une liste de signataires (**Signers**) qui ont un niveau de confiance allant du plus élevé (**WinSystem**) au plus faible (**None**).

Ci-dessous, la liste des types de protection associées aux “**Signers**”.

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_14.png" style="display:block;margin:auto">

Et ci-dessous, la liste des niveaux de confiance associés à leur utilisation :

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_15.png" style="display:block;margin:auto">

Cette hiérarchie donne lieu à des différences de protection des processus. 

Par exemple, un processus protégé avec **PsProtectedSignerWinSystem** aura accès à tous les autres processus.

Dans le cas de S1, on remarque que les processus SentinelAgent.exe et SentinelHelperService.exe semblent être protégés :

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_16.png" style="display:block;margin:auto">

Voici ce qu’il se passe lorsqu’on essaye de terminer le processus SentinelAgent.exe ou l’un de ces processus enfants :

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_17.png" style="display:block;margin:auto">

Ainsi, pour répondre à la deuxième question, il n’est théoriquement pas possible pour les utilisateurs (même les plus privilégiés) de terminer un processus protégé. On verra qu’en pratique, c’est différent.

## Protected services

Comme les solutions antivirales utilisent les services pour faire tourner leur ~~joyeux bordel~~ ensemble de programmes, les attaquants ont commencé à les cibler directement, plutôt que de tuer un processus qui va de toute façon réapparaître.

C’est pour cette raison que Microsoft a introduit le concept **protected service** qui est une variante du modèle **protected process,** principalement pour protéger les services anti-malware contre la modification (injection de code, démarrage, arrêt, activation et désactivation). 

En clair, seul du code signé, par Microsoft ou par les certificats des éditeurs de solutions antivirales, est autorisé à être chargé dans le service protégé (il faut donc aussi oublier le chargement de DLL malicieuse).

Pour qu’un service anti-malware s’exécute en tant que service protégé, l’éditeur doit disposer d’un pilote ELAM installé sur la machine. 

Par ailleurs, Microsoft dit : “*Le nouveau modèle de sécurité permet également aux services protégés contre les programmes malveillants de **lancer des processus enfants comme protégés**. Ces processus enfants s’exécutent au même niveau de protection que le service parent et leurs fichiers binaires doivent être signés avec le même certificat qui a été inscrit via la section de ressource ELAM.*”

Enfin, les niveaux de protection d’un service sont définis dans la structure **[SERVICE_LAUNCH_PROTECTED_INFO](https://learn.microsoft.com/en-us/windows/win32/api/winsvc/ns-winsvc-service_launch_protected_info).**

Voici ce qu’il se passe lorsqu’on essaye d’arrêter le service **SentinelAgent** :

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_18.png" style="display:block;margin:auto">

Donc pour répondre à la troisième question, tout comme pour les processus protégés, il n’est théoriquement pas possible pour un utilisateur d’interagir avec le service protégé. 

## Registre Windows

Le registre est une base de données qui joue un rôle central dans la configuration et le contrôle des systèmes Windows.

Généralement, les éditeurs utilisent des clés de registre Windows pour configurer leur solution anti-malware et notamment les services.

Voici un exemple de clés de registre de la solution S1 :

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_19.png" style="display:block;margin:auto">

Les clés de registre qui gèrent les services de la solution anti-malware se trouvent dans la ruche **HKLM\SYSTEM.** 

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_20.png" style="display:block;margin:auto">

On constate constate la présence d’une clé **LaunchProtected** dont la valeur (0x3) correspond à la protection **SERVICE_LAUNCH_PROTECTED_ANTIMALWARE_LIGHT.**

# Contournements et fourberies de Scapin

## Protected process

Un attaquant peut terminer un processus protégé s’il a la possibilité d’écrire dans la mémoire kernel. En exploitant un driver vulnérable par exemple, il peut tout simplement modifier l’attribut **Protection** de la structure **EPROCESS** correspondant au processus qu’il cible.

Cet attribut est en réalité une structure de type **[PS_PROTECTION](https://learn.microsoft.com/en-us/windows/win32/procthread/zwqueryinformationprocess)** qui contient les informations sur la protection du processus cible.

Le problème c’est que même si on arrive à terminer le processus, ce dernier risque d’être initialisé à nouveau par le service qui démarre le programme automatiquement à intervalle régulier

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_21.png" style="display:block;margin:auto">

Cela reste tout de même intéressant d’un point de vue offensif. Si le service est configuré de sorte à ce qu’il redémarre après un temps assez long alors cela nous laisse la possibilité d’effectuer des actions malveillantes, sans être interrompus par l’agent EDR durant cette fenêtre de temps.

## Protected service

Comme on l’a vu plus haut, un service est également représenté par des clés de registre qui définissent ses propriétés. 

Si un attaquant a la possibilité de modifier ces clés, alors il peut tout simplement désactiver la protection. 

Voici ce qu’il se passe lorsqu’on essaye de modifier la valeur de la clé **LaunchProtected**

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_22.png" style="display:block;margin:auto">

## Clé de registre

Si on peut pas attaquer les services alors autant attaquer le registre !

De façon générale, les éditeurs protègent les clés de registre en utilisant des méthodes de rappel comme **CmRegisterCallback** ou **PsSetCreateProcessNotifyRoutine** qui sont enregistrées par les pilotes (ce mécanisme est d’ailleurs expliqué dans [l’article précédent](https://virtualsamuraii.github.io/redteam/anatomie-des-edr-pt-2/)).

Ainsi, toute tentative de modification notifierait le pilote et pourrait éventuellement lever une alerte sur la console.

Tout comme pour les processus protégés, si un attaquant a la possibilité d’écrire dans la mémoire du kernel alors il peut tout simplement “patcher” les méthodes de rappel de sorte à ce que le pilote ne soit plus averti lorsqu’un utilisateur modifie la valeur d’une clé.

## Résumé

Voici un superbe schéma, fait par Daniel Feichter qui résume les possibilités de contournement :

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-3/img_23.png" style="display:block;margin:auto">

# Conclusion

- Les éditeurs de solutions antivirales utilisent les services pour orchestrer automatiquement l’ensemble des programmes.
- Ces services et processus sont souvent attaqués et Microsoft a introduit les concepts de “**protected processes**” et “**protected services**” pour empêcher les utilisateurs, même privilégiés, d’interagir avec.
- Ces concepts ne sont pas infaillibles et peuvent être contournés si l’attaquant dispose d’un accès à la mémoire kernel (via une vulnérabilité sur un pilote ou en enregistrant son propre pilote signé).

# Références :

- *Windows Internals 7th Edition Parts 1 and 2 de Pavel Yosifovich (Auteur), Mark E. Russinovich, David A. Solomon, Alex Ionescu - 3 mai 2017*
- [https://learn.microsoft.com/fr-fr/windows/win32/services/services](https://learn.microsoft.com/fr-fr/windows/win32/services/services)
- [https://learn.microsoft.com/fr-fr/windows/win32/services/protecting-anti-malware-services-](https://learn.microsoft.com/fr-fr/windows/win32/services/protecting-anti-malware-services-)
- [https://itm4n.github.io/lsass-runasppl/](https://itm4n.github.io/lsass-runasppl/)
- [https://www.sentinelone.com/blog/log-monitoring/](https://www.sentinelone.com/blog/log-monitoring/)
- [https://redops.at/en/blog/a-story-about-tampering-edrs](https://redops.at/en/blog/a-story-about-tampering-edrs)
- [https://blog.scrt.ch/2023/03/17/bypassing-ppl-in-userland-again/](https://blog.scrt.ch/2023/03/17/bypassing-ppl-in-userland-again/)

# Outils :

- [https://github.com/itm4n/PPLmedic](https://github.com/itm4n/PPLmedic)
- [https://github.com/itm4n/PPLdump](https://github.com/itm4n/PPLdump)
- [https://github.com/itm4n/PPLcontrol](https://github.com/itm4n/PPLcontrol)
- [https://github.com/RedCursorSecurityConsulting/PPLKiller](https://github.com/RedCursorSecurityConsulting/PPLKiller)
- [https://github.com/wavestone-cdt/EDRSandblast](https://github.com/wavestone-cdt/EDRSandblast)