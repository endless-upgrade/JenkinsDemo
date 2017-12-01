
[Install_centos]: img/InstallazioneCentos6.7.png "Installazione di centos con minimal Destktop GUI"
[create_user]: img/createUser.png "Creazione utente"
[fire1]: img/firewall1.jpg "Firewall TUI 1"
[fire2]: img/firewall2.jpg "Firewall TUI 2"
[fire3]: img/firewall3.jpg "Firewall TUI 3"
[fire4]: img/firewall4.jpg "Firewall TUI 4"
[fire5]: img/firewall5.jpg "Firewall TUI 5"
[unlock_jenkins]: img/unlock_jenkins.png "unlock"



# Deploy di Jenkins 2.91 per Mondadori

-----------
**Attenzione** Se ti trovi dietro a un proxy e installi Jenkins su una VM in localhost, nessun servizio Git* sarà in grado di connettersi ad esso.

Se non ti trovi dietro un proxy ti basta sostituire "localhost" con il tuo **IP pubblico** nei vari webhook.

-------- 

## Installazione VM

* Download di Centos v6.7 da http://archive.kernel.org/centos-vault/6.7/isos/x86_64/

---

## Creazione macchina virtuale su GCP o simile:
* Centos v6.7
* 8 GB RAM
* 20 GB Hdd

---

## Installazione di Centos 6.7
* Selezionare lingua inglese
* Selezionare lingua italiana per la tastiera
* Impostare l'orario su "Europe,Rome"
* Root Password : password

![alt text][Install_centos]

* Create a user : **user** con password : **password**

![alt text][create_user]

* Reboot

* Imposta il proxy (se sei in sede reply)

```bash
su
Password: *******

vi /etc/yum.conf

#aggiungi in fondo
proxy=http://proxy.reply.it:8080

vi /etc/profile

#aggiungi in fondo
export http_proxy=http://proxy.reply.it:8080
export https_proxy=http://proxy.reply.it:8080

source /etc/profile
```

* Imposta la connessione automatica al boot

```bash
vi /etc/sysconfig/network-scritps/ifcfg-eth0

#modifica
ONBOOT=yes
```

* Aggiorna il sistema

```bash
sudo yum update -y
```

* Se vuoi, installa le Virtual Box guest additions (io lo trovo molto comodo, michele le odia)

```bash
sudo yum update
sudo yum install gcc
sudo yum install kernel-headers
sudo yum install kernel-devel
```

---

## Installazione di jenkins

Per l'installazione di jenkins seguirò un misto tra questa [guida per Centos 7](https://www.digitalocean.com/community/tutorials/how-to-set-up-jenkins-for-continuous-development-integration-on-centos-7)
 e questa [guida specifica per Centos 6](https://wiki.jenkins.io/display/JENKINS/Installing+Jenkins+on+Red+Hat+distributions).

### Ottenere un utente NON root con privilegi di root

Il nostro utente *user* va più che bene a questo scopo, per garantigli privilegi di root è necessario editare il file `/etc/sudoers` ma, mi raccomando, **NON FARLO MAI CON VIM**.

Usa invece:
```bash
su
Password: ******

visudo
```
Giusto per capire cosa andremo a fare: `root ALL=(ALL:ALL) ALL`
* *root* indica l'utente
* Il primo *ALL* indica che la regola si applica ad ogni host
* Il secondo *ALL* indica che l'utente root esegue comandi per ogni altro utente (tramite sudo)
* Il terzo *ALL* indica che può eseguire comandi per ogni gruppo
* L'ultimo *ALL* indica che la regola si applica a qualsiasi comando

Decommenta la riga

```bash
%wheel ALL=(ALL) ALL
```

In modo da garantire tutti i diritti agli utenti del gruppo wheel.
Quindi aggiungi *user* al gruppo wheel

```bash
sudo usermod -aG wheel username

reboot #per rendere valida la modifica
```

Puoi trovare maggiori informazioni su come gestire i diritti sudo a [questa pagine](https://www.digitalocean.com/community/tutorials/how-to-edit-the-sudoers-file-on-ubuntu-and-centos).

### Installazione di Jenkins da pacchetto RPM

Tutti i comandi dovranno essere fatti con l'utente **NON ROOT** tramite sudo.

#### Installare java 1.8

Per prima cosa installiamo java, questa versione di jenkins richiede java 1.8

Connettiti alla pagina di [oracle](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html), accetta la licenza e scarica **jdk-8u151-linux-x64.tar.gz**.

``` bash
cd /opt
wget http://download.oracle.com/otn-pub/java/jdk/8u151-b12/e758a0de34e24606bca991d704f6dcbf/jdk-8u151-linux-x64.rpm?AuthParam=1511780178_e971f91c277e5926c5a9b60179d896de

sudo rpm -ivh jdk-8u151-linux-x64.rpm\?AuthParam\=1511780178_e971f91c277e5926c5a9b60179d896de

java -version
java version "1.8.0_151"
Java(TM) SE Runtime Environment (build 1.8.0_151-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.151-b12, mixed mode)

# per salvare spazio
sudo rm -rf jdk-8u151-linux-x64.rpm\?AuthParam\=1511780178_e971f91c277e5926c5a9b60179d896de
```

#### Download del repo RPM

Scarica il pacchetto RPM di jenkins, puoi spostarti in `/etc/yum.repos.d` oppure eseguire direttamente

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo http://pkg.jenkins-ci.org/redhat/jenkins.repo
```

Importa il codice di verifica di validità del pacchetto

```bash
sudo rpm --import https://jenkins-ci.org/redhat/jenkins-ci.org.key
```

Se, per una qualche ragione, rpm non riesce a scaricare la chiave restituendo un errore 404 (come è successo a me):

```bash
sudo wget https://jenkins-ci.org/redhat/jenkins-ci.org.key
sudo rpm --import jenkins-ci.org.key
```

#### Installazione con yum

Una volta fatto possiamo installare jenkins con yum.

```bash
sudo yum install jenkins
```

Terminata l'installazione, avviamo il servizio di jenkins e impostiamo l'avvio automatico

```bash
sudo service jenkins start
sudo chkconfig jenkins on
```

#### File utili

E' stato creato un utente *jenkins* pensato per eseguire tutto quello che è legato al servizio, puoi modificare questo utente nel file di config , ma ricorda di cambiare i privilegi a `/var/log/jenkins, /var/lib/jenkins e /var/cache/jenkins.

Troverai i file di log in 

Cosa    | File
------- | -------
Config  | `/etc/sysconfig/jenkins`
Log     | `/var/log/jenkins/jenkins.log`

#### Connessione al servizio e firewall

Usa `service jenkins start/stop/status` per gestire il servizio.

All'avvio del servizio, potrai interagire con jenkins tramite il tuo browser alla porta **8080**, il firewall potrebbe darti dei disagi, per disabilitarlo, in Centos 6.7 non possiamo usare `firewalld`, utilizzeremo la TUI di `system-config-firewall`

```bash
sudo iptables -L #per vedere le attuali regole
```

Per modificare permanentemente il firewall, in modo da aprire la porta **tcp:8080**

```bash
sudo system-config-firewall-tui
```

![alt text][fire1]
![alt text][fire2]
![alt text][fire3]
Ovviamente inserisci la porta 8080
![alt text][fire4]

OK -> Back -> Close -> Ok

![alt text][fire5]

---

## Avvio e utilizzo di jenkins

```bash
sudo service jenkins start
Starting Jenkins                                           [  OK  ]

```
E collegati alla pagina `http://35.196.238.203:8080`, dovrebbe aprirsi una pagina del genere.

![alt text][unlock_jenkins]

Scopriamo la password autogenerata

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
ab21de47d4cf4d70856f57d953fb84bf
```

### Installazione plugin

Quindi installiamo i plugin che ci servono (io devo fare una prova, ho installato quelli di default).

![alt text](img/troll.png)
**Attenzione** se sei dietro a un proxy questa procedura fallirà malamente, questo perchè jenkins ha bisogno di una configurazione proxy ma nessuno ci ha dato la possibilità di configurarlo.

Non preoccuparti per ora, fai continua e ci pensiamo dopo.

### Creazione utente admin

Per comodità ho creato un utente admin default:
* Username = admin
* Password = admin

Se tutto va bene dovrebbe aprirsi l'homepage di jenkins.

![alt text](img/jenkins_home.png)

---

## Configurare Jenkins

### Proxy

Manage Jenkins -> Manage Plugins -> "Advanced" tab.
Qui puoi impostare il proxy reply.

![alt text](img/proxy.png)


### Enable Global Security

Manage Jenkins -> Configure Global Security

Se la spunta non è presente, spuntare *Enable Security*

//TODO Spuntare Matrix-based security e configura un utente admin (non c'è il checkbox) (https://wiki.jenkins.io/display/JENKINS/Matrix-based+security)

### Installare Plugin

Manage Jenkins -> Manage Plugins -> "Available" tab

Seleziona i plugin che desideri, per la mia Demo ho selezionato solo **GIT plugin** e **GitHub Plugin** dal momento che il progetto da gestire si trova su github.
In ogni caso jenkins ha installato anche tutti i plugin che erano falliti nell'installazione iniziale.

Esegui l'installazione con restart e, al termine, torna alla Top Page.

### Configurare GIT

Se Git non è installato nell'host questo non funzionerà nemmeno su jenkins (grazie al cazzo).

```bash
sudo yum install git -y

# Per sapere dove è stato installato
sudo which git
/usr/bin/git
```
Inoltre dovremo configuralo in Jenkins, andiamo quindi in Manage Jenkins -> Configure System -> sezione Git Plugin
e inseriamo nome utente e email dell'account usato nel repository.

![alt text](img/git_plugin_config.png)

Non possiamo però ancora accedere al repository, la configurazione sarà completata all'inserimento del link (prossia sezione).

## Demo Git Hub

Nella homepage creiamo un *New Job* dal pulsante centrale se non ne abbiamo o da *New Item* a sinistra.

![alt text](img/new_job.png)

Metti il nome che preferisci e seleziona *Freestyle Project*

![alt text](img/job_name.png)

Imposta la descrizione, spunta *GitHub project* e inserisci il link del repository (intendo l'**URL nella barra di navigazione**)

![alt text](img/demo_name.png)

Qui invece imposta il **link per clonare il repository** e le credenziali dell'account GitHub

![alt text](img/set_credentials.png)

Imposta poi il build del job Jenkins in modo che venga eseguito ogni volta che viene fatto un commit git.

Inoltre puoi impostare una serie di comandi che vengono fatti durante la build, strumento potentissimo, **da imparare ad usare**.

![alt text](img/hook.png)

### Configurazione Webhook GitHub

Un webhook è un collegamento che permette di scatenare un determinato evento tra due differenti servizi web. In questo caso, vogliamo fare in modo che, nel momento in cui viene fatto il commit di su GitHub, venga fatta una POST `http://35.196.238.203:8080/hithub-webhook`. Questa chiamata scatenerà build, test, deploy e compagnia bella.

La configurazioni deve essere fatta lato GitHub, al momento è implementata come un servizio, non come un webhook, andiamo quindi nella pagina **Integration & Services**

![alt text](img/github_webhook.png)

Imposta il link del server su cui è presente jenkins.

![alt text](img/hook_jenkins_url.png)


## Demo GitLab

Seguirò questa [guida](https://medium.com/@teeks99/continuous-integration-with-jenkins-and-gitlab-fa770c62e88a).

GitLab ha davvero un sacco di problemi con Jenkins, da quel che dice il web funziona solo con la versione enterprise di GitLab, oppure con il tool di CI proprietario di GitLab.

Al momento utilizzo:
* GitLab 9.5.4
* GitLab Plugin [1.5.2](https://github.com/jenkinsci/gitlab-plugin)


Per questa combinazione di versioni esiste una [Know Issue](https://gitlab.com/gitlab-org/gitlab-ce/issues/37288) e [issue2](https://github.com/jenkinsci/gitlab-plugin/issues/608) per cui il bottone "Webhook" test di Git Lab lancia una POST errata, vedremo al momento del test come gestirlo.


### Chiave SSH per il deploy

Per prima cosa è necessario configurare una chiare RSA per il deploy, questa procedura è comoda perchè permette al server jenkins di accedere ai repository di nostro interesse, senza la necessità di creare utenti "dummy" con diritti custom.

#### Generare la chiave

```bash
su
Password: ****

cd /var/lib/jenkins/.ssh
sudo -u jenkins ssh-keygen

Generating public/private rsa key pair.
Enter file in which to save the key (/var/lib/jenkins/.ssh/id_rsa): id_rsa
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in id_rsa.
Your public key has been saved in id_rsa.pub.
The key fingerprint is:
SHA256:pELwRD6I+aWPm3XZ2R2mbZxh018dLNL4BBSm9/LPbzY jenkins@devops-worker
The keys randomart image is:
+---[RSA 2048]----+
|  ..o     .=.    |
| o *      o + .  |
|o . *   .. + + o |
| . + . o  . =.. o|
|  o . . S  .*o. o|
|   o . o o Bo= ..|
|  . o o o o *.  .|
|   + .     .  oEo|
|  o            =+|
+----[SHA256]-----+
```

Una volta generata la chiave, copiala e inseriscila nel repository GitLab

```bash
cat id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC9WxJsoJXl1XpMQk2bAaRSfKzYjBq5NeNOkhzvCP2cnyKm0NK409IlWpDpTec5qOSaOsJ6e0IGftKHXVRXLnH39jOWZVGEbZ096pzZIyGZAMdq2xM+Azm6CpcS6Vl/XGyer8gtsvKDJ5kD5q8EUFALKV6y642V7dh6UIcacyQuL6tF6xumaCLW+20ASUmkucqmDQ9aSi08c8HsxiIIAQmZs6XCS8ecCRoO8LyhLCLSguwwaMFtWpomINTOApH/6k9nQDWWM3Kl/1+uUoomAyuj7G1zj7gHAp4oxXO5/rIZ4l0Ynvyw9ods8/NWmy6349/5nXnITuDOafrQLJnaj0Dv jenkins@devops-worker
```
#### Inserire la chiave su GitLab

Apri il repository GitLab a cui vuoi applicare la Chiave di Deploy

![alt text](img/gitlab.png)

Settings -> Repository -> Deploy Keys

![alt text](img/deploy_keys.png)

Dai un nome alla chiave, copiala e spunta *Write access allowed*

![alt text](img/add_key.png)

#### Inserire la chiave in jenkins

La stssa chiave deve essere inserita in Jenkins, per farlo vai in Credentials -> System -> freccina nel dominio -> Add Credentials

![alt text](img/jenkins_add_rsa.png)

![alt text](img/jenkins_config_rsa.png)

### Creare un Access Token GitLab

Accedere alle configurazioni del profilo in GitLab -> Access Tokens

![alt text](img/gitlab_profile_settings.png)

Dai un nome al tuo token e seleziona una expiration date

![alt text](img/gitlab_pat.png)

[//]: <> (d5188a5c982e87a9a9207e13bcb94677)

Viene quindi generato e mostrato il tuo Personal Access Token super segretissimo, mi raccomando **SALVALO** perchè non te lo faranno vedere mai più.

Metti quindi il token in jenkins creando una nuova Credentials come prima.

![alt text](img/jenkins_pat.png)

### Configurare la connessione tra Jenkins e GitLab

Torniamo in Manage Jenkins -> Configure System.

Sotto **Git Lab** sarà tutto rosso-incazzato, inserisci le informazioni richieste e sarà contento.

Fai **attenzione**, il *Gitlab host URL non è quello del tuo progetto, ma la radice, cioè il server su cui gira Gitlab.

Premendo *Test Connection* potrai ottenere un dolcissimo e gratificante *Success*.

![alt text](img/jenkins_gitlab_connection.png)

### Creazione del Job Jenkins

Esattamente come per Github, notare che la *GitLab Connection* viene inserita da sola
![alt text](img/so_good.jpg)
![alt text](img/gitlab_job.png)

Nella sezione *Gestione Codice Sorgente* inserisci

Config  | Value
------- | -------
URL     | url del repository Git Lab
Credentials | Le credenzioni RSA configurate con l'utente jenkins
Name    | origin
Refspec | +refs/heads/*:refs/remotes/origin/* +refs/merge-requests/*/head:refs/remotes/origin/merge-requests/*
Branch  | origin/${gitlabSourceBranch}

![alt text](img/jenkins_scm.png)

Impostiamo in modo che la compilazione avvenga ogni volta che avviene un qualche evento sul repository. Ovviamente configurare sulla base delle proprie esigenze.

**Attenzione** Segnati l'URL del servizio, nel mio caso http://35.196.238.203:8080/project/GitLab-demo.

![alt text](img/jenkins_job_trigger.png)

Aggiugi i passi di compilazione utili: **QUI VERRANNO INSERITI I TEST DIREI**

![alt text](img/jenkins_job_compilation_step.png)

Aggiungi un'azione di post compilazione, possiamo per esempio inviare una email a destinatari custom, oppure, ome in questo caso, mostrare sulla GUI di GitLab lo stato del commit. 

![alt text](img/jenkins_job_post.png)

### Creazione webhook lato GitLab

Lato GitLab è necessario configurare il Webhook in modo che venga fatta una POST in risposta a desterminati eventi.
Ovviamente configura il webhook in modo consistente con la configurazione degli eventi in jenkins.

![alt text](img/gitlab_webhook.png)

**********************************************************

## Test

Ok, dovrebbe essere tutto a posto, ora fai qualche test.

<!--
Commando    |   Stato CSRF  |   Risultato
------------|---------------|------------- 
http://35.196.238.203:8080/job/gitlab/                                                  |on|   403 No valid crumb 
http://jenkins:538e8c802c3f50f3dea529eaf68dcb69@35.196.238.203:8080/job/gitlab/build    |on|   403 No valid crumb 
http://jenkins:538e8c802c3f50f3dea529eaf68dcb69@35.196.238.203:8080/job/gitlab/build    |off|   WORKING
http://jenkins:538e8c802c3f50f3dea529eaf68dcb69@35.196.238.203:8080/project/gitlab      |on|   404 Not Found
http://35.196.238.203:8080/project/gitlab/                                              |on|   404 Not Found
curl -X POST  -H "Jenkins-Crumb:34549b0579c5d49eb3ee3a93a17e5d0a" http://localhost:8080/job/gitlab/build    |on|    WORKING
curl -X POST  -H "Jenkins-Crumb:34549b0579c5d49eb3ee3a93a17e5d0a" http://35.196.238.203:8080/job/gitlab/build    |on|    403
**http://35.196.238.203:8080/job/gitlab/build with `Jenkins-Crumb:dbb48add58a1862a3e0ea949143eb754` header**    |**on**|    **WORKING!**

[More Info](https://support.cloudbees.com/hc/en-us/articles/219257077-CSRF-Protection-Explained)

#### Per ottenere Crumb Anonimo

curl 'http://localhost:8080/crumbIssuer/api/xml?xpath=concat(//crumbRequestField,":",//crumb)'

#### Per ottenre Crumb con utente

curl -u "utente:password" 'http://localhost:8080/crumbIssuer/api/xml?xpath=concat(//crumbRequestField,":",//crumb)'

**Il crumb dipende sia dal nome utente che dall'IP** per questo la chiamata a `localhost` funziona mentre quella a `35.196.238.203` non funziona!!

# Il grosso problema è che non è possibile inserire il Crumb nella chiamata GitLab :(
    Quindi o disabiliti il CSRF o FANCULO!!!!
    Opto per installare GitLab 7.4.3 e testare
-------------->

Come già detto, esiste una know issure per cui il bottone "Webhook Test" di Git Lab (nella pagina di configurazione del webhook) invia una chiamata POST con un header errato.
Per testare che tutto funzioni possiamo quindi effettuare un normale commit oppure usare un tool come [Postman](https://www.getpostman.com/) per simulare la chimata REST.


### Postman

#### URL
```
http://35.196.238.203:8080/project/GitLab-demo
```
#### Headers
```
Content-Type:application/json
X-Gitlab-Event:Push Hook
X-Gitlab-Token:d5188a5c982e87a9a9207e13bcb94677
```
#### Body
```json
{
  "object_kind": "push",
  "event_name": "push",
  "before": "e35bfb275c5953be8acf19284618061749684de8",
  "after": "062380a3f6421bc376568f19b0445385fbdeaf2c",
  "ref": "refs/heads/master",
  "checkout_sha": "062380a3f6421bc376568f19b0445385fbdeaf2c",
  "message": null,
  "user_id": 119,
  "user_name": "Dario Pasquali",
  "user_username": "d.pasquali",
  "user_email": "d.pasquali@reply.it",
  "user_avatar": "http://www.gravatar.com/avatar/3fd07fe896722b6563f7ef62824d9bec?s=80&d=identicon",
  "project_id": 147,
  "project": {
    "name": "jenkins-CI-test",
    "description": "PoC of Jenkins CI with GitLab",
    "web_url": "http://jarvis.datareply.eu/d.pasquali/jenkins-CI-test",
    "avatar_url": null,
    "git_ssh_url": "git@jarvis.datareply.eu:d.pasquali/jenkins-CI-test.git",
    "git_http_url": "http://jarvis.datareply.eu/d.pasquali/jenkins-CI-test.git",
    "namespace": "d.pasquali",
    "visibility_level": 20,
    "path_with_namespace": "d.pasquali/jenkins-CI-test",
    "default_branch": "master",
    "ci_config_path": null,
    "homepage": "http://jarvis.datareply.eu/d.pasquali/jenkins-CI-test",
    "url": "git@jarvis.datareply.eu:d.pasquali/jenkins-CI-test.git",
    "ssh_url": "git@jarvis.datareply.eu:d.pasquali/jenkins-CI-test.git",
    "http_url": "http://jarvis.datareply.eu/d.pasquali/jenkins-CI-test.git"
  },
  "commits": [
    {
      "id": "062380a3f6421bc376568f19b0445385fbdeaf2c",
      "message": "test\n",
      "timestamp": "2017-11-27T19:18:17+01:00",
      "url": "http://jarvis.datareply.eu/d.pasquali/jenkins-CI-test/commit/062380a3f6421bc376568f19b0445385fbdeaf2c",
      "author": {
        "name": "d.pasquali",
        "email": "d.pasquali@reply.it"
      },
      "added": [

      ],
      "modified": [
        "prova.sh"
      ],
      "removed": [

      ]
    },
    {
      "id": "e35bfb275c5953be8acf19284618061749684de8",
      "message": "initial commit\n",
      "timestamp": "2017-11-27T19:15:12+01:00",
      "url": "http://jarvis.datareply.eu/d.pasquali/jenkins-CI-test/commit/e35bfb275c5953be8acf19284618061749684de8",
      "author": {
        "name": "d.pasquali",
        "email": "d.pasquali@reply.it"
      },
      "added": [
        "prova.sh"
      ],
      "modified": [

      ],
      "removed": [

      ]
    }
  ],
  "total_commits_count": 2,
  "repository": {
    "name": "jenkins-CI-test",
    "url": "git@jarvis.datareply.eu:d.pasquali/jenkins-CI-test.git",
    "description": "PoC of Jenkins CI with GitLab",
    "homepage": "http://jarvis.datareply.eu/d.pasquali/jenkins-CI-test",
    "git_http_url": "http://jarvis.datareply.eu/d.pasquali/jenkins-CI-test.git",
    "git_ssh_url": "git@jarvis.datareply.eu:d.pasquali/jenkins-CI-test.git",
    "visibility_level": 20
  }
}
```
Non avremo alcun feedback su Postman ma, aprendo la dashboard jenkins, vedrmo che la build è in corso...
![alt-text](jenkins_build.png)

...e poi viene aggiornato il conteggio dall'ultima compilazione
![alt-text](jenkins_build_done.png)

Cliccando sul job possiamo vedere il report di build, in questo caso essendo una push di test, non avremo nessun cambiamento.
![alt-text](jenkins_build_done2.png)

### Commit & Push

Ripetiamo il test facendo un commit & push.
![alt-text](bash_push.png)

Non appena viene fatto il push parte il build sulla dashboard jenkins.
![alt-text](jenkins_build_2.png)

Al termine del quale potremo ammirare il nuovo cambiamento avvenuto con successo.
![alt-text](jenkins_build_done3.png)

E il report console, dove possiamo notare i nostri passi di compilazione.
![alt-text](jenkins_build_done_console.png)

### Fallimento

Si noti che, in caso di fallimento, sia jenkins che GitLab mostreranno il l'evento. Per testare questa casistica ho inserito una `istruzione_inesistente` tra i passi di build, tutto il job fallisce se anche uno solo dei passi fallisce (anche se jenkins mostra la percentuale di completamento).

![alt-text](jenkins_build_fail.png)

Su GitLab possiamo osservare la differenza tra un push terminato con successo o fallimento.

![alt-text](jenkins_build_fail_gitlab.png)