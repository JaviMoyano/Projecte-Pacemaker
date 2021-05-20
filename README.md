# Projecte Pacemaker edt 2020/2021

![Imatge Logo Pacemaker](cluster/Logo_Pacemaker.png)


## Objectius del projecte
L'objectiu es conéixer les funcionalitats de Pacemaker i d'altres tipus de software amb els que funciona conjuntament, per a dotar d'*Alta disponibilitat* a un clúster. Per demostrar el seu funcionament, aquest clúster estarà format per 3 nodes, i funcionarà com a un servidor que ofereix servei d'HTTP.

## Que es Pacemaker ?
Pacemaker es un tipus de software de control de recursos de clúster, que pot proporcionar diferents característiques al conjunt de hosts. 
S'utilitza generalment amb [Corosync] per . 

- Replicació de dades en els diferents nodes
- Protecció i replicació de dades en cas de la caiguda de algún node  (redundància)


## Clúster
Un **clúster** es un conjunt de dos o més nodes que funcionen amb l'objectiu de portar a terme una o varies tasques. Aquestes funcions poden ser: 
- **Emmagatzemament:** 
- **Alta disponibilitat:** proporciona un servei sense aturades en cas de que algún element del sistema es caigués o fallés. 
Un cop detectat un error de hardware o de software, s'engega automàticament la mateixa aplicació o servei a un altre sistema/node sense necessitat de fer-ho manualment. Aquest procés es conegut com *failover*, que es tradueix com a *migració a causa d'error*
- **Balanç de càrregues:** 
- **Alt rendiment:** 

### Corosync
Proporciona al nodes la capacitat de formar part del clúster, així com de comunicar-se entre ells. També pot habilitar al clúster per fer quorum i atorgar una eina més per a una millor protecció de les dades.