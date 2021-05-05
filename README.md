# wpvm 
Deploy automatico di una architettura ridondata e scalabile per wordpress basata su vm.

# Architettura
La ridondanza e la scalabilità del db (db1..N nello schema) viene garantita tramite cluster galera (builtin nella versione di mariadb installata):
* replica sincrona parallela a livello di riga;
* multi-master active/active;
* controllo, esclusione e unione dei nodi/membri automatica;
* look and feel nativo di MySQL (i client possono contattare uno qualsiasi dei nodi che compongono il cluster senza strati ulteriori);
* sono richieste almeno 3 istanze per la gestione di quorum e split brain e per la sopravvivenza al fail di 1 nodo.
Il db viene esposto mediante una coppia di bilanciatori active/passive (keepalived/vrrp) chiamati nello schema dbb1/2 e implementati con linux ip virtual server in modalità direct routing:
* Il bilanciamento viene fatto in kernel space a livello trasporto e sfrutta l'algoritmo wlc (Weighted Least-Connections: nuove connessioni assegnate al server con meno connessioni attive in proporzione al peso assegnatogli);
* La modalità direct routing è quella più prestante (il vincolo e' che bilanciatori e backend si parlino in L2, il backend viene contattato tramite bilanciatore ma risponde direttamente al chiamante);
* Fra i 2 nodi è attiva la sincronizzazione delle tabelle di persistenza: in caso di failover viene mantenuta l'associazione fra client e server di backend.
I nodi wp1..N sono i web server che erogano il frontend wordpress. La sincronizzazione del file system (necessaria per temi e upload) avviene mediante volume gluster montato in /var/lib/wordpress/wp-content:
* il volume gluster è configurato per replicare i dati su ogni nodo così da garantire l'alta affidabilità;
* per una maggiore scalabilita' (in termini di spazio occupato e banda utilizzata) e' possibile adottare altre tipologie di volume (vedere Dispersed o Distributed Replicated Volume);
* sono richieste almano 3 istanze per la gestione di quorum e split brain e per la sopravvivenza al fail di 1 nodo;
* a livello di design avrebbe senso disaccoppiare in server differenti i brick gluster dai webserver.
I client non contattano direttamente i webservice ma una coppia di reverse proxy nginx in bilanciamento ip_hash:
* il bilanciamento ip_hash e' meno prestante delle altre tipologie nginx ma è l'unico che consente ad un client di atterrare sempre sullo stesso server di fronted;
* i reverse proxy sono in ha (active/passive con keepalived/vrrp).

<pre>
        +-------galera--------+
        |                     |
+-------+-+   +---------+   +-+-------+
|         |   |         |   |         |   // mariadb
|   db1   +---+   db2   +---+   dbN   |   // galera
|         |   |         |   |         |   // cluster 
+---------+   +---------+   +---------+
     ^             ^             ^
     |             |             |
    vip           vip           vip
     |             |             |
     +---+  +------+             |
         |  |  +-----------------+
         |  |  |
       +-+--+--+-+   +---------+
       |         |   |         |   // vrrp
       |   dbb1  +---+   dbb2  |   // linux ipvs dr wlc
       |         |vip|         |   // persistence sync
       +---------+   +---------+
         ^  ^  ^
         |  |  |
         v  v  v
         i  i  i
         p  p  p
         |  |  |
     +---+  |  +-----------------+
     |      +------+             |
     |             |             |
+----+----+   +----+----+   +----+----+
|         |   |         |   |         |   // gluster
|   wp1   +---+   wp2   +---+   wpN   |   // volume
|         |   |         |   |         |   // replica N
+---------+   +---------+   +---------+
     ^             ^             ^
     |             |             |
     |             |             |
     |             |             |
     |      +------+             |
     +----+ |  +-----------------+
          | |  |
       +--+-+--+-+   +---------+
       |         |   |         |   // vrrp
       |   wpb1  +---+   wpb2  |   // nginx load balancer ip_hash 
       |         |vip|         |   // persistence sync
       +---------+   +---------+
</pre>

# Bootstrap
## Tool e parametrizzazione
* L'intera infrastruttura viene costruita con vagrant e provisioning ansible; 
* E' possibile modificare alcune impostazioni in vmm/provisioning/vm-config.yml;
* La documentazione delle impostazioni e' direttamente nel file di configurazione.

### Prerequisiti e setup dell'ambiente
ambiente: debian buster + vagrant con provider libvirt 
* sudo apt-get install git vagrant sudo qemu libvirt-daemon-system libvirt-clients ebtables dnsmasq-base libxslt-dev libxml2-dev libvirt-dev zlib1g-dev ruby-dev
* sudo adduser *user* libvirt
* sudo adduser *user* libvirt-qemu
* sudo echo "*user* ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers 
* sudo echo "deb http://ppa.launchpad.net/ansible/ansible/ubuntu trusty main" >> /etc/apt/sources.list
* sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 93C4A3FD7BB9C367
* sudo apt-get update
* sudo apt-get install ansible python-jmespath
* sudo gem install ipaddress
* ansible-galaxy collection install gluster.gluster
* ansible-galaxy collection install community.mysql

## Startup
* Verificare le conf in vmm/provisioning/vm-config.yml (attenzione ai parametri legati al networking)
* Per deployare l'infrastruttura eseguire:
<pre>
cd vmm
vagrant up
</pre>

## Accesso
* Con un browser puntando a http://192.168.122.100

# Note:
* Il playbook ansible per il provisioning non e' completamente idempotente.
