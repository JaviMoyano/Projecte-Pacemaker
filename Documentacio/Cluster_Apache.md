1. Instal·lar apache en tots els nodes
```
No tenim que engegar httpd. Els serveis que s'han de administrar mitjançant el software del cluster, mai han de ser administrats pel sistema operatiu.
```
Crear el mateix html en els dos nodes.

Amb l'objectiu de monitoritzar el fncionament de la instancia d'Apache, i recuperar-la si falla, el agent de recursos utilitzar per Pacemaker considera que la URL del server en està disponible. En els dos nodes, tenim que habilitar la URL amb:
```
*/etc/htpd/conf.d/status.conf*
<Location /server-status>
        SetHandler server-status
        Order deny,allow
        Deny from all
        Allow from 127.0.0.1
</Location>
```

###Configurar el clúster

Afegim la configuració que hem fet a Apache al clúster mitjançant el recurs *Website*. Utilitzarem un recurs script OCF de recursos anomenat apache del heartbeat. El únic paràmetre requerit per l'script es el fitxer de configuració de Apache. Cada minut checkejará si el Apache segueix engegat.
```
pcs resource create WebSite ocf:heartbeat:apache configfile=/etc/httpd/conf/httpd.conf statusurl="http://localhost/server-status" op monitor interval=1min
```

Fiquem un timeout de 2 minuts, que es el adequat per el nostre servei
```
pcs resource op defaults update timeout=120s
pcs resource op defaults
```

En cas de que no funcioni el Website_start (*pcs status*), mirar de no tenir engegat el httpd en cap dels nodes.
Una alra opció pot ser que no haguem habilitat la URL del status correctament. Potser podem trobar el problema a: 
```
wget -O - http://127.0.0.1/server-status
```
En cas de que ens doni **Connection refused**, assegurar-nos que al status.conf hem ficat la opció **Allow from 127.0.0.1**

### Funcionament dels recursos al mateix node. Colocation constraints.

Pacemaker tracta de distribuir automàticament els recursos entre tots els nodes del clúster. Tot i així, es pot indicar que alguns recursos han de funcionar conjuntament al mateix host, o al contrari, que no tenen que estar engegats al mateix host.

Per aconseguir aixó, es poden ficar restriccions de ubicació o *colocation constraints*. Aquestes restriccions son "direccionals", és a dir, depenent de l'ordre en que les escrivim, els recursos tindràn una col·locació o un altra, però no té res a veure amb l'ordre d'engegada/aturada del serveis.

Per indicar la part "obligatoria" del *constraint* utilitzant un score *INFINITY*.

Si ClusterIP no està actiu enlloc, WebSite no tindrà permés estar actiu enlloc tampoc.
```
pcs constraint colocation add WebSite with ClusterIP INFINITY
```
### Ordenar l'engegada i l'aturada dels recursos. Ordering constraints.

Per assegurar-nos que WebSite respon independentment de la configuració de l'adreça de Apache, necessitem comprovar que ClusterIP s'engega abans de WebSite, a part de funcionar al mateix node. 

Si Apache *binds to the Wildcard*, no importa que s'afegeixi una IP abans o després d'engegar Apache; funcionará igual. Però si Apache només *binds to certain IPs*, importa l'ordre: si ja estava engegat Apache un cop he afegit l'adreça, no respondrà a aquesta.

```
pcs constraint order ClusterIP then Website
```

### Preferencia hosts per a ubicar recursos
Pot ser que preferim ubicar els recursos en una màquina en concret perquè es més potent que les altres, o té un Hardware en concret. 
Per a això, podem utilitzar les *location constraints* o restriccions de localització. 
L'score indica quantifica la preferencia que tenim de que el servei funcioni en la localització indicada.

```
pcs constraint location WebSite prefers i02=50
```

Si així no s'han col·locat els recursos al node que hem indicat la preferencia, pot ser perque el nostre *score de preferència* es menor que la *stickiness dels recursos* (quantificat el temps innecesari que no voliem que estigués aturat un node)

```
crm_simulate -sL
```
### Esborrar constraints

```
pcs constraint list --full
pcs constraint remove constraint_id
```

## Moure recursos manualment

En cas de que necessitem forçar moure recursos a una localització específica. Això ho podem fer amb INFINITY, que supera qualsevol score de preferència.

Si eliminem la nova restricció feta manualment per a que el clúster continui funcionant de forma normal, si prèviament hem ficat una stickiness de 1, el clúster es quedarà com l'hem deixat després del canvi.
COMPROBAR

```
pcs constraint location WebSite prefers i02=INFINITY
```
