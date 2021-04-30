On each node, allow cluster-related services through the local firewall:

# firewall-cmd --permanent --add-service=high-availability
success
# firewall-cmd --reload
success

If you are using iptables directly, or some other firewall solution besides firewalld, simply openthe following ports, which can be used by various clustering components: TCP ports 2224, 3121,and 21064, and UDP port 5405.

**Procediment per crear un clúster**

The pcs command will configure corosync to use UDP unicast transport
Instal·lar en els nodes: pacemaker pcs psmisc corosync drbd gfs2-utils

ssh als nodes
Set up hacluster passwd in all nodes (same password)

1. Un cop reaitzada la configuració, fem 
```
pcs host auth i02 i26
pcs cluster setup mycluster i02 i26 (si no funciona, asegurarse que la contraseña de hacluster es la misma en todos los nodos)
pcs cluster start --all "or" pcs cluster start i02 i26 "or" pcs cluster start en cada nodo
```
we are not enabling the corosync and pacemaker services to start at boot. 
If a cluster node fails or is rebooted, you will need to run pcs cluster start nodename (or--all) 
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
