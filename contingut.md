# Procediment pràctic

## Objectiu
Muntaré un clúster d'alta disponibilitat de dos nodes amb Fedora, que ofereixi el servei HTTP de forma continuada. En cas de caiguda d'un dels dos nodes, l'altre node seguirà oferint aquest servei. 
El contingut de el document html que oferirà el nostre servei, estarà sincronitzat en ambdós nodes, és a dir, si ho modifiquem a un dispositiu, el contingut es replicarà a l'altre.

En aquest cas, he utilitzat com a nodes dos ordinadors de l'aula, i02 i i26. Generalment s'aconsella treballar desde un sol dispositiu, fent ús de la connexió ssh per a configurar els demés nodes.

Primer de tot, ens tenim que assegurar que els ordinadors on treballarem permetin l'*alta disponibilitat* per treballar amb el clúster.
Podem obrir manualment els ports TCP 2224, 3121, 21064, i el port UDP 5405, a la vegada que deshabilitem el Selinux.
L'altra opció es habilitar-ho desde el servei del comandes de firewall:

```
firewall-cmd --permanent --add-service=high-availability
firewall-cmd --reload
```

## Procediment per crear un clúster

Es necessari que totes les omandes per a la configuració del clúster, dels nodes, instal·lacions, i, fins i tot, consultes de l'estat dels recursos de l'estructura, es realitzin com a usuari *root* o administrador.
Instal·lem el programari necessari a tots els nodes: 

```
dnf -y install pacemaker pcs psmisc corosync drbd gfs2-utils
```

Engeguem el servei *pcsd* per a poder generar i configurar el clúster, i també habilitem que s'engegui quan fem boot de l'ordinador, per a agilitzar el procés.

```
systemctl start pcsd
systemctl enable pcsd
```

Seguidament, tindrem que ficar una contrasenya a l'usuari *hacluster*, ja que serà necessari per a sincronitzar la configucació de corosync, o permetre el control dels nodes desde els altres nodes del clúster.
Es convenient utilitzar la mateixa contrasenya a tots els usuaris hacluster dels diferents nodes.

```
passwd hacluster
```

### Configuració del Corosync

Ens autentiquem a tots els nodes amb l'usuari esmentat per a poder començar el procés, i per a que posteriorment puguem crear el cluster desde un sol node.
```
pcs host auth i02 i26

pcs cluster setup mycluster i02 i26 (si no funciona, assegurar-se que la contrasenya de hacluster es la mateixa a tots els nodes)
```

Podem engegar tots els nodes del clúster a la vegada, o de forma separada:

```
pcs cluster start --all (engegar tots els nodes del clúster)
pcs cluster start i02 i26 (múltiples nodes desde qualsevol node)
pcs cluster start (desde el node que vols engegar)
```

we are not enabling the corosync and pacemaker services to start at boot. 
If a cluster node fails or is rebooted, you will need to run pcs cluster start nodename (or --all) 
to start the cluster on it. While you could enable the services to start at boot, requiring a manual 
start of cluster services gives you the opportunity to do a post-mortem investigation of a node failure before returning it to the cluster.

Comprobar con corosync que los nodos están conectados: 
```
corosync-cfgtool -s
```

Para chequear las "membresías" del clúster
```
corosync-cmapctl | grep members
pcs status corosync
```
El ID es la IP, y en faults no debería salir nada (si no hay fallos). If  you  see  something  different,  you  might  want  to  start  by  checking  the  node’s  network,  firewall  andSELinux configurations

Fencing 

STONITH ("Shoot The Other Node In The Head" or "Shoot The Offending Node In The Head"), sometimes called STOMITH ("Shoot The Other Member/Machine In The Head"), is a technique for fencing in computer clusters.[1]
Fencing is the isolation of a failed node so that it does not cause disruption to a computer cluster. As its name suggests, STONITH fences failed nodes by resetting or powering down the failed node.

Using IPMI as a power fencing device may seem like a good choice. However, if the IPMI shares powerand/or network access with the host (such as most onboard IPMI controllers), a power or network failurewill cause both the host and its fencing device to fail. The cluster will be unable to recover, and must stopall resources to avoid a possible split-brain situation.

'''
pcs property set stonith-enabled=true
'''

Por lógica, hay que tener esta opción activada, ya que no se entiende el funcionamiento de un clúster sin ella. Tener STONITH desactivado provoca que los nodos que no funcionen correctamente, se interpreten como nodos apagados de forma segura.

### Añadir IP al clúster

Tenim que afegir una ip mitjançant la qual s'identificará qualsevol node de cara l'exterior del clúster, independentment del servei que s'estigui duent a terme.

Generem una floating IP, amb el nom de ClusterIP, que el clúster comprovi cada 30 segons que el dispositiu identificat estigui engegat.
```
pcs resource create ClusterIP ocf:heartbeat:IPAddr2 ip=x.x.x.x cidr_netmask=xx op monitor interval=30s
```


### Testejar failover

### Quorum
(2 * active nodes) > total nodes
Excluyendo los clústers de 2 nodos, si esta condición no se cumple, pacemaker aturará tots els nodes per a prevenir la corrupció de dades. Corosync permet que això es doni també en els clústers de dos nodes. 


### Prevent Resources from Moving after Recovery
El transport de recursos entre nodes, requereix un temps determinat, que augmenta quan es tracten bases de dades, per exemple. En general, es preferible que aquests recursos siguin engegats on hi eren prèviament. 
El concepte *stickiness* de Pacemaker, controla en quina mida preferim que el servei sigui engegat on ja hi és. Per defecte, Pacemaker assumeix que no hi ha cap tipus de cost en moure els recursos entre nodes, per això la existencia de aquest concepte.
```
pcs resource defaults resource-stickiness=100
pcs resource defaults
```
