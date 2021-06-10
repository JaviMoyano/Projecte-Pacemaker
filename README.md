# Projecte Pacemaker edt 2020/2021

![Imatge Logo Pacemaker](imatges/Logo_Pacemaker.png)


## Objectius del projecte
L'objectiu és conéixer les funcionalitats de Pacemaker i d'altres tipus de software amb els que funciona conjuntament, per a dotar d'*Alta disponibilitat* a un clúster. Per demostrar el seu funcionament, aquest clúster estarà format per 3 nodes, i funcionarà com a un servidor que ofereix servei d'HTTP.

## Que és Pacemaker ?
Pacemaker és un tipus de software de control de recursos de clúster, que s'utilitza generalment per preservar la integritat de les dades i poder proporcionar un servei amb el mínim d'aturades possible. 
Algunes de les característiques i funcionalitats que pot atorgar al conjunt de hosts són: 

- Replicació de dades en els diferents nodes
- Protecció de les dades en cas de la caiguda d'algún node (redundància)
- Emmagatzemament compartit
- Actualització automàtica de les dades desde qualsevol dels nodes
- Generar condicions per al funcionament dels serveis que ofereix el clúster
    - Col·locació del servei a un node en concret (o que no s'engegi mai a )
    - Serveis que només s'engegen si ja hi ha un altre actiu
    - Ordre d'engegada dels serveis


## Clúster
Un **clúster** es un conjunt de nodes que funciona amb l'objectiu de portar a terme una o varies tasques com si fóssin un únic ordinador. Pot tenir diverses funcions, tals com:

- **Alt rendiment:** els diferents nodes poden treballar paral·lelament per a reduir el temps d'execució dels serveis i millorar el rendiment global del sistema.
- **Emmagatzemament:** permet allotjar tot el conjunt de dades als diferents nodes, el que ofereix la possibilitat de accedir a les dades del servidor a través de qualsevol dels ordinadors que composen el clúster. Aquesta característica la proporciona [GFS2](GFS2), conjuntament amb Pacemaker.
- **Balanç de càrregues:**: distribueix les sol·licituds que provenen de la xarxa entre els diferents nodes, per a que no hi hagi una sobrecàrrega en un d'aquests. Es redirigeixen les ordres a un altre node en cas de que algún quedi inoperatiu.
- **Alta disponibilitat:** proporciona un servei sense aturades en cas de que algún element del sistema caigués o fallés. 
Un cop detectat un error de hardware o de software, s'engega automàticament la mateixa aplicació o servei a un altre sistema/node sense necessitat de fer-ho manualment. Aquest procés es conegut com *failover*, que es tradueix com a *migració a causa d'error*. 

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


### Clúster Passive/Passive
![Imatge Cluster PA](imatges/actiupassiu.png)





### Clúster Active/Active
![Imatge Cluster AA](imatges/actiuactiu.png)

