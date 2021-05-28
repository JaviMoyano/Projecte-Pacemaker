# Projecte Pacemaker edt 2020/2021

![Imatge Logo Pacemaker](cluster/Logo_Pacemaker.png)


## Objectius del projecte
L'objectiu és conéixer les funcionalitats de Pacemaker i d'altres tipus de software amb els que funciona conjuntament, per a dotar d'*Alta disponibilitat* a un clúster. Per demostrar el seu funcionament, aquest clúster estarà format per 3 nodes, i funcionarà com a un servidor que ofereix servei d'HTTP.

## Que és Pacemaker ?
Pacemaker és un tipus de software de control de recursos de clúster, que pot proporcionar diferents característiques al conjunt de hosts. 
S'utilitza generalment amb [Corosync] per . 

- Replicació de dades en els diferents nodes
- Protecció i replicació de dades en cas de la caiguda d'algún node (redundància)


## Clúster
Un **clúster** es un conjunt de dos o més nodes que funcionen amb l'objectiu de portar a terme una o varies tasques. Aquestes funcions poden ser: 
- **Emmagatzemament:** 
- **Alta disponibilitat:** proporciona un servei sense aturades en cas de que algún element del sistema es caigués o fallés. 
Un cop detectat un error de hardware o de software, s'engega automàticament la mateixa aplicació o servei a un altre sistema/node sense necessitat de fer-ho manualment. Aquest procés es conegut com *failover*, que es tradueix com a *migració a causa d'error*.
- **Balanç de càrregues:** 
- **Alt rendiment:** 

### Components
#### Cluster Information Base (CIB)
És un dimoni que utilitza XML internament que té la funció de distribuir i sincronitzar la configuració actual, i l'estat del *Designated Coordinator*(DC) - node assignat per Pacemaker per a detectar i distribuir l'estat del clúster - a tots els nodes.

#### Cluster Resource Management Daemon (CRMd)
Dimoni que gestiona les accions relacionades amb els recursos del clúster. Aquests recursos es poden manipular, per exemple, generant instàncies a partir d'aquests.

Cada node inclou també un gestor dels recursos local (LMRd), que funciona com a interfície entre el dimoni CMRd i els recursos. LMRd passa les comandes de CMRd als agents, com per exemple, fer *start* o *stop*, o proporcionar informació sobre l'estat del node.

#### Shoot the Other Node in the Head (STONITH)
És una funcionalitat del clúster que processa les sol·licituds de *fence*, forçant l'aturada de nodes que s'hagin corromput o que no funcionin correctament, i eliminant-los del node per a protegir l'integritat de les dades. Está configurat en CIB i pot ser monitoritzat com un recurs més del clúster.

## Corosync
Cluster Corosync es el component que s'encarrega de gestionar el sistema de comunicació, implementant
 alta disponibilitat entre aplicacions, és a dir, la transmissió de informació entre els
 nodes del clúster.

## DRBD
DRBD (Distributed Replicated Block Device) és un sistema d'emmagatzemament replicat i di
stribuit per a Linux. S'implement a nivell de driver de kernel, i ens permet replicar le
s ades a temps real entre els nodes del clúster.


### Corosync
Proporciona al nodes la capacitat de formar part del clúster, així com de comunicar-se entre ells. També pot habilitar al clúster per fer quorum i atorgar una eina més per a una millor protecció de les dades.