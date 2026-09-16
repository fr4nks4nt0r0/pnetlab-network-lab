# pnetlab-network-lab
Ambiente di laboratorio virtualizzato realizzato con PNetLab su VMware, progettato per simulare l'infrastruttura di rete di un'azienda distribuita su più sedi.  

OBIETTIVO DEL PROGETTO

Dopo aver realizzato un homelab focalizzato sulla sistemistica Windows, con active directory, GPO e pfSense, ho voluto spostare l'attenzione sul networking vero e proprio, per imparare a lavorare con VLAN, tagging 802.1Q, routing inter-VLAN e una struttura composta da due sedi separate.
In modo da capire cosa succede nel livello 2 del modello OSI, cosa viaggia taggato, cosa rimane untagged, dove viene modificato un frame e perché un pacchetto riesce o meno a raggiungere la destinazione.

STACK TECNOLOGICO
Hypervisor: VMware Workstation

Emulatore di rete: PNetLab

Router e switch: MikroTik CHR 7.23.5 con RouterOS

Endpoint: Alpine Linux 3.24

Reti utilizzate:

-  192.168.10.0/24, VLAN 10 Prod

-  192.168.20.0/24, VLAN 20 DMZ

-  192.168.99.0/24, VLAN 99 Mgmt

-  192.168.50.0/24, rete della filiale

-  10.0.12.0/30, link punto-punto

SETUP

La preparazione del laboratorio ha richiesto diversi passaggi, le immagini sono state caricate in /opt/unetlab/addons/qemu/ tramite SFTP. Il disco RAW di MikroTik è stato convertito in qcow2 usando qemu-img convert, mentre Alpine Linux è stato installato da ISO all'interno di un nodo temporaneo.
La topologia finale comprende 9 nodi su PNetLab: 2 router, 3 switch e 4 endpoint, e a questi si aggiunge il nodo Net, utilizzato per simulare l'uscita verso internet e come rete di transito tra le due sedi.
Sul livello 2, l'Access-Switch utilizza un bridge con vlan-filtering.
La porta ether1 funziona come trunk e trasporta tutte e tre le VLAN tramite tagging, mentre le porte ether2, ether3 ed ether4 funzionano invece come porte access con PVID rispettivamente 10, 20 e 99.
Sul Core-Switch sono state create tre sub-interfacce VLAN sulla porta trunk, e ognuna dispone del proprio indirizzo IP e funziona come gateway della relativa VLAN, permettendo il routing inter-VLAN.
La filiale utilizza invece una rete layer 2 piatta sul Branch-Switch, mentre il gateway si trova direttamente sull'interfaccia LAN di R-Branch.
Gli endpoint Alpine utilizzano indirizzi IP statici configurati in /etc/network/interfaces.

SEGMENTAZIONE LAYER 2 CON VLAN

La differenza principale rispetto al progetto precedente riguarda proprio il modo in cui viene realizzata la segmentazione, nel laboratorio precedente avevo separato le reti utilizzando diverse reti host-only, collegate alle varie interfacce del firewall. Mentre qui invece tutte e tre le VLAN condividono lo stesso collegamento fisico tra Access-Switch e Core-Switch.
A distinguerle è il tagging 802.1Q, sul trunk le VLAN viaggiano taggate.
Sulle porte access il traffico rimane untagged. Gli host alpine non devono conoscere l'esistenza delle VLAN visto inviano e ricevono normali frame ethernet, mentre è lo switch a gestire il tagging in base al PVID della porta.
Il Core-Switch riceve il trunk e associa ogni VLAN alla propria sub-interfaccia. L'indirizzo configurato su queste interfacce diventa il gateway del relativo segmento e permette di effettuare il routing tra VLAN diverse e la verifica pratica può essere fatta tramite la tabella MAC del bridge con /interface bridge host print.
La tabella mostra gli host appresi sulle porte corrette e associati al relativo VLAN ID.
Il traffico tra VLAN diverse passa sempre dal Core-Switch e non viene instradato direttamente dall'Access-Switch.

PROBLEMI E SOLUZIONI

----->>>>>nodi che si avviavano e crashavano subito

Alcuni nodi QEMU partivano correttamente e dopo pochi secondi passavano dallo stato verde a quello rosso, senza mostrare errori utili, lLa causa era la virtualizzazione "annidata".
VMware non riusciva ad abilitare VT-x e mostrava il messaggio not supported on this platform.
Il problema non dipendeva dalla mia CPU (i5-12450H) come ipotizzavo, ma da Windows, che stava utilizzando la virtualizzazione attraverso la Piattaforma macchina virtuale, necessaria anche per funzionalità come WSL2.
Ho risolto disattivando questa funzionalità dalle impostazioni di Windows, riavviando il sistema e abilitando successivamente VT-x/EPT nelle impostazioni della macchina virtuale.

----->>>>>nomi delle interfacce diversi dalle etichette della topologia

Le etichette visualizzate sui collegamenti di PNetLab non corrispondono necessariamente ai nomi delle interfacce presenti dentro RouterOS.
Su un nodo le porte comparivano come ether1 ed ether2, mentre su un altro nodo creato dallo stesso template comparivano come ether5, ether6, ether7 ed ether8 con la numerazione dipende dal numero di interfacce assegnate durante la creazione del nodo.
Questo mi ha fatto perdere parecchio tempo perché inizialmente configuravo gli indirizzi IP sulle porte sbagliate.
Da quel momento ho iniziaro ad eseguire sempre /interface print appena entro in un nuovo nodo e configurare soltanto le interfacce effettivamente collegate e con il flag R.

----->>>>>ERRORE INDIRIZZO IP

A un certo punto il link punto-punto tra R-Edge e Core-Switch non trasferiva traffico, ho cambiato porta e configurazione diverse volte cercando di capire quale fosse il problema ma era molto più semplice di quanto pensassi, su R-Edge avevo configurato 10.0.212.1/30 invece di 10.0.12.1/30.
Le due configurazioni appartenevano a subnet completamente diverse, quindi nessun ping poteva funzionare.
Una volta corretto l'indirizzo, la porta originale funzionava perfettamente.

----->>>>>CONFIGURAZIONE IP PERSA DOPO RIAVVIO
Sugli host Alpine avevo inizialmente utilizzato ip addr add per configurare gli indirizzi, però il comando modifica la configurazione corrente, non la rende persistente.
Dopo ogni riavvio gli indirizzi sparivano e il laboratorio sembrava improvvisamente rotto, la soluzione è stata spostare la configurazione in /etc/network/interfaces utilizzando iface eth0 inet static e applicarla con /etc/init.d/networking restart.
Sui dispositivi MikroTik questo problema non si presenta perché RouterOS mantiene automaticamente la configurazione.

----->>>>>NUOVI NODI CON DISCO VUOTO

Ogni nuovo nodo Alpine partiva nuovamente dall'installer invece di utilizzare il sistema già installato, il motivo era il funzionamento dei dischi overlay di PNetLab.
Ogni nodo utilizza un proprio disco overlay che fa riferimento al template condiviso in sola lettura e di conseguenza, installare Alpine all'interno di un nodo non modifica il template originale.
Per risolvere il problema ho appiattito il disco di un nodo già installato utilizzando qemu-img convert -O qcow2 e l'ho utilizzato per sostituire il virtioa.qcow2 del template.
Da quel momento i nuovi nodi Alpine sono partiti direttamente dal sistema già configurato.
Bisogna però prestare attenzione al backing file. Se il template viene spostato mentre esistono nodi che lo utilizzano, la catena dei backing file può rompersi e la conversione può fallire.

----->>>>>ERRORE NO MIRROR FOUND INSTALLANDO ALPINE

L'installer di Alpine non riusciva a scaricare i pacchetti perché il nodo era collegato a una VLAN interna senza accesso a internet così per completare l'installazione ho collegato temporaneamente il nodo al Net.
Terminata l'installazione, l'ho ricollegato alla VLAN prevista dalla topologia.

----->>>>>HOST RAGGIUNGIBILE DAL GATEWAY MA NON ALTRE VLAN

Alpine-DMZ rispondeva correttamente al ping proveniente dal Core-Switch, che rappresentava il suo gateway diretto, ma non rispondeva alle richieste provenienti da Alpine-Client, che si trovava su una VLAN diversa, così ho verificato progressivamente PVID, tabella VLAN del bridge, tabella MAC, ARP sul gateway e l'assenza di regole firewall che potessero bloccare il traffico.
A livello di switch sembrava tutto corretto.
Anche i contatori di ifconfig su Alpine-DMZ davano un'indicazione interessante: durante il ping aumentavano sia RX sia TX.
Il pacchetto quindi arrivava e l'host stava anche generando una risposta.
Il problema era nel file /etc/network/interfaces.
Avevo lasciato configurato gateway 192.168.10.1, che apparteneva a un altro segmento, invece di 192.168.20.1.
L'host riceveva correttamente la richiesta, ma quando doveva rispondere utilizzava un gateway appartenente a un'altra subnet. Non trovando una route valida per la risposta, scartava il pacchetto.
Dall'esterno sembrava semplicemente che l'host non rispondesse.
In realtà il layer 2 funzionava perfettamente e il problema si trovava nella configurazione di routing dell'host.

VERIFICA FINALE

La configurazione dell'Access-Switch può essere verificata con /interface bridge vlan print detail, che conferma la presenza delle tre VLAN, il trunk taggato e le porte access untagged.
Con /interface bridge host print è possibile verificare che ogni endpoint venga appreso sulla porta e sul VLAN ID corretti.
Sul Core-Switch, /ip address print conferma la presenza dei tre gateway SVI e del link punto-punto verso R-Edge.
I ping da Alpine-Client, appartenente alla VLAN 10, verso Alpine-DMZ nella VLAN 20 e Alpine-Mgmt nella VLAN 99 hanno restituito risposta regolare con 0% di packet loss, confermando il corretto funzionamento dell'inter-VLAN routing.
Alpine-Branch raggiunge correttamente il proprio gateway.
Il ping da Alpine-Client verso Alpine-Branch restituisce invece 100% di packet loss.
In questo caso il risultato è previsto: le due sedi non sono ancora collegate e manca il tunnel site-to-site.
Dopo un riavvio completo della topologia, tutti gli host mantengono la propria configurazione IP.

Gli screenshot del progetto sono disponibili nella cartella /screenshots.

COMPETENZE DIMOSTRATE

* progettazione di un piano di indirizzamento coerente con subnet dimensionate in base al ruolo, utilizzando /30 per i link punto-punto e /24 per i segmenti utente
* configurazione di VLAN con tagging 802.1Q, trunk, porte access, PVID e VLAN filtering su bridge
* configurazione dell'inter-VLAN routing tramite sub-interfacce SVI
* troubleshooting strutturato a più livelli, partendo da layer 2 e passando attraverso ARP, routing e configurazione degli host
* utilizzo di tabelle MAC, contatori delle interfacce e packet sniffer durante il troubleshooting
* gestione delle immagini QEMU direttamente a livello di filesystem
* conversione dei formati disco e comprensione del funzionamento di overlay e backing file
* valutazione di diverse soluzioni software anche in base ai relativi vincoli di licenza
* utilizzo di strumenti legalmente ottenibili senza rinunciare alle funzionalità di networking necessarie al laboratorio

SVILUPPI FUTURI

Gli sviluppi previsti per il progetto sono:

* configurazione di OSPF tra Core-Switch e R-Edge, con propagazione della default route verso gli host interni
* configurazione di un firewall stateful zone-based su R-Edge, con isolamento della DMZ dalla LAN
