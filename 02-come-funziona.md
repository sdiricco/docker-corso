# Come funziona

> Cgroups e Namespaces sono fondamentali per implementare un sistema basato su container: impariamo a capire cosa hanno a che fare con Docker.

Il modello di esecuzione basato su container si fonda su una serie di funzionalità sviluppate nel kernel **Linux** e successivamente adottate anche da altre piattaforme, espressamente studiate per questo scopo.

Un sistema basato su container inteso come alternativa alla virtualizzazione deve garantire prestazioni superiori pur offrendo le stesse caratteristiche in termini di **flessibilità** nella gestione delle risorse e di **sicurezza**.

L'incremento prestazionale è dovuto essenzialmente all'eliminazione di uno strato: a differenza di quanto avviene quando un processo viene eseguito all'interno di una macchina virtuale, i processi eseguiti da un container sono di fatto eseguiti dal sistema ospitante, usufruendo dei servizi offerti dal kernel che esso esegue. Viene quindi eliminato l'overhead dovuto all'esecuzione di ogni singolo kernel di ogni VM.

I requisiti di flessibilità e sicurezza in un sistema virtualizzato sono a carico dell'*Hypervisor*, lo strato software in esecuzione nella macchina ospitante che si occupa di gestire le risorse allocate a ciascuna VM e che adotta (anche con l'ausilio dell'hardware) tutte le politiche necessarie per isolare i processi in esecuzione su VM differenti.

In un ambiente basato su container, dove quindi non è presente un Hypervisor, queste funzionalità sono assolte dal kernel del sistema operativo ospitante. Linux dispone di due caratteristiche progettate proprio per questo scopo: **Control Groups (o cgroups)** e **Namespaces**.

## I control groups (cgroups)


I control groups sono lo strumento utilizzato dal kernel Linux per gestire l'utilizzo delle risorse di calcolo da parte di un gruppo specifico di processi. Grazie ai cgroups è possibile limitare la quantità di risorse utilizzate da uno o più processi. Ad esempio, è possibile limitare il quantitativo massimo di memoria RAM che un gruppo di processi può utilizzare.

In un sistema Linux esistono più cgroups, ciascuno associato ad uno o più **resource controllers** (talvolta chiamati anche subsystems), in base alle risorse che gestiscono. Ad esempio, un cgroup può essere associato al resource controller memory per gestire la quantità di memoria allocabile da un certo insieme di processi.

I cgroup sono organizzati in modo gerarchico in una struttura ad albero. Ogni nodo dell'albero rappresenta una gruppo, definito dalle **regole** che gestiscono la risorsa a cui è associato (ad esempio la dimensione massima di memoria allocabile) e dalla **lista dei processi** che ne fanno parte. Ciascun processo in quella lista risponderà alle regole definite nel gruppo.

Si può interagire manualmente con i cgroup attraverso il filesystem virtuale **/sys**. I cgroup attualmente in uso dal kernel sono accessibili come subdirectory di `/sys/fs/cgroup`. Per creare un nuovo cgroup è sufficiente creare una subdirectory in quel ramo del filesystem.

```sh
# mkdir /sys/fs/cgroup/memory/miocgroup
# echo 104857600 > /sys/fs/cgroup/memory/miocgroup/memory.limit_in_bytes
# echo 1234 > /sys/fs/cgroup/memory/miocgroup/cgroup.procs
```

Come si intuisce, questa caratteristica permette di gestire le risorse allocate ad un particolare container in esecuzione. Qualora sia necessario impostare un limite per una risorsa specifica, sarà sufficiente creare un cgroup configurandolo opportunamente.

Docker sfrutta questa caratteristica del kernel Linux per implementare i limiti delle risorse allocate ai container. Quando un container è configurato con un limite su una o più risorse, Docker crea i corrispondenti cgroups in */sys/fs/cgroup/* ed aggiunge automaticamente i PID dei processi in esecuzione nel container.

## Namespaces

Oltre a gestire l'allocazione delle risorse, il kernel ospitante ha anche il compito di garantire l'**isolamento dei processi** in esecuzione in container differenti. Per ovvi motivi di sicurezza non deve essere possibile per un processo in esecuzione in un container accedere direttamente alla macchina ospite o ad altri container.

Questa funzionalità è implementata nel kernel Linux mediante l'uso dei namespaces. Un namespace è essenzialmente un "contenitore" che astrae le risorse offerte dal kernel. Quando un processo fa parte di un certo namespace esso potrà accedere soltanto alle risorse presenti nel namespace.

In particolare, esistono diversi namespaces di default, ciascuno associato ad una tipologia differente di risorse: cgroup, IPC, Network, Mount, PID, User, UTS.

Per ciascun container in esecuzione Docker crea un opportuno gruppo di namespaces ed associa i processi in esecuzione nel container a quel namespace. In questo modo, i processi in esecuzione nel container non accederanno ai namspace della macchina ospitate, né a quelli di altri container. Ciò permette effettivamente di isolare i processi in esecuzione in un container.

Se non si creassero dei namespace ad hoc per ogni container, i processi del container potrebbero accedere direttamente alle risorse dell'host. Ad esempio, se un container non fosse associato al proprio namespace "cgroups", esso potrebbe accedere ai cgroups della macchina host, avendo così il potere di gestire le risorse ad esso assegnate a piacimento.

Analogamente, se un container non fosse ristretto al proprio namespace "Network", esso potrebbe accedere allo stack di rete della macchina host, avendo il potere di interferire arbitrariamente con le socket attive sull'host.

Come si intuisce dal nome, il namespace "PID" permette di isolare l'albero dei processi in esecuzione: un processo che voglia enumerare i processi in esecuzione potrà solo vedere quelli nel proprio namespace. Inoltre ogni namespace PID gestisce una propria numerazione, quindi i processi enumerati all'interno di un container avranno un PID diverso rispetto a quello effettivamente assegnatogli nel namespace globale.

Assegnare dei namespace differenti ad ogni container significa quindi creare delle sandbox in grado di isolare i processi del container dal resto della macchina.

## Conclusioni

Cgroups e Namespaces sono caratteristiche fondamentali per l'implementazione di un sistema basato su container. Esse sono considerate mature essendo presenti in Linux da più di un decennio e basandosi su concetti già concepiti durante lo sviluppo di OpenVZ (nell'ormai lontano 2005).

Funzionalità analoghe sono state successivamente **integrate anche in ambiente Windows**, come parte dei *Windows Native Containers*. Tuttavia, ricordiamo che non è possibile eseguire container Linux direttamente sul kernel Windows (dato che, per l'appunto, non si virtualizza il sistema operativo ospite). In questi casi (host Windows ed immagini Linux), Docker ricorre comunque ad una macchina virtuale Linux, in esecuzione su HyperV.