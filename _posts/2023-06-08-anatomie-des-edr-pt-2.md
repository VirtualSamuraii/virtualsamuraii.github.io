---
layout: post
categories: redteam
title: "Anatomie des EDR pt.2 : Kernel, Drivers et Callbacks"
permalink: "/:categories/:title/"
---

# Anatomie des EDR Pt.2 : Kernel, Drivers & Callbacks

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-2/0-xp.png" style="display:block;margin:auto;max-height:400px;width:auto">

Nous avons vu précédemment qu’un **EDR** c’est avant tout des **pilotes** qui sont installés sur le système.

Dans cet article, nous allons voir le **rôle et le fonctionnement de ces pilotes** avec un peu plus de détails. 

Pour commencer, je présenterai l’**architecture du système Windows** et l’intérêt d’utiliser des pilotes pour un EDR. 

Ensuite, nous verrons comment **ces pilotes peuvent surveiller un système** à l’aide de mécanismes fournis par Microsoft ainsi que des techniques de contournement qui ont été découvertes. 

Des bases en théories des systèmes d’exploitation sont requises pour faciliter la lecture et la compréhension de cet article.

# Architecture du système Windows

## Fondamentaux

Les processeurs de la famille x86 implémentent 4 anneaux de privilèges : 

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-2/1-rings.png" style="display:block;margin:auto">

Seuls 2 anneaux sont utilisés par le système d’exploitation Windows : ring 0 et ring 3 qui correspondent respectivement au **Kernel Mode et User Mode**.

Le livre *Windows Internals 7th edition Part 1* décrit l’architecture du système d’exploitation Windows avec le schéma ci-dessous :

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-2/2-archi.png" style="display:block;margin:auto">

*Source : Windows Internals 7th Edition Part 1*

Sur le schéma, on distingue clairement la frontière entre le User Mode et le Kernel Mode.

Parmi les composants du Kernel Mode, il y en a trois qui nous intéressent : 

- **Executive** : Ensemble de services de base du système d’exploitation (gestion de la mémoire, des processus et threads, réseau…)
- **Kernel** : Ensemble de fonctions système de bas-niveau comme la planification des threads, interruptions systèmes et gestion d’erreurs. Il fournit aussi des “routines” et des objets que l’**executive** utilise.
- **Device** **Drivers** : inclut à la fois les pilotes matériels qui traduisent les entrées/sorties utilisateur en requête d’entrée/sortie pour le matériel mais également le reste des pilotes non-matériels comme ceux qui gèrent le système de fichier ou le réseau par exemple.

Les fichiers qui correspondent à ces composants sont les suivants :

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-2/3-fichiers.png" style="display:block;margin:auto">

**Mais alors pourquoi utiliser un pilote ?**

On remarque sur le schéma précédent que les pilotes (**device drivers**) et le noyau (**kernel**) s’exécutent avec le même niveau de privilèges. Ainsi, lorsqu’un pilote de type kernel est exécuté, il possède les mêmes droits que le kernel lui même et **peut donc le modifier**.

Cela devient alors très intéressant pour les développeurs de malwares et les éditeurs d’antivirus qui peuvent contrôler la totalité du système.

De plus, Microsoft met à disposition des développeurs une API (ensemble de fonctions kernel) qui est plutôt bien documentée ([https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/_kernel/](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/_kernel/)).

## Kernel Patching Protection (Patchguard)

**Qu’est-ce que le kernel ?**

Le noyau est la partie la plus basse et la plus centrale d’un système d’exploitation. C’est l’un des premiers programmes qui s’exécutent lorsque la machine démarre. 

Le noyau permet aux applications de communiquer avec le matériel de l’ordinateur (carte son, carte vidéo, interface réseau…) et fournit de nombreuses fonctionnalités (gérer la mémoire, lancer des programmes, gérer les données sur le disque…).

Puisque tous les autres programmes dépendent du noyau, le moindre « bug » sur ce dernier peut les faire crash ou les faire agir de manière imprévisible.   

Le schéma ci-dessous permet de mieux comprendre l’importance du kernel :

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-2/4-archi.png" style="display:block;margin:auto">

Pour faire simple, le kernel met à disposition des fonctions (**system services routines**) qui peuvent être appelées par des pilotes ou des applications du user mode.

Ces fonctions permettent l’écriture de fichiers, la création de processus et plein d’autres actions sur le matériel comme la caméra ou le micro par exemple.

Les adresses de ces fonctions sont référencées dans la **SSDT** (System Service Dispatch Table).

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-2/5-ssdt.png" style="display:block;margin:auto">

*Source: Windows Internals 7th Edition Part 1*

Chaque fonction possède un identifiant d’appel système (**syscall identifier**) qui lui est associé et qui correspond en réalité à un index dans la table SSDT.

Ce syscall est utilisé par le **System Service Dispatcher** pour convertir les appels de fonctions de l’API Windows venant du user mode vers les fonctions du kernel correspondantes. 

Il existe une table des syscalls documentée par des volontaires : [https://windiff.vercel.app/](https://windiff.vercel.app/) 

Ci-dessous un magnifique schéma illustrant un appel à la fonction **ReadFile** par une application user mode (notepad.exe ou une autre application qui ouvre un fichier sur le disque par exemple).

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-2/7-syscall.png" style="display:block;margin:auto;max-height:600px;width:auto">

*Source : [https://www.matteomalvica.com/minutes/windows_kernel/](https://www.matteomalvica.com/minutes/windows_kernel/)*

**Qu’est-ce que le Kernel Patching ou Kernel Hooking ?**

Il s’agit d’une pratique de développement qui consiste à modifier le code du noyau durant son exécution.

Les développeurs de malwares utilisaient cette technique car elle était puissante et leur permettait d’implémenter des **rootkits** qui cachaient également la présence d’autres malwares sur le système.

En face, les éditeurs de solutions de sécurité EDR ont commencé à faire pareil pour traquer les malwares directement dans le noyau (kernel hooking). 

Cette pratique viole l’intégrité du noyau et peut le rendre instable, provoquant ce qu’on appelle un **BSoD** (Blue Screen of Death). Elle n’a jamais été officiellement documentée et a toujours été fortement déconseillée par Microsoft.

Il existe plusieurs techniques de kernel hooking mais je vais en citer que 2 :

- **SSDT Hooking** : consiste à modifier l’adresse des fonctions du kernel qui sont définies dans la table SSDT pour les faire pointer sur les fonctions de l’EDR ou du rootkit.
- **Inline Hooking** : consiste à modifier des instructions assembleur au début d’une fonction (routine) exposée par le kernel de sorte à rediriger le flux d’exécution vers une fonction de l’EDR ou du rootkit.

Le schéma ci-dessous illustre un exemple de hooking SSDT de la fonction **NtOpenProcess** :

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-2/hookssdt.png" style="display:block;margin:auto">

*Source : [https://www.adlice.com/kernelmode-rootkits-part-1-ssdt-hooks/](https://www.adlice.com/kernelmode-rootkits-part-1-ssdt-hooks/)*

- Ici, le malware ou l’EDR modifie l’adresse (ou l’offset sur les systèmes 64 bits) de la fonction **NtOpenProcess** dans la table SSDT.
- Lorsqu’une application ou un pilote du user mode fait appel à cette fonction, elle sera renvoyée vers l’adresse de la fonction du malware ou de l’EDR.
- La fonction du malware ou de l’EDR effectue alors ses actions et redirige ensuite le flux vers la fonction originale **NtOpenProcess**.

Pour les ~~fous furieux~~ curieux qui veulent se lancer dans le développement de pilotes, voici un exemple d’un driver écrit en langage C qui pratique ce qu’on appelle de l’**inline hooking** sur le kernel : [https://github.com/adrianyy/kernelhook](https://github.com/adrianyy/kernelhook)

**Qu’est-ce que le Kernel Patch Protection (Patchguard) ?**

Comme on l’a dit plus haut, le moindre bug sur le code du kernel peut rendre le système instable et ça, Microsoft ne voulait pas en entendre parler.

C’est à partir de Windows Server 2003 SP1 et Windows XP Professional x64 Edition que cette nouvelle fonction de sécurité fait son apparition pour la première fois.

**Patchguard** surveille la modification de certaines ressources clés utilisées par le kernel ou bien le code du kernel lui même. **Si une modification non autorisée a lieu, il force le système à s’éteindre** et c’est à partir de là que les maldevs et les éditeurs de solutions de sécurité ont commencé à rager.

Scott Field a publié un article sur ce mécanisme. Il y décrit également les alternatives au kernel patching :

[https://learn.microsoft.com/en-us/archive/blogs/windowsvistasecurity/an-introduction-to-kernel-patch-protection](https://learn.microsoft.com/en-us/archive/blogs/windowsvistasecurity/an-introduction-to-kernel-patch-protection)

# Kernel Callbacks

## Fonctionnement

Microsoft a fourni une alternative aux éditeurs de solutions de sécurité : les **Callbacks**. 

Un pilote peut créer un **objet Callback**, qui possède une ou plusieurs **callback routines** (fonctions). D’autres pilotes peuvent demander à être notifiés par cet objet callback lorsque les conditions définies par le pilote créateur sont réunies.

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-2/8-callback.png" style="display:block;margin:auto">

*Source : Microsoft*

Il existe quelques fonctions mises à disposition par Microsoft et qui sont généralement utilisés par les éditeurs d’antivirus :

- **PsSetCreateProcessNotifyRoutine**
- **PsSetCreateThreadNotifyRoutine**
- **PsSetLoadImageNotifyRoutine**

Ces fonctions permettent aux éditeurs de surveiller la création de nouveaux processus, threads ou encore le chargement d’images (driver, exécutable, DLL).

Ainsi, lorsqu’un nouveau processus est créé, le driver est notifié et il a la possibilité d’effectuer une action :

- bloquer le démarrage du processus,
- injecter une DLL dans son espace mémoire pour faire du **hooking**.

Néanmoins, ceci a forcé les éditeurs à effectuer le **patching ou hooking** en user mode plutôt qu’en kernel mode.

Par exemple, **Sysmon** et les **EDR** enregistrent, à travers leur driver, une **callback routine qui sera exécutée chaque fois qu’un processus est créé**. 

Pour faire ça, le driver utilise l’une de ces routines :

- **[PsSetCreateProcessNotifyRoutine](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/ntddk/nf-ntddk-pssetcreateprocessnotifyroutine?redirectedfrom=MSDN)**
- **[PsSetCreateProcessNotifyRoutineEx](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/ntddk/nf-ntddk-pssetcreateprocessnotifyroutineex?redirectedfrom=MSDN)**

Ci-dessous, un exemple de code en C pour un driver qui enregistre une callback routine en utilisant **PsSetCreateProcessNotifyRoutineEx :**

```c
#include <ntddk.h>

// Callback routine à exécuter lorsqu'un processus est crée
VOID ProcessNotifyRoutineEx(
    _Inout_ PEPROCESS Process,
    _In_ HANDLE ProcessId,
    _Inout_opt_ PPS_CREATE_NOTIFY_INFO CreateInfo
)
{
    UNREFERENCED_PARAMETER(Process);

    if (CreateInfo != NULL)
    {
        KdPrint(("Process created: PID=%lu, ImagePath=%wZ\n", (ULONG)ProcessId, CreateInfo->ImageFileName));
    }
    else
    {
        KdPrint(("Process terminated: PID=%lu\n", (ULONG)ProcessId));
    }
}

// Fonction pour gérer le déchargement du pilote
VOID UnloadDriver(PDRIVER_OBJECT pDriverObject)
{
    UNREFERENCED_PARAMETER(pDriverObject);

    // Supprimer la callback routine enregistrée
    PsSetCreateProcessNotifyRoutineEx(ProcessNotifyRoutineEx, TRUE);

    KdPrint(("Driver unloaded\n"));
}

// Driver entry point
NTSTATUS DriverEntry(PDRIVER_OBJECT pDriverObject, PUNICODE_STRING pRegistryPath)
{
    UNREFERENCED_PARAMETER(pRegistryPath);

    KdPrint(("Driver loaded\n"));

    // Enregistrer la callback routine 
    NTSTATUS status = PsSetCreateProcessNotifyRoutineEx(ProcessNotifyRoutineEx, FALSE);
    if (!NT_SUCCESS(status))
    {
        KdPrint(("Failed to register process notification callback: %08X\n", status));
        return status;
    }

    // Fonction de déchargement du driver
    pDriverObject->DriverUnload = UnloadDriver;

    // Effectuer d'autres actions ici

    return STATUS_SUCCESS;
}
```

**Petit rappel** : pour pouvoir charger un driver sur un système il faut disposer du privilège **SeLoadDriverPrivilege** qui est attribué par défaut aux comptes **Administrator** et **Print** **Operator.**

Par ailleurs, ces callbacks routines enregistrées par les EDR sont stockées dans le tableau **PspCreateProcessNotifyRoutine** en mémoire kernel. 

Chaque entrée dans ce tableau contient un pointeur vers la routine enregistrée par chaque driver.

# Contournement

Au fil des années, les chercheurs ont mis en évidence des techniques pour contourner certains callbacks. C’est notamment le cas de l’excellente équipe MDSec qui a démontré comment tromper les callbacks enregistrés par **PsSetLoadImageNotifyRoutine** dans cet article : [https://www.mdsec.co.uk/2021/06/bypassing-image-load-kernel-callbacks/](https://www.mdsec.co.uk/2021/06/bypassing-image-load-kernel-callbacks/)

L’article est accompagné d’un projet **[DarkLoadLibrary](https://github.com/bats3c/DarkLoadLibrary)** qui est une réimplémentation de LoadLibrary, sauf qu’aucune notification est émise lors du chargement d’une image (DLL).

Enfin, il est tout à fait possible, **à partir d’un utilisateur sans privilèges**, d’exploiter une vulnérabilité dans un driver signé pour lire et écrire dans la mémoire kernel. Ces primitives de lecture et d’écriture de la mémoire permettent d’énumérer les callbacks enregistrés dans le but de les supprimer. Le projet [**CheekyBlinder**](https://github.com/br-sn/CheekyBlinder) est un exemple qui exploite la vulnérabilité CVE-2019-16098.

La nouvelle tendance du moment est le terme **BYOVD** (Bring your own vulnerable driver). Il s’agit d’exploiter des vulnérabilités dans des pilotes signés afin de contourner les mécanismes de détection des EDR ou de les éteindre tout simplement.

Le projet **Terminator** ([https://github.com/ZeroMemoryEx/Terminator](https://github.com/ZeroMemoryEx/Terminator)) exploite le driver vulnérable *zam64.sys* et permet justement de tuer tous les processus des AV/EDR/XDR sur une machine. 

Il était initialement vendu 3000$ par un développeur et a été gratuitement reproduit par **[ZeroMemoryEx](https://github.com/ZeroMemoryEx/)** pour le plaisir de la communauté.

# Minifiltre

Selon Microsoft : “Un pilote de filtre de système de fichiers, ou minifiltre, intercepte les demandes ciblées sur un système de fichiers ou un autre pilote de filtre de système de fichiers.”

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-2/11-minifiltre.png" style="display:block;margin:auto;max-height:500px;width:auto">

**Ca veut dire quoi concrètement ?**

Les éditeurs peuvent développer des pilotes (**minifiltres**) qui surveillent les entrées / sorties de fichiers sur le système. 

Par exemple, lorsqu’on sauvegarde un fichier édité par l’application notepad.exe, un fichier est crée sur le système à un emplacement défini par l’utilisateur.

En réalité, l’application notepad va effectuer une requête d’entrée/sortie pour le fichier à sauvegarder. 

Cette requête est ensuite transférée au gestionnaire de filtre (**Filter Manager)** qui contient plusieurs minifiltres.

Chaque minifiltre effectue ses propres actions. Les minifiltres des éditeurs antivirus vont généralement scanner le contenu du fichier à créer et comparer la signature avec une base de données de signatures virales.

# ELAM (Early Launch Anti-Malware)

A partir de Windows 8 et Windows Server 2012, Microsoft a introduit la fonctionnalité **ELAM** et fournit des instructions permettant aux éditeurs de solutions de sécurité de **développer des pilotes qui sont initialisés avant d’autres pilotes de démarrage**. L’objectif est de garantir que les pilotes suivants ne contiennent pas de programmes malveillants.

Néanmoins, Microsoft exige que l’éditeur soit membre du programme **MVI** (Microsoft Virus Initiative) et qu’il soumette le driver au laboratoire de qualité matériel Windows (**WHQL**).

Comme le dit Microsoft, les AV/EDR deviennent meilleurs dans la détection des rootkits mais les attaquants progressent aussi en trouvant toujours de nouveaux moyens de contournement.

ELAM est lancé en premier par le kernel, avant tous les autres pilotes et est donc en mesure de détecter les malwares dans le processus de démarrage de la machine et de les empêcher d’être initialisés.

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-2/9-elam.png" style="display:block;margin:auto">

*Source : [https://www.anoopcnair.com/understanding-windows-trusted-boot/](https://www.anoopcnair.com/understanding-windows-trusted-boot/)*

Le driver ELAM développé par l’éditeur va vérifier la signature des autres drivers et la classifie en **good**, **bad** ou **unknown** selon une base de signatures malveillantes stockée dans le registre Windows (**HKLM\ELAM\<VendorName>**). Cette base n’est accessible que durant le démarrage de la machine pour des raisons de performance.

Pour analyser chaque pilote, le driver ELAM peut faire appel au kernel callbacks :

- [IoRegisterBootDriverCallback](https://learn.microsoft.com/fr-fr/windows-hardware/drivers/ddi/ntddk/nf-ntddk-ioregisterbootdrivercallback)
- [IoUnRegisterBootDriverCallback](https://learn.microsoft.com/fr-fr/windows-hardware/drivers/ddi/ntddk/nf-ntddk-iounregisterbootdrivercallback)

Ainsi, le driver ELAM sera notifié lors du chargement d’un pilote et des DLL qu’il utilise.

# Driver Signing

À partir de Windows 10, version 1607, Windows ne charge plus les nouveaux pilotes en mode noyau qui ne sont pas signés par le portail de développement.

Ci-dessous un tableau récapitulatif des stratégies de signature pour les versions du système d’exploitation client :

<img src="../../assets/images/posts/redteam/anatomie-edr-pt-2/10-driver.png" style="display:block;margin:auto;max-height:500px;width:auto">

# Conclusion

En résumé :

- Les maldevs et les éditeurs d’antivirus utilisaient des **drivers kernel** pour modifier le code du kernel durant son exécution mais ça rendait le système instable.
- Microsoft a sorti une fonctionnalité de sécurité appelée **Patchguard** (Kernel Patch Protection) empêchant le **kernel patching** (ou kernel hooking).
- À la place, Microsoft a fourni une alternative aux éditeurs d’antivirus : les **kernel callbacks**.
- Les EDR développent un ou plusieurs drivers qui enregistrent des kernel callbacks pour être notifiés de certains évènements sur le système (création d’un processus, thread, chargement d’image).
- Une fois notifiés, les fonctions enregistrées par les drivers sont exécutées, leur permettant d’interagir avec les nouveaux processus, threads, fichiers ou drivers par exemple.
- Les éditeurs peuvent utiliser des minifiltres pour surveiller les entrées/sorties de fichiers et vérifier leur signature.
- Microsoft a fournit la fonctionnalité **ELAM** permettant au kernel de charger le driver ELAM des éditeurs en premier afin de vérifier les signatures des drivers suivants.

Dans le prochain article, nous verrons comment fonctionnent les services et utilisateurs installés par les EDR ainsi que les mécanismes fournis par Microsoft pour les protéger.

# Références

- ***Windows Internals 7th Edition Parts 1 and 2 de Pavel Yosifovich (Auteur), Mark E. Russinovich, David A. Solomon, Alex Ionescu - 3 mai 2017***
- *Microsoft - User Mode et Kernel Mode : [https://learn.microsoft.com/fr-fr/windows-hardware/drivers/gettingstarted/user-mode-and-kernel-mode](https://learn.microsoft.com/fr-fr/windows-hardware/drivers/gettingstarted/user-mode-and-kernel-mode)*
- *WinDBG : Notifications Kernel - [https://blog.gentilkiwi.com/retro-ingenierie/windbg-notifications-kernel](https://blog.gentilkiwi.com/retro-ingenierie/windbg-notifications-kernel)*
- *Removing Kernel Callbacks using signed drivers* : [https://br-sn.github.io/Removing-Kernel-Callbacks-Using-Signed-Drivers/](https://br-sn.github.io/Removing-Kernel-Callbacks-Using-Signed-Drivers/)
- *Microsoft - Guide de conception des pilotes de systèmes de fichiers : [https://learn.microsoft.com/fr-fr/windows-hardware/drivers/ifs/](https://learn.microsoft.com/fr-fr/windows-hardware/drivers/ifs/)*
- *Windows ELAM : [https://learn.microsoft.com/fr-fr/windows-hardware/drivers/install/early-launch-antimalware](https://learn.microsoft.com/fr-fr/windows-hardware/drivers/install/early-launch-antimalware)*

# Remerciements

Je tiens à remercier [Atsika](https://twitter.com/_atsika), [Narek](https://twitter.com/0xnarek) et [RBCH](https://twitter.com/br00x__) pour la relecture et la correction de cet article.