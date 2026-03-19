# De VMware à Hyper-V avec Windows Admin Center

Le rachat de VMware par Broadcom a agi comme un électrochoc. Aujourd'hui, la question n'est plus *faut-il migrer?*, mais *comment migrer sans tout casser ou y perdre tous ses cheveux?*. Entre les solutions tierces payantes et les méthodes manuelles risquées, Microsoft a discrètement sorti une arme redoutable : l'extension VM Conversion pour Windows Admin Center (WAC).

![](https://jlou.eu/wp-content/uploads/2026/03/image-20.png)

Après avoir testé l'outil encore en préversion, je voulais partager avec vous mon expérience. Et pour vous guider plus facilement dans cet article très long, voici des liens rapides :

- **FAQ**
  - [Pourquoi quitter VMware ?](#FAQ-I)
  - [Qu'est-ce que Windows Admin Center ?](#FAQ-II)
  - [Qu'est-ce que l'extension VM Conversion ?](#FAQ-III)
  - [Combien coûte VM Conversion ?](#FAQ-IV)
  - [Quelles versions d'OS sont supportées ?](#FAQ-V)
  - [Quels sont les prérequis pour VM Conversion ?](#FAQ-VI)
  - [Combien de VM puis-je migrer simultanément ?](#FAQ-VII)
  - [Comment l'outil VM Conversion marche ?](#FAQ-VIII)
  - [Quid du temps d'arrêt ?](#FAQ-IX)
  - [Que devient VMware Tools sur le VM migré ?](#FAQ-X)
- **Test de migration**
  - [Etape 0 – Rappel des prérequis](#Etape-0)
  - [Etape I – Installation de Windows Admin Center](#Etape-I)
  - [Etape II – Installation de prérequis WAC](#Etape-II)
  - [Etape III – Connexion à Hyper-V depuis WAC](#Etape-III)
  - [Etape IV – Test Windows : Synchronisation de la VM](#Etape-IV)
  - [Etape V – Test Windows : Migration de la VM](#Etape-V)
  - [Etape VI – Test Linux : Synchronisation de la VM](#Etape-VI)
  - [Etape VII – Test Linux : Migration de la VM](#Etape-VII)

---

<a id="FAQ-I"></a>
## Pourquoi quitter VMware ?

Le rachat de VMware par Broadcom a agi comme un électrochoc. Beaucoup de clients ont vu les coûts augmenter fortement après le rachat, car Broadcom a changé le modèle de licences (par exemple passage d'une licence perpétuelle à un modèle d'abonnement) et restructuré les offres. Certains ont reçu des renouvellements jusqu'à plusieurs fois plus chers qu'avant pour les mêmes besoins.

Ces changements rapides dans la structuration des produits, des bundles et du support ont créé de l'incertitude sur la roadmap et l'avenir des produits VMware, ce qui amène les DSI à repenser leurs choix technologiques à long terme

On retrouve d'ailleurs pas mal de blogueurs parlant de cet exode :

- [Why Leave VMware in 2025? Alternatives and Migration Guide](https://pandorafms.com/blog/why-leave-vmware-alternatives-2025/)
- [Navigating the VMware Exit: Why OpenStack is the Smart Alternative for 2025 and Beyond | OpenMetal IaaS](https://openmetal.io/resources/blog/navigating-the-vmware-exit/)
- [VMware : 6 options pour en sortir, 1 piste pour y rester](https://www.cio-online.com/actualites/lire-vmware-6-options-pour-en-sortir-1-piste-pour-y-rester-16231.html)
- [Life After VMware: Accelerating VMware Exits with In-Place Migration • Platform9](https://platform9.com/blog/life-after-vmware-accelerating-vmware-exits-with-in-place-migration/)
- [Hyper-V 2026: Not the Same Beast You Ditched in 2012 (And That's a Good Thing) | by Mr.PlanB | Jan, 2026 | Medium](https://medium.com/@PlanB./hyper-v-2026-not-the-same-beast-you-ditched-in-2012-and-thats-a-good-thing-5522709a5c7c)
- [Ditching Broadcom, Hyper-V Server, & Live Migrations | by Rich | Medium](https://happycamper84.medium.com/ditching-broadcom-hyper-v-server-live-migrations-3b41cf2a8830)

<a id="FAQ-II"></a>
## Qu'est-ce que Windows Admin Center ?

Présent depuis des années, Windows Admin Center (souvent abrégé WAC) est un outil d'administration web développé par Microsoft pour gérer des serveurs Windows, des clusters et des environnements hyperconvergés, depuis une interface moderne accessible via navigateur.

![](https://jlou.eu/wp-content/uploads/2026/02/image-403-1024x576.png)

Concrètement, Windows Admin Center vous permet d'administrer, sans passer par du RDP, un grand nombre de services Microsoft :

- Windows Server
- Hyper-V
- Clusters (Failover Clustering)
- Machines virtuelles
- Serveurs distants (on-prem ou Azure)
- Stockage (Storage Spaces Direct)

https://youtu.be/wT2ps\_71VKU

<a id="FAQ-III"></a>
## Qu'est-ce que l'extension VM Conversion ?

C'est un nouvel outil Microsoft intégré à Windows Admin Center qui permet de migrer des machines virtuelles depuis VMware vCenter/ESXi vers Hyper-V, en mode agentless :

![](https://jlou.eu/wp-content/uploads/2026/02/image-400-1024x428.png)

Actuellement, Il s'agit d'une extension prévue pour minimiser le temps d'arrêt grâce à la réplication en ligne des disques.

Attention, cette extension n'est pas pour but de migrer vers Azure Local. Un autre article déjà écrit il y a plusieurs mois traite de ce type de migration : [de VMware à Azure Local](https://jlou.eu/vmwareazurestackhci/).

<a id="FAQ-IV"></a>
## Combien coûte VM Conversion ?

L'extension est actuellement en préversion. Son usage n'implique aucun coût de licence supplémentaire, mais pas non plus de support garanti par Microsoft.

<a id="FAQ-V"></a>
## Quelles versions d'OS sont supportées ?

Microsoft annonce [une large liste](https://learn.microsoft.com/en-us/windows-server/manage/windows-admin-center/use/vm-conversion-extension-overview#vcenter-versions-and-guest-operating-systems) sur la compatibilité :

- Tous les vCenter VMware 6.x, 7.x et 8.x sont pris en charge
- Côté OS invités :
  - Windows Server 2025, 2022, 2019, 2016, 2012 R2
  - Windows 10/11
  - Ubuntu 20.04, 24.04
  - Debian 11, 12
  - Alma Linux
  - CentOS
  - Red Hat Linux 9.0

L'extension fonctionne également dans un environnement **Hyper-V en cluster (WSFC)** : elle est cluster-aware et détecte correctement les nœuds et ressources.

Il supporte la migration de VM depuis ESXi vers des clusters Windows Server Failover (WSFC) sous Hyper-V. Vous pouvez donc distribuer les VM migrées sur plusieurs nœuds Hyper-V pour HA ou performances.

![](https://jlou.eu/wp-content/uploads/2026/02/image-405-1024x683.png)

<a id="FAQ-VI"></a>
## Quels sont les prérequis pour VM Conversion ?

- Côté Windows Admin Center :
  - WAC en version au moins v2410 build 2.4.12.10 ou supérieure
  - PowerShell et PowerCLI installés
  - VMware VDDK version 8.0.3 installé
  - Visual Studio 2013 et 2015 installés
- Côté Hyper-V :
  - le rôle Hyper-V doit être installé sur le (s) hôte (s) cible
  - Le compte utilisé durant le processus doit être administrateur local ou membre du groupe Hyper-V Administrators).
  - Si des VMs migrées sont sous Linux, les modules Hyper-V (hv\_vmbus, hv\_netvsc, hv\_storvsc) et Linux Integration Services (hyperv-daemons ou linux-cloud-tools) doivent être pré-installés dans l'initramfs avant migration.

<a id="FAQ-VII"></a>
## Combien de VM puis-je migrer simultanément ?

Jusqu'à 10 machines virtuelles par lot. On peut grouper ces machines selon leur dépendance applicative, leur placement dans un cluster ou même selon des critères métier (ex : séparer environnement de test/prod). Cela facilite les migrations massives par workload.

<a id="FAQ-VIII"></a>
## Comment l'outil VM Conversion marche ?

Microsoft explique bien les deux phases présentes dans l'outil VM Conversion :

> **Synchronisation** : l'extension effectue une copie complète initiale des disques de la machine virtuelle pendant que la VM source continue de fonctionner. Cette phase minimise les temps d'arrêt en vous permettant de planifier la migration finale à un moment qui vous convient.
>
> **Migration** : l'extension utilise la fonctionnalité Change Block Tracking (CBT) pour rechercher et répliquer uniquement les blocs modifiés depuis la dernière synchronisation. Pendant la transition, la machine virtuelle source est mise hors tension et une synchronisation delta finale capture toutes les modifications restantes avant d'importer la machine virtuelle dans Hyper-V.
>
> [Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/manage/windows-admin-center/use/vm-conversion-extension-overview#how-it-works)

![](https://jlou.eu/wp-content/uploads/2026/02/image-407.png)

<a id="FAQ-IX"></a>
## Quid du temps d'arrêt ?

Pendant la phase de synchronisation, la VM source continue de tourner, pendant qu'une copie initiale des disques est envoyée sur l'hôte Hyper-V, via la création d'un snapshot pour suivre les changements.

Au moment de la bascule, la VM source est arrêtée pour un dernier delta-sync avant import; ce processus réduit donc au minimum la fenêtre d'arrêt.

<a id="FAQ-X"></a>
## Que devient VMware Tools sur le VM migré ?

La présence de VMware Tools dans une VM sous Hyper-V peut provoquer des conflits de pilotes, donc il vaut mieux les retirer.

- Initialement, il fallait supprimer les outils VMware Tools manuellement après la bascule vers Hyper-V.
- Depuis la [version 1.8.0](https://learn.microsoft.com/en-us/windows-server/manage/windows-admin-center/use/whats-new-vm-conversion-extension), l'extension supprime automatiquement VMware Tools sur les VM Windows migrées en fin de processus.
- Pour les VM Linux, il est recommandé de ne pas réinstaller open-vm-tools sur la cible Hyper-V (on s'appuie uniquement sur les pilotes Hyper-V ajoutés).

![](https://jlou.eu/wp-content/uploads/2026/03/image-18.png)

Et enfin, comme toujours, je vous propose dans la suite de cet article d'effectuer ensemble un pas à pas pour couvrir la migration de machines virtuelles hébergées sur VMware vers Hyper-V :

- [Etape 0 – Rappel des prérequis](#Etape-0)
- [Etape I – Installation de Windows Admin Center](#Etape-I)
- [Etape II – Installation de prérequis WAC](#Etape-II)
- [Etape III – Connexion à Hyper-V depuis WAC](#Etape-III)
- [Etape IV – Test Windows : Synchronisation de la VM](#Etape-IV)
- [Etape V – Test Windows : Migration de la VM](#Etape-V)
- [Etape VI – Test Linux : Synchronisation de la VM](#Etape-VI)
- [Etape VII – Test Linux : Migration de la VM](#Etape-VII)

---

<a id="Etape-0"></a>
## Etape 0 – Rappel des prérequis

Des prérequis sont nécessaires pour réaliser cet exercice dédié à la migration d'une machine virtuelle hébergée sur VMware vers Hyper-V via Windows Admin Center. Pour tout cela, j'ai utilisé :

- Un environnement VMware
- Un environnement Hyper-V
- Une connexion réseau entre le réseau les deux hyperviseurs

Commençons par l'installation de Windows Admin Center sur une nouvelle machine hébergée sur VMware.

<a id="Etape-I"></a>
## Etape I – Installation de Windows Admin Center

Sur une machine virtuelle dédiée dans votre environnement VMware, téléchargez la dernière version de Windows Admin Center depuis la [page officielle de Microsoft](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-admin-center) :

![](https://jlou.eu/wp-content/uploads/2026/03/image-1-1024x401.png)

Choisissez l'installation avec l'option Express :

![](https://jlou.eu/wp-content/uploads/2026/03/image-3.png)

Dans cette démonstration, utilisez un certificat temporaire :

![](https://jlou.eu/wp-content/uploads/2026/03/image-5.png)

Lancez l'installation :

[![](https://jlou.eu/wp-content/uploads/2026/02/image-319.png)](https://jlou.eu/?attachment_id=24024)

Une fois l'installation terminée, ouvrez la page de Windows Admin Center, puis authentifiez-vous avec votre compte :

![](https://jlou.eu/wp-content/uploads/2026/02/image-326.png)

Une fois connecté dans la console Windows Admin Center, attendez quelques minutes la fin de l'installation des extensions préinstallées :

![](https://jlou.eu/wp-content/uploads/2026/02/image-328-1024x555.png)

Rendez vous dans les paramétrages dans WAC afin d'ajouter l'extension dédiée à la migration :

![](https://jlou.eu/wp-content/uploads/2026/02/image-329-1024x644.png)

<a id="Etape-II"></a>
## Etape II – Installation de prérequis WAC

Comme indiqué dans la documentation Microsoft, certains prérequis doivent être installés sur la machine de Windows Admin Center.

Commencez par installer [Visual C++ Redistributable Packages for Visual Studio 2013](https://www.microsoft.com/en-us/download/details.aspx?id=40784&msockid=3d75fb163d8768940e5fedff3c3f690d)

![](https://jlou.eu/wp-content/uploads/2026/02/image-320.png)

Continuez en installant également [Visual C++ 2015-2022 Redistributable 14.50.35719.0](https://aka.ms/vc14/vc_redist.x64.exe) :

![](https://jlou.eu/wp-content/uploads/2026/02/image-321.png)

Récupérez et décompressez **VMware Virtual Disk Development Kit (VDDK),** en version 8.0.3, dans le dossier WAC suivant :

![](https://jlou.eu/wp-content/uploads/2026/02/image-336-1024x767.png)

Redémarrez ensuite votre machine virtuelle contenant Windows Admin Center.

<a id="Etape-III"></a>
## Etape III – Connexion à Hyper-V depuis WAC

Une fois la machine virtuelle WAC redémarrée, rouvrez la console, puis cliquez ici pour ajouter une connexion vers votre serveur Hyper-V :

![](https://jlou.eu/wp-content/uploads/2026/02/image-330-1024x168.png)

Cliquez sur **Ajouter** :

![](https://jlou.eu/wp-content/uploads/2026/03/image-8.png)

Renseignez l'**adresse IP** de votre Hyper-V, les éléments d'identification, puis cliquez sur **Ajouter** :

![](https://jlou.eu/wp-content/uploads/2026/03/image-10.png)

La connexion est établie, cliquez dessus pour l'ouvrir :

![](https://jlou.eu/wp-content/uploads/2026/02/image-333-1024x175.png)

Dans le menu de gauche, cherchez l'extension consacrée à la migration, cochez la case suivante pour installer **PowerCLI**, puis cliquez ici :

![](https://jlou.eu/wp-content/uploads/2026/02/image-334-1024x689.png)

Attendez la fin de l'installation de PowerCLI :

![](https://jlou.eu/wp-content/uploads/2026/02/image-335-1024x640.png)

Une fois l'installation terminée, cliquez ici ajouter une connexion à votre **vCenter**, renseignez vos informations de connexion, puis cliquez ici :

![](https://jlou.eu/wp-content/uploads/2026/02/image-337-1024x639.png)

Une fois la connexion établie avec **vCenter**, les machines virtuelles s'afficheront :

![](https://jlou.eu/wp-content/uploads/2026/02/image-338-1024x641.png)

Le protocole de test est maintenant en place. Commençons les tests par la migration d'une machine virtuelle fonctionnant sous Windows Server.

<a id="Etape-IV"></a>
## Etape IV – Test Windows : Synchronisation de la VM

Toujours dans la liste des machines virtuelles, cochez sur une des VMs Windows disponibles, puis cliquez ici pour démarrer la synchronisation :

![](https://jlou.eu/wp-content/uploads/2026/02/image-339-1024x397.png)

Renseignez le dossier de destination sur votre serveur Hyper-V, puis cliquez sur Synchroniser :

![](https://jlou.eu/wp-content/uploads/2026/03/image-14.png)

La phase de vérification préalable à la migration vient de démarrer (accessibilité, compatibilité matérielle, compatibilité OS, configuration réseau, ...) :

![](https://jlou.eu/wp-content/uploads/2026/02/image-341-1024x310.png)

La première copie des disques VMDK vers Hyper-V est en cours pour la création des fichiers VHDX par la suite :

![](https://jlou.eu/wp-content/uploads/2026/02/image-343-1024x325.png)

Constatez d'ailleurs la création du dossier sur le serveur Hyper-V :

![](https://jlou.eu/wp-content/uploads/2026/02/image-344-1024x544.png)

La migration passe en mode synchronisation différentielle afin de ne copier que les blocs modifiés depuis la synchronisation initiale, réduisant ainsi le volume de données à transférer avant le cutover :

![](https://jlou.eu/wp-content/uploads/2026/02/image-345-1024x396.png)

Provisioning du disque cible Hyper-V (VHDX) : l'infrastructure prépare le stockage avant l'application des blocs synchronisés issus de la VM VMware :

![](https://jlou.eu/wp-content/uploads/2026/02/image-346-1024x417.png)

Phase de delta sync : les blocs modifiés identifiés via CBT sont appliqués au disque Hyper-V afin d'aligner la VM cible avec l'état courant de la VM source :

![](https://jlou.eu/wp-content/uploads/2026/02/image-347-1024x412.png)

Le monitoring de la carte réseau montre bien le transfert des données :

![](https://jlou.eu/wp-content/uploads/2026/02/image-348-1024x776.png)

Synchronisation complète (100 %) : l'intégralité des données de la VM source est maintenant répliquée côté Hyper-V :

![](https://jlou.eu/wp-content/uploads/2026/02/image-349-1024x292.png)

La VM cible est maintenant entièrement alignée avec la VM VMware et peut être basculée vers Hyper-V avec un dernier delta minimal.

Nous allons pouvoir procéder à la migration.

<a id="Etape-V"></a>
## Etape V – Test Windows : Migration de la VM

A ce stade, la machine virtuelle sous VMware est toujours allumée :

![](https://jlou.eu/wp-content/uploads/2026/02/image-351-1024x429.png)

Testez le delta de migration en rajoutant sur votre machine virtuelle source de nouveaux fichiers :

![](https://jlou.eu/wp-content/uploads/2026/02/image-356-1024x763.png)

Retournez ensuite sur Windows Admin Center afin de lancer la migration de celle-ci :

![](https://jlou.eu/wp-content/uploads/2026/02/image-363-1024x259.png)

Choisissez ou non de désinstaller les outils VMware, puis cliquez ici :

![](https://jlou.eu/wp-content/uploads/2026/03/image-12.png)

Lancement de la migration : exécution des vérifications préalables avant bascule définitive vers la VM Hyper-V cible :

![](https://jlou.eu/wp-content/uploads/2026/02/image-364-1024x289.png)

Vérifications pré-migration terminées : la synchronisation finale des blocs modifiés (delta sync) est en cours avant l'arrêt et la bascule de la VM :

![](https://jlou.eu/wp-content/uploads/2026/02/image-359-1024x247.png)

Un snapshot est bien créé côté VMware :

![](https://jlou.eu/wp-content/uploads/2026/02/image-367-1024x313.png)

Delta Sync en cours : application des derniers blocs modifiés afin d'aligner définitivement la VM cible avant le cutover final vers Hyper-V.

![](https://jlou.eu/wp-content/uploads/2026/02/image-365-1024x287.png)

Arrêt de la VM source : la machine VMware est éteinte et la phase finale de synchronisation est lancée avant le démarrage côté Hyper-V :

![](https://jlou.eu/wp-content/uploads/2026/02/image-368-1024x282.png)

La machine virtuelle est bien arrêtée côté VMware :

![](https://jlou.eu/wp-content/uploads/2026/02/image-369-1024x427.png)

Migration terminée (100 %) : la VM cible Hyper-V est créée, synchronisée et prête à être démarrée en production :

![](https://jlou.eu/wp-content/uploads/2026/02/image-370-1024x293.png)

La VM migrée s'exécute désormais sur Hyper-V, confirmant le succès du cutover depuis l'environnement VMware :

![](https://jlou.eu/wp-content/uploads/2026/02/image-371-1024x609.png)

Windows détecte un arrêt inattendu lié au cutover, confirmant que la VM a bien été basculée depuis l'environnement VMware vers Hyper-V :

![](https://jlou.eu/wp-content/uploads/2026/02/image-372.png)

Erreur VMware Tools au premier démarrage : les anciens composants VMware, désormais incompatibles sous Hyper-V, doivent être désinstallés après la migration :

![](https://jlou.eu/wp-content/uploads/2026/02/image-373.png)

Désinstallez les VMware Tools devenus inutiles après le passage de la VM sous Hyper-V :

![](https://jlou.eu/wp-content/uploads/2026/02/image-374.png)

Les fichiers créés avant la commande Migrate sont bien présents après la bascule, confirmant l'intégrité de la synchronisation finale :

![](https://jlou.eu/wp-content/uploads/2026/02/image-375.png)

Notre serveur Windows a bien été migré depuis VMware vers Hyper-V avec succès. Testons maintenant la même opération avec un serveur Linux.

<a id="Etape-VI"></a>
## Etape VI – Test Linux : Synchronisation de la VM

Exemple réalisé ici sur AlmaLinux / RHEL-like. Le but étant de :

- Désinstaller VMware Tools
- Installer composants Hyper-V
- Recréer initramfs si nécessaire
- Reboot

Pour cela créez une machine virtuelle Linux dont l'OS est compatible avec l'outil de Migration :

![](https://jlou.eu/wp-content/uploads/2026/02/image-376-1024x783.png)

Avant la migration, ajoutez les pilotes Hyper-V à l'initramfs, reconstruction avec **dracut**, puis redémarrage pour assurer un démarrage correct sous Hyper-V.

```bash
echo 'add_drivers+=" hv_vmbus hv_storvsc hv_netvsc "' | sudo tee /etc/dracut.conf.d/hyperv.conf
sudo dracut -f --regenerate-all
sudo reboot
```

Toujours avant la migration, installez des services d'intégration Hyper-V (hyperv-daemons) afin d'optimiser les performances et l'interaction entre la VM Linux et l'hôte Hyper-V.

Exemple réalisé sur une distribution RHEL-like (AlmaLinux / Rocky / RHEL). (Adaptez la commande si vous êtes sur Ubuntu ou Debian) :

```bash
sudo dnf install hyperv-daemons -y
```

![](https://jlou.eu/wp-content/uploads/2026/02/image-377-1024x691.png)

Vérifiez le chargement des modules Hyper-V (hv\_\*) dans le noyau Linux afin de confirmer la bonne prise en charge de l'environnement Hyper-V :

![](https://jlou.eu/wp-content/uploads/2026/02/image-378.png)

Toujours dans la liste des machines virtuelles, cochez sur une des VMs Linux disponibles, puis cliquez ici pour démarrer la synchronisation :

![](https://jlou.eu/wp-content/uploads/2026/02/image-379-1024x270.png)

Renseignez le dossier de destination sur votre serveur Hyper-V, puis cliquez sur Synchroniser :

![](https://jlou.eu/wp-content/uploads/2026/03/image-16.png)

La phase de vérification préalable à la migration vient de démarrer (accessibilité, compatibilité matérielle, compatibilité OS, configuration réseau, …) :

![](https://jlou.eu/wp-content/uploads/2026/02/image-381-1024x284.png)

La première copie des disques VMDK vers Hyper-V est en cours pour la création des fichiers VHDX par la suite :

![](https://jlou.eu/wp-content/uploads/2026/02/image-382-1024x278.png)

Constatez d'ailleurs la création du dossier sur le serveur Hyper-V :

![](https://jlou.eu/wp-content/uploads/2026/02/image-383.png)

La migration passe en mode synchronisation différentielle afin de ne copier que les blocs modifiés depuis la synchronisation initiale, réduisant ainsi le volume de données à transférer avant le cutover :

![](https://jlou.eu/wp-content/uploads/2026/02/image-385-1024x288.png)

Phase de delta sync : les blocs modifiés identifiés via CBT sont appliqués au disque Hyper-V afin d'aligner la VM cible avec l'état courant de la VM source :

![](https://jlou.eu/wp-content/uploads/2026/02/image-387-1024x277.png)

Le monitoring de la carte réseau montre bien le transfert des données :

![](https://jlou.eu/wp-content/uploads/2026/02/image-388.png)

Synchronisation complète (100 %) : l'intégralité des données de la VM source est maintenant répliquée côté Hyper-V :

![](https://jlou.eu/wp-content/uploads/2026/02/image-389-1024x284.png)

La VM cible est maintenant entièrement alignée avec la VM VMware et peut être basculée vers Hyper-V avec un dernier delta minimal.

Nous allons pouvoir procéder à la migration.

<a id="Etape-VII"></a>
## Etape VII – Test Linux : Migration de la VM

Retournez ensuite sur Windows Admin Center afin de lancer la migration de celle-ci :

![](https://jlou.eu/wp-content/uploads/2026/02/image-394-1024x311.png)

Lancement de la migration : exécution des vérifications préalables avant bascule définitive vers la VM Hyper-V cible :

![](https://jlou.eu/wp-content/uploads/2026/02/image-395-1024x301.png)

Arrêt de la VM source : la machine VMware est éteinte et la phase finale de synchronisation est lancée avant le démarrage côté Hyper-V :

![](https://jlou.eu/wp-content/uploads/2026/02/image-396-1024x303.png)

La machine virtuelle est bien arrêtée côté VMware :

![](https://jlou.eu/wp-content/uploads/2026/02/image-397-1024x796.png)

Migration terminée (100 %) : la VM cible Hyper-V est créée, synchronisée et prête à être démarrée en production :

![](https://jlou.eu/wp-content/uploads/2026/02/image-398-1024x302.png)

La VM migrée s'exécute désormais sur Hyper-V, confirmant le succès du cutover depuis l'environnement VMware :

![](https://jlou.eu/wp-content/uploads/2026/02/image-401-1024x727.png)

---

## Conclusion

La migration d'un environnement VMware vers Hyper-V n'est jamais une décision anodine. Elle est souvent motivée par des enjeux économiques, stratégiques ou organisationnels. Mais quelle qu'en soit la raison, elle doit rester avant tout une opération maîtrisée, structurée et techniquement fiable.

L'extension **VM Conversion** de Windows Admin Center n'est pas un simple assistant graphique. Elle propose une approche orchestrée et cohérente : synchronisation initiale, delta via CBT, puis phase de bascule contrôlée. Ce n'est pas "magique" et cela ne dispense pas d'une préparation sérieuse. En revanche, l'outil apporte un cadre clair qui simplifie fortement un processus historiquement complexe.

Dans mes tests, l'expérience s'est révélée stable et lisible. La gestion des clusters Hyper-V (WSFC), la logique de synchronisation progressive et l'interface unifiée dans WAC rendent la migration bien plus accessible qu'avec des méthodes artisanales. Cela reste une extension en preview, donc à évaluer sérieusement en environnement de test avant toute utilisation en production, mais la base technique est solide.

Ce qui est certain, c'est qu'une alternative crédible existe désormais pour les organisations qui souhaitent diversifier ou repositionner leur stratégie d'hyperviseur. La migration ne doit plus être perçue comme un saut dans l'inconnu, mais comme un projet structuré, pilotable et documenté.
