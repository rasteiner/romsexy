+++
date = "2016-03-03T19:41:56+01:00"
title = "L'ostica conquista del sito vecchio"

+++

Nel nostro mondo moderno di git, taskrunner, infrastrutture "programmate", applicazioni che vengono distribuite insieme ad un proprio intero filesystem linux (leggi: [docker](http://docker.com)) e altro, è difficile ricordarsi come su pianeti meno fortunati le cose non funzionano così. 

Ognitanto, però, ti viene ricordato tuo malgrado. 

Raccontiamo qui l'eroica storia di come Omicron Perseo 8 si è impadronito di un vecchio pianeta, senza accesso FTP, fino ad arrivare ad una soluzione di deployment completamente automatizzata.

<!--more-->

Questi giorni mi è capitato di dover mettere gli artigli su un sito ereditato, modificandone un template. Uno di quei <strike>siti</strike> **cosi** di cui non vuoi sentirti responsabile. Tuttavia, giusto per rendere le cose più divertenti, questa volta l'unico accesso dato era quello di un CMS proprietario, niente FTP, figuratevi un accesso alla shell. 
Il CMS *ovviamente* non dava la possibilità di modificare i template tramite interfaccia web. Quindi? Questo pianeta non lo conquistiamo? 

Non lasciamoci scoraggiare. Il vantaggio dei CMS fatti in casa, spesso, è quello che ti aprono delle possibilità mai viste prima. E così è stato anche questa volta. 

Fortunatamente il CMS offriva la possibilità di caricare delle immagini. Fortunatamente queste immagini potevano essere anche dei file PHP, o qualsiasi cosa in realtà.

Esistono degli script PHP assai interessanti, uno di questi si chiama "[PHP Shell](http://phpshell.sourceforge.net/)". Il grande riassunto dice che si tratta di uno script che fa da ponte tra un'interfaccia web e `shell_exec` di PHP. Insomma, è una riga di comando sul web. E qui il mondo si apre. Tutto d'un tratto si ha un accesso shell al server, con le credenziali di apache. (La premessa ovviamente è che "Safe mode" di PHP sia disabilitato e che quindi `shell_exec` funzioni).
Quindi mi ritrovo con un URL tipo "http://www.ilovebees.com/cms/immagini/phpshell.php" che allegramente esegui i miei comandi (dietro password, s'intende).

Qui diventa interessante. Cosa possiamo fare? (Oltre ai danni, s'intende). PHP Shell offre anche la possibilità di caricare dei file (almeno ovunque Apache ne abbia i permessi). Quindi, teoreticamente, potrei usarlo per caricare i file del template modificati, uno alla volta. Però questo sarebbe un processo lento e fatidico, non adatto ad un Omicroniano. 

Mettiamo dunque tutto il sito in un archivio e scarichiamolo tramite web. Così possiamo modificarlo in locale, ricaricare tramite PHP Shell un archivio con file modificati e quindi spachettarlo sul server per avere un "deployment" più atomico. Mi fanno male le zampe, a pensarci. 

Vediamo quindi cosa c'è installato su questo server, non si sa mai: magari c'è git. 

```console
$ git --version
sh: 1: git: not found
```

Ecco no. Ovviamente Apache non ha i diritti per installare niente. Vediamo quindi che macchina è? 

```console
$ uname -m
x86_64
```

GUARDA PAPÀ! È la stessa architettura del mio portatile! (È una tipica frase che si sente spesso dire ai bambini, no?)

*Un paio di ricerche google dopo, in locale:*

1. Scarichiamo il codice sorgente di git (https://github.com/git/git/releases)
2. Spacchettiamolo in una cartella
3. Compiliamolo con linking statico (significa che le librerie, che normalmente sono separate dal programma, vengono incluse nell'eseguibile. In questo modo questi saranno più grossi, ma sono anche indipendenti e quindi portatili).
   
   ```console
   $ make configure
   $ ./configure --prefix=/home/cdrrr/git-static/ CFLAGS="${CFLAGS} -static-libgcc"
   $ make
   $ make install 
   ```

Se tutto va bene, questo ci crea in `/home/cdrrr/git-static/` un paio di cartelle con quasi l'intera suite di git (non comprende le manpages, ma anche chissene).

Pacchettiamo tutta questa cartella e carichiamola sul server. 

Quindi, tornati in PHP Shell:

```console
$ ./static-git/bin/git --version
git version 2.7.2
```
Yeee! 
Ma ancora meglio:

```console
$ PATH="$PATH:$PWD/static-git/bin" && git --version
git version 2.7.2
```
La seconda variante garantisce che git trova se stesso, se lo desidera. Per esempio `git pull`, dietro le quinte, esegue un altro file che dev'essere trovato e quindi trovarsi nella path.

Creiamo quindi un piccolo script wrapper che comprende il seguito:

```bash
#!/bin/bash

# The MIT License (MIT)
# Copyright (c) 2013 Alvin Abad
# https://alvinabad.wordpress.com/2013/03/23/how-to-specify-an-ssh-key-file-with-the-git-command

if [ $# -eq 0 ]; then
    echo "Git wrapper script that can specify an ssh-key file
Usage:
    git.sh -i ssh-key-file git-command
    "
    exit 1
fi

# remove temporary file on exit
trap 'rm -f /tmp/.git_ssh.$$' 0

if [ "$1" = "-i" ]; then
    SSH_KEY=$2; shift; shift
    echo "ssh -o StrictHostKeyChecking=no -i $SSH_KEY \$@" > /tmp/.git_ssh.$$
    chmod +x /tmp/.git_ssh.$$
    export GIT_SSH=/tmp/.git_ssh.$$
fi

# in case the git command is repeated
[ "$1" = "git" ] && shift

# Run the git command
PATH="$PATH:$PWD/static-git/bin" && git "$@"

```
Come potete leggere dal commento, è un wrapper che consente di specificare a git una chiave con la quale accedere ai nostri repository privati su github o bitbucket, o quello che è. Serve, fondamentalmente, perché `www-data` (cioè l'utente di Apache) in questo caso non aveva una "home" e non aveva nemmeno il permesso di crearla. Quindi bisogna mettere la chiave in un altro posto e bisogna anche dire a git (o meglio: SSH) dove trovarla. 
Come creare coppie di chiavi probabilmente lo sapete, altrimenti lo trovate ovunque su internet.

Io ho solo aggiunto l'ultima riga, quella che c'imposta dove trovare il nostro git (siccome non è veramente "installato"). 

Dato che ora abbiamo git dove prima git non l'avevamo, possiamo trasformare il sito in repositorio.  
Come al solito:

```console
$ git init
$ git add .
$ git commit -m "initial"
```
e caricarlo dove volete:

```console
$ git remote add ssh://github.com/user/repo.git
$ ./git.sh -i ./lamiachiave push
```
Questo è un buon momento per togliere "lamiachiave.pub" dalle impostazioni del vostro utente github e metterla come "deployment key" del progetto.

A questo punto possiamo scaricare da github il nostro progetto in locale, fare le modifiche, pusharle e quindi in PHP Shell scaricarle con 
```console
$ ./git.sh -i ./lamiachiave pull
```

Ecco abbiamo finito. 

---

tanto lo so cosa pensate... "Dobbiamo fare `./git.sh -i lamichiave pull` ogni volta?!? CHE FATICA!"  
E avete ragione. 

Quindi ci manca un ultimo passaggio. 

Basta caricare un ultimo file nella webroot che ha piuomeno questo aspetto:

```php
<?php
chdir('/pathDelRepo');
echo `./git.sh -i ./lamiachiave pull`;
```
e quindi impostare tale script PHP come webhook "postcommit" in github. In questo modo, quando pushate a github, il vostro stupido vecchio sito, ora intelligente, si scarica da solo gli aggiornamenti.
