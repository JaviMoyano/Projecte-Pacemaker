###DRBD
Reflexa es continguts dels dispositius com discs durs, particions, volums lógic, etc. entre servidors. Implica una copa de dades en dos o més dispositius de emmagatzemament, de manera que si falla un, les dades de l'altre puguin ser utilitzades. 

Comentari --> Semblant a Raid 1

DRDB necessitarà el seu propi block device a cada node. Per a aquest exercici

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
	disk /dev/diskedt/drbd-demo;
	address 10.200.243.226:7789;
}
}
```

Creamos los metadatos de forma local para el recurso DRBD, comprobamos que está cargado el módulo DRDBD, y actvamos el recurso DRBD. Ejecutamos el proceso en un sólo nodo.
```
drbdadm create-md wwwdate 
modprobe drbd
drbdadm up wwwdate
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

Després tenim que configurar el cluster per al device DRBD
```
pcs cluster cib drbd_cfg
```
