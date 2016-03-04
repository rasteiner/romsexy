+++
date = "2016-03-04T12:52:33+01:00"
title = "La mia prima esperienza con hugo e wercker"

+++

Buone notizie, ciurmaglia! 

Questo sito si è aggiorna da solo, malgrado si tratti di un sito statico gentilmente costruito dal ["static site generator" di nome hugo](https://gohugo.io).
Ci ho messo un po' e quindi vorrei condividere come l'ho fatto. 

<!--more-->

Per chi non lo conoscesse, hugo è un utility che aiuta a creare siti "statici" partendo da una serie di file markdown e un template. Insomma, un po' come avere un CMS che però non gira sul server, ma in locale, e poi genera l'intero sito in HTML. 
A quanto pare, nella sua categoria, hugo è il più veloce. [Qualcuno sostiene](https://youtu.be/29LLRKIL_TI?t=261) che dove altri generatori sprecano diverse ore e poi crashano, hugo termina e completa il suo lavoro in 10 secondi. Significa che tutta una categoria di siti che prima erano "per forza" stampati da CMS (pensiamo a blog con migliaia di post), ora sarebbe possibile crearseli in "statico", con tutti gli vantaggi di sicurezza e scalabilità compresi. 

In questo post però non vorrei parlare di hugo in sé (probabilmente lo farò in un'altro). Qui vorrei parlare di come sfruttare un servizio gratuito come [wercker](http://wercker.com) per generare e *deployare* un sito fatto con hugo su un nostro server a cui abbiamo un accesso ssh ma non possiamo installare nulla. 

Wercker è un servizio online di "Continuous Integration", un po' come "travis ci", ma a mio avviso più semplice. 

Per prima cosa dobbiamo mettere il codice sorgente del nostro sito da qualche parte dove per wercker è raggiungibile, il servizio si integra fantasticamente con github ovviamente. Un'idea ve la potete fare qui: https://github.com/rasteiner/romsexy (è il sorgente di questo sito).

Come potete vedere in .gitignore c'è aggiunto "public/" che è la cartella in cui hugo genera il sito statico. In altre parole non troviamo il sito pronto nella nostra repository, prima di poterlo mettere online qualcuno lo deve generare. Questo è il compito di wercker. 

Nella repository abbiamo un file di nome `werker.yml`, con questo contenuto:

```yaml
box: debian
build:
  steps:
    - arjen/hugo-build:
        version: "0.15"
        prod_branches: master
deploy:
  steps:
    - install-packages:
        packages: openssh-client
    - add-to-known_hosts:
        hostname: rom.sexy
    - mktemp:
        envvar: PRIVATEKEY_PATH
    - create-file:
        name: write key
        filename: $PRIVATEKEY_PATH
        content: $KEY_PRIVATE
        overwrite: true
        hide-from-log: true
    - script:
        name: transfer application
        code: |
          pwd
          ls -la
          scp -r -i $PRIVATEKEY_PATH -o StrictHostKeyChecking=no -o UserKnownHostsFile=no public/* $DEPLOY_PATH
```

wercker funziona tramite docker, infatti `box: debian` sta ad indicare che partiamo con un'immagine di debian. Quello che c'è sotto "build" sono istruzioni di come generare il nostro sito, quello che sta sotto "deploy" sono i passi ("steps") necessari per fare il deploy. 

In questo caso il piano è di usare uno step definito da qualcuno, di nome `arjen/hugo-build`, per generare il nostro sito. E poi caricarlo sul nostro server tramite scp e una chiave privata.

Per quanto riguarda `build`: `version` si riferisce alla versione di hugo e `prod_branches: master` definisce di usare il branch `master` per la produzione. 

Gli step per il deploy invece sono questi:

1. Installa un client ssh (così possiamo usare `scp`)
2. Aggiungi il nostro host remoto al file `~/.ssh/known_hosts`. Così SSH non reclama quando ci connettiamo
3. Creiamo il nome di un file temporaneo e salviamolo nella variabile di ambiente `PRIVATEKEY_PATH`
4. Creaiamo un file con il nome sopradetto e salviamoci dentro la nostra chiave privata prendendola dalla variabile `$KEY_PRIVATE`. Questa la potete creare nelle impostazioni della vostra app su wercker - vedere le impostazioni "SSH Keys" e "Environment variables".
5. Infine eseguiamo il comando `scp` per caricare i file. Ho messo anche un `pwd` e `ps -la` per verificare (a occhio) che ci troviamo nel posto giusto. `DEPLOY_PATH` è un'altra variabile definita nelle impostazioni di wercker che contiene semplicemente il path della cartella pubblica sul server. 

L'App su wercker è definita in modo da eseguire il deploy automaticamente quando qualcuno pusha su `master`. Tra build e deploy passano circa 2 minuti e mezzo, la maggior parte dovuto all' "installare cose" e non all'esecuzione di hugo (che invece sono un paio di ms). Quindi anche se il sito dovesse crescere, non dovrebbe cambiare molto. 

Rimane soltanto di scambiare `scp` con `rsync`, in modo da non dover caricare tutto il sito ogni volta, ma la pausa pranzo e finita e questo verrà fatto presumibilmente un'altra volta.
