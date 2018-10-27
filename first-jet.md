# Cloud

* un SI ? c'quuoi
    * "I" = information = data
    * SI = gestion de ces datas (création, distribution, stockage etc)
    * hard
        * réseau 
            * switches/routeurs/IPBX/PABX/FW/WiFi/Interco
        * virtu
        * stockage
            * scsi/iscsi
        * liens
            * classique, FC, interco etc
    * process soft
        * supervision 
            * sup hard (dispo réseau, cpu, ram, etc)
            * sup soft/applicative
            * métro réseau
            * tools :
                * logiciel dédié pour le réseau
                * tool commun pour stack applicative
        * backup
        * sécu
            * 5 points
            * sécu physique (porte, contrôle d'acccès, surveillance)
            * gestion de risques
    * HA hard & soft
    * PRA / PCA (= PRA++)

* cloud ? c koi ?
    * pas de déf ultime, terme tech et market et autres, donc... réfléchissons
    * => brainstorming ?
        * public privé hybride
    * qui ?
        * aws ggl azure
        * youtube IG slack discord
        * SSII : cheops, cis valley, etc
        * petit hébergeur online


* comment fonctionne le cloud ?
    * saas paas iaas etc
    * infra/resssources mutu
    * env homogènes
    * anecdote amazon black friday
        * migration dans le cloud ? aws snow


* VS on premises
    * opex/capex
    * cloud flexibilité du dimensionnement (réponse à la demande)
    * on premises : mises à jour (soft/hard) (réseau/système) très impactantes
    * on premises : équipes dédiées (maintenance, infra, chefs de projet etc)
    * on premises : need changer le matos (évolution rapide, un équip ~ 10 > 30 ans)

* éthique ?

* législation ?
    * RGPD
    * informatique & libertés

* mon cours ?
    * orienté système, provisionning, virtualisation, stockage, applicatif
    * stockage, fs distribué, bloc sur réseau, hyperconvergence, iscsi
    * provisionning hard, soft, et hybride : création de VM cloud/on-premise transparent
    * métier : déploiement de stacks applicative
    * organisation des process sup/monitoring/backup
    * un peu de sécu ?
    * un peu de réseau ?

  * but : gestion de SI en environnnement hybride
    * hard : gestion de réseaux/hyperviseurs
    * gestion du provisionning
    * déploiements applicatifs répétables et uniformes
    * infra immuables ?
