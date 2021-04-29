## Pacemaker

Pacemaker es un tipus de software que otorga la característica d'alta disponibilitat a un conjunt de nodes. Proporciona unes funcions bàsiques a aquests nodes per treballar conjuntament com a clúster:


## Components del clúster
### Cluster Information Base (CIB)
És un dimoni que utilitza internament XML, per distribuir i sincronitzar la configuració actual i l'estat del Designated Coordinator (DC) - node assignat per Pacemaker per a detectar i distribuir l'estat del clúster - a tots els nodes del clúster.

### Cluster Resource Management Daemon (CRMd)
Dimoni que gestiona les accions relacionades amb els recursos del clúster. Aquests recursos es poden manipular, per exemple, generant instàncies a partir d'aquests.

Cada node inclou també un gestor dels recursos local (LMRd), que funciona com a interfície entre el dimoni CMRd i els recursos. LMRd passa les comandes de CMRd als agents, com per exemple, fer *start* o *stop*, o proporcionar informació sobre l'estat del node.

### Shoot the Other Node in the Head (STONITH)
És una funcionalitat del clúster que processa les sol·licituds de *fence*, forçant l'aturada de nodes que s'hagin corromput o que no funcionin correctament, i eliminant-los del node per a protegir l'integritat de les dades. Está configurat en CIB i pot ser monitoritzat com un recurs més del clúster.

## Corosync
Cluster Corosync es el component que s'encarrega de gestionar el sistema de comunicació, implementant
 alta disponibilitat entre aplicacions, és a dir, la transmissió de informació entre els
 nodes del clúster.

## DRBD
DRBD (Distributed Replicated Block Device) és un sistema d'emmagatzemament replicat i di
stribuit per a Linux. S'implement a nivell de driver de kernel, i ens permet replicar le
s ades a temps real entre els nodes del clúster.

