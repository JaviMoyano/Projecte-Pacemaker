# Procediment pràctic

## Sincronització de dades amb DRBD

Quan oferim un servei com apache a més dun node, i les dades que es mostren son cambiants, implica una réplica de les dades en els dos o més dispositius d'emmagatzemament. 
Si falla un, les dades de l'altre node han de estar correctament sincronitzades i poder ser utilitzades.

DRBD otorga aquesta característica als nodes del clúster. Això pot ser comparat amb un RAID de nivell 1.

[Imatge RAID1](imatges/raid1.png)

Per a poder aplicar aquest software, DRDB necessitarà el seu propi block device a cada node.

***
Per replicar l'emmagatzemament, hem de afegir les configuracions necessaries a l'arxiu de */etc/drbd.d/global_common.conf* que conté les seccions globals i comuns de la configuració DRBD.

global {
 usage-count  yes;
}
common {
 net {
  protocol C;
 }
}

DRBD admet tres modes de replicació:
- Protocol A: protocol de replicació asíncrona.
- Protocol B: protocol de replicació semi-asíncrona. Sincronització de memoria.
- Protocol C: utilitzat normalment en nodes de xarxes petites. Es l'ús més comú.

Definim el nostre recurs a */etc/drbd.d/test.res*

DRDB necessitarà el seu propi block device a cada node. Per a aquest exercici utilitzarem un volum logic de 1GiB, que es més que sificient per a un document HTML,  posteriorment, GFS2 metadata.
***

### Crear Volumen Lógico

En los dos nodos
```
cd /opt/lvm
dd if=/dev/zero of=disk01.img bs=1k count=1000K
dd if=/dev/zero of=disk02.img bs=1k count=1000K

losetup /dev/loop1 /opt/lvm/disk01.img
losetup /dev/loop2 /opt/lvm/disk02.img

pvcreate /dev/loop1
pvcreate /dev/loop2

vgcreate diskedt /dev/loop1 /devloop2 

lvcreate --name drbd.demo --size 1G diskedt
```

Sample DRBD configuration

Editar según los datos del nodo: la IP, el nombre del host y los volúmenes lógicos.
La opció allow-two-primaries, no s'utilitza en nodes passive-active, però la afegim per a posteriorment canviar-ho a un clúster active/active.

*/etc/drbd.d/wwwdata.res*
```
resource wwwdata {
	protocol C;
	meta-disk internal;
	device /dev/drbd1;
	syncer {
		verify-alg sha1;
		}
net {
	allow-two-primaries;
}
on i02 {
	disk /dev/diskedt/drbd.demo;
	address 10.200.243.202:7789;
}
on i26 {
	disk /dev/diskedt/drbd.demo;
	address 10.200.243.226:7789;
}
}
```

Creamos los metadatos de forma local para el recurso DRBD, comprobamos que está cargado el módulo DRDBD, y actvamos el recurso DRBD. Ejecutamos el proceso en un sólo nodo.
```
drbdadm create-md wwwdata
modprobe drbd
drbdadm up wwwdata
```

Comprobamos el estado de DRBD en este nodo.
```
cat /proc/drbd
```
Està marcat com a *Inconsistent* perquè encara no hem inicialitzat les dades.
Està marcat com a *Waiting for Connection* perquè encara no hem inicialitzat el segon node, i aquest suposat node, està marcat com a *Unknown*.


Si repetim el procés al segon node, quan fem les comprobacions, surten com a *Connected, Secondary, i Inconsistent data* al segon node.

Per a que les dades siguin consistents, necessitem indicar-li al DRBD quin node es el que té les dades correctes. En aquest cas escollim i02.

```
drbdadm primary --force wwwdata
```

Quan el node secundari estigui UpToDate podem procedir a crear i *populate* un sistema de fitxers per al nostres documents del recurs WebSite, al node primari.

```
mkfs.ext4 /dev/drbd1
mount /dev/drbd1 /mnt
vim /mnt/index.html
umount /dev/drbd1
```

Després, guardem tota la configuració del *Clúster information Base* (està en format XML) en un arxiu (drbd_cfg).
```
pcs cluster cib drbd_cfg
```

[Instalar en todos los nodos *dnf -y install drbd-pacemaker* para tener linbit y los recursos de clonación]

Amb la opció *pcs -f* podem fer canvis directament a l'arxiu de configuració, però no es veuran reflexats al clúster fins que fem un push a la versió live del CiB.

Creem un recurs per a DRBD, i una copia de aquest per permetre al recurs ser utilitzar en els dos nodes a la vegada.
```
pcs -f drbd_cfg resource create WebData ocf:linbit:drbd drbd_resource=wwwdata op monitor interval=60s
pcs -f drbd_cfg resource promotable WebData \         promoted-max=1 promoted-node-max=1 clone-max=2 clone-node-max=1 \         notify=true
```

Al acabar fem push per a instaurar-ho en el cluster actiu.
```
pcs cluster cib-push drbd-cfg
```

[Insertar foto *pcs status*]

###Configurar el clúster para el sistema de archivos

Després de engegar el device DRBD, tenim que muntar el sistema de arxius. Li indicarem que el filesystem només s'allotgi en el Primary DRBD, i que s'engegi després de que s'ahi clonat el Primary.

```
pcs -f fs_cfg resource create WebFS ocf:heartbeat:Filesystem device="/dev/drbd1" directory="/var/www/html" fstype="ext4"
pcs -f fs_cfg constraint colocation add WebFS with WebData-clone INFINITY with-rsc-role=Master
pcs -f fs_cfg constraint order promote WebData-clone then start WebFS
```

També li direm al cluster que s'ha de engegar a la mateixa màquina que el Filesystem i que ha de ser actiu abans de iniciar Apache.

```
pcs -f fs_cfg constraint colocation add WebSite with WebFS INFINITY
pcs -f fs_cfg constraint order WebFS then WebSite
pcs cluster cib-push fs_cfg
pcs status
```

Per últim, per simular un failover del clúster, podriem aturar el node i02, o també ficar el node en *standby*. Els nodes en aquest estat, continuen amb el corosync i el pacemaker engegat, pero no permeten utiltizar els recursos. Pot servir per simular actualitzacions de paquets utilitzats per recursos.

```
pcs node standby i02
```

Podem observar que tots els recursos es mouen al node i26


### Configurar STONITH
STONITH protegeix les dades de ser corrompudes per nodes que no funcionin correctament. Un node pot no respondre, pero no significa que les dades estiguin protegides; STONITH s'assegura que el node realment estigui offline abans de permetre l'accés a les dades desde altres nodes.

Tenim que diferenciar entre errors del node, i errors de la xarxa. 


