# Procediment pràctic

## Objectiu
Muntaré un clúster d'alta disponibilitat de dos nodes amb Fedora, que ofereixi el servei HTTP de forma continuada. En cas de caiguda d'un dels dos nodes, l'altre node seguirà oferint aquest servei. 
El contingut de el document html que oferirà el nostre servei, estarà sincronitzat en ambdós nodes, és a dir, si ho modifiquem a un dispositiu, el contingut es replicarà a l'altre.

En aquest cas, he utilitzat com a nodes dos ordinadors de l'aula, i02 i i26. Generalment s'aconsella treballar desde un sol dispositiu, fent ús de la connexió ssh per a configurar els demés nodes.


## Procediment per crear un clúster

Primer de tot, ens tenim que assegurar que els ordinadors on treballarem permetin l'**alta disponibilitat** per treballar amb el clúster.
Podem obrir manualment els ports TCP 2224, 3121, 21064, i el port UDP 5405, a la vegada que deshabilitem el Selinux.
L'altra opció es habilitar-ho desde el servei del comandes de firewall:

```
firewall-cmd --permanent --add-service=high-availability
firewall-cmd --reload
```

Es necessari que totes les comandes per a la configuració del clúster, dels nodes, instal·lacions, i, fins i tot, consultes de l'estat dels recursos de l'estructura, es realitzin com a usuari *root* o administrador.
Instal·lem el programari necessari a tots els nodes: 

```
dnf -y install pacemaker pcs psmisc corosync drbd gfs2-utils
```

Engeguem el servei **pcsd** per a poder generar i configurar el clúster, i també habilitem que s'engegui quan fem boot de l'ordinador, per a agilitzar el procés.

```
systemctl start pcsd
systemctl enable pcsd
```

Seguidament, tindrem que ficar una contrasenya a l'usuari **hacluster**, ja que serà necessari per a sincronitzar la configucació de corosync, o permetre el control dels nodes desde els altres nodes del clúster.
Es convenient utilitzar la mateixa contrasenya a tots els usuaris hacluster dels diferents nodes.

```
passwd hacluster
```

### Configuració de Corosync

Ens autentiquem a tots els nodes amb l'usuari esmentat per a poder començar el procés, i per a que posteriorment puguem crear el cluster desde un sol node.
```
pcs host auth i02 i26

pcs cluster setup mycluster i02 i26 (si no funciona, assegurar-se que la contrasenya de hacluster es la mateixa a tots els nodes)
```


### Engegada del clúster

Un cop hem configurat Corosync, podem engegar tots els nodes del clúster a la vegada, o de forma separada:

```
pcs cluster start --all (engegar tots els nodes del clúster)
pcs cluster start i02 i26 (múltiples nodes desde qualsevol node)
pcs cluster start (desde el node que es pretén engegar)
```

Podem comprovar que Corosync funciona correctament amb l'eina *corosync-cfgtool*. Si dóna un missatge d'error, tindrem que repassar la configuració del firewall, selinux o de la xarxa.
També podem comprovar els membres del clúster i el quorum amb:

```
pcs status corosync
pcs status
```

En qualsevol moment del procés podem utilitzar *pcs cluster cib* per a veure l'estat actual de la estructura en format XML. Es pot utilitzar per a fer "backups" dels estats a mesura que anem seguint els passos.
Es poden modificar certs aspectes de la configuració del clúster desde aquest document, pero es millor fer-ho per comandes per si de cas, ja que si cometem un error de sintaxi, no arrencarà.


### Configuració del fencing 

Escollim el dispositiu de fencing que s'adapti millor a les nostres necessitats, i posteriorment configurem el *fence agent* necessari per a configurar el dispositiu.
Veiem la llista de **fence agents**:

```
pcs stonith list
```

Generem una copia local del CiB per a treballar sobre aquesta, i, un cop escollit el mecanisme d'stonith corresponent, creem el recurs de fencing. 
Després habilitem l'ús de stonith al clúster, i finalment fem un *push* del CIB:
```
pcs cluster cib stonith_cfg

pcs -f stonith_cfg stonith create stonith_id disposotiu_stonith [opcions_stonith]
pcs -f stonith_cfg property set stonith-enabled=true

pcs cluster cib-push stonith_cfg
```


## Clúster Actiu/Passiu

Començarem afegint un recurs, que en aquest cas serà una **IP flotant** que podrà ser assignada a qualsevol node. Aquesta adreça haurà de estar disponible per a que puguin contactar els usuaris amb ella.
Li ficarem el nom de ClusterIP i demanarem que el clúster validi cada 30 segons que està funcionant:

```
pcs resource create ClusterIP ocf:heartbeat:IPAddr2 ip=192.168.122.202 cidr_netmask=24 op monitor interval=30s

ocf -> estàndar
heartbeat -> proveidor de recursos ocf
IPaddr2 -> resource agent per a un OCF provider
```

Comprovem amb *pcs status* que el recurs ha sigut afegit al clúster, o amb *pcs resource show CLusterIP*.

### Testejant un failover

Per a comprovar si el sistema de **failover** funciona al nostre clúster. Aturarem el node on està engegat el recurs ClusterIP, i observarem com aquest es mou al node que quedarà actiu.
```
pcs cluster stop i02
```
Si observem l'estat del clúster desde i26, veurem que indica a i02 com a OFFLINE.
Cal comentar que gràcies a Corosync, als clústers de dos nodes, només es demana que hi sigui un node actiu per a que hi hagi quorum, sino sempre s'aturaria el clúster al apagar-se un node.
Aquesta característica es pot observar al *corosync.conf**, on hi ha activada la opció **two_node: 1**.

Si tornem a engegar el node i02, el recurs ClusterIP es tornarà a situar al node i02 desde i26.

Aquest transport de recursos entre nodes, requereix un temps per petit que sigui, que augmenta quan es tracten bases de dades, per exemple. 
En general, es preferible que aquests recursos siguin engegats on hi eren prèviament. 
El concepte **stickiness**, controla en quina mida preferim que el servei romangui sigui engegat on ja hi és. 
Per defecte, Pacemaker assumeix que no hi ha cap tipus de cost en moure els recursos entre nodes, per això la existencia de aquest concepte.
```
pcs resource defaults resource-stickiness=100
pcs resource defaults
```
Ara, si apaguem i02, i ClusterIP es mou a i26, un cop tornem a engegar i02, el recurs es quedarà a i26.



## Afegim Apache com a servei de clúster

Necessitem apache instal·lat a tots els nodes, però **no s'ha de engegar**. Els serveis que s'han de administrar mitjançant el software del cluster, mai han de ser administrats pel sistema operatiu.

Per començar, creem el mateix html als dos nodes.

Amb l'objectiu de monitoritzar el funcionament de la instancia d'Apache, i recuperar-la si falla, el agent de recursos utilitzat per Pacemaker considera que la URL status-server està disponible. En els dos nodes, tenim que habilitar la URL amb:
```
*/etc/htpd/conf.d/status.conf*
<Location /server-status>
        SetHandler server-status
        Order deny,allow
        Deny from all
        Allow from 127.0.0.1
</Location>
```


Seguidament, per afegir apache al clúster, crearem un recurs anomenat **WebSite**.
L'únic paràmetre requerit per l'script es el fitxer de configuració de Apache. Cada minut checkejará si el Apache segueix engegat.
```
pcs resource create WebSite ocf:heartbeat:apache configfile=/etc/httpd/conf/httpd.conf statusurl="http://localhost/server-status" op monitor interval=60s
```

Un cop afegit el recurs, hauriem de veure'l engegat a i02 amb **pcs status**.


### Funcionament dels recursos a un mateix node (colocation constraints)

Pacemaker tracta de distribuir automàticament els recursos entre tots els nodes del clúster. Tot i així, es pot indicar que alguns recursos **han de funcionar conjuntament al mateix host**, o al contrari, que no tenen que estar engegats al mateix host.

Per aconseguir aixó, es poden ficar restriccions de ubicació o *colocation constraints*. Aquestes restriccions son "direccionals", és a dir, depenent de l'ordre en que les escrivim, els recursos tindràn una col·locació o un altra, però no té res a veure amb l'ordre d'engegada/aturada del serveis.
Per indicar la part "obligatoria" de la restricció s'utilitza un score **INFINITY**.

Afegim la restricció de que, si ClusterIP no està actiu enlloc, WebSite no tindrà permés estar actiu enlloc tampoc.
```
pcs constraint colocation add WebSite with ClusterIP INFINITY
pcs constraint
```


### Ordenar l'engegada i l'aturada dels recursos (ordering constraints)

Per assegurar-nos que WebSite respon independentment de la configuració de l'adreça de Apache, necessitem comprovar que ClusterIP s'engega abans de WebSite, a part de funcionar al mateix node. 

Quan Apache s'assigna a una IP, importa l'ordre. Si ja estava engegat Apache un cop s'afegeix l'adreça, no respondrà a aquesta.

```
pcs constraint order ClusterIP then Website
pcs constraint
```

### Preferencia de node per a ubicar recursos (location constraints)

Pot ser que preferim ubicar els recursos en una màquina en concret perquè es més potent que les altres, o perque té un hardware en concret. 
Per a això, podem utilitzar les **location constraints** o restriccions de localització. 

L'score quantifica la preferencia que tenim de que el servei funcioni en la localització indicada.
Si fiquéssim una score de preferència a i02 major que l'score de preferència a i26, sempre que estiguéssin els dos engegats, el recurs se situaria a i02.

Si només fiquem un score de preferencia a un node, sempre que hi sigui actiu, el recurs es situarà a aquest node.

```
pcs constraint location WebSite prefers i02=50
```

Si així no s'han col·locat els recursos al node que hem indicat la preferencia, pot ser perque el nostre **score de preferència** es menor que la *stickiness dels recursos*.



### Moure recursos manualment

Existeix una comanda per a situar recursos manualment, i també d'esborrar les restriccions prèvies.
```
pcs resource move WebSite i02
pcs resource clear WebSite
```

En cas de que necessitem forçar moure recursos a una localització específica. Això ho podem fer amb INFINITY, que supera qualsevol score de preferència.
```
pcs constraint location WebSite prefers i02=INFINITY
```

### Esborrar constraints

```
pcs constraint list --full
pcs constraint remove constraint_id
```
