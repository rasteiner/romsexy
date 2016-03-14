+++
date = "2016-03-14T18:41:00+01:00"
title = "Docker per sviluppatori - Episodio 1"

+++

Questo è il primo di una serie di articoli che parlano delle ovvietà che ho e avrò scoperto nella tematica di come sfruttare al meglio docker in un workflow di sviluppo. In questo episodio: come risolvere un problema di permessi tra host e container. 

<!--more-->

Avete mai provato ad aver installato più di una versione di PHP sul vostro portartile? È una cosa possibile, più o meno quanto l'autofellatio.
Avere degli stack di Apache + PHP isolati in container è quindi una bella idea. Personalmente mi sono creato [questa immagine](https://hub.docker.com/r/rasteiner/common-php-apache/). Nulla di che, il docker file è il seguente:

```
FROM php:5-apache

RUN apt-get update && apt-get install -y \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libmcrypt-dev \
        libpng12-dev \
    && docker-php-ext-install iconv mcrypt \
    && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-install gd mcrypt exif mbstring

RUN a2enmod rewrite
```

Insomma, prendo l'immagine officiale di PHP su Apache e ci installo le estensioni che uso spesso. Questo va benissimo come base per i miei progetti, potrei quindi fare per ogni progetto un Dockerfile come questo:

```
FROM rasteiner/common-php-apache
COPY ./public /var/www/html 
```

Significa però che dovrei: 

1. Farlo, per ogni progetto (anche quelli scaricati a cacchio da internet per guardarli un po') 
2. eseguire `docker build . -t applicazioneMia && docker run -p 80:80 applicazioneMia` ogni volta che cambio qualcosa al codice sorgente. 

Insomma, un consumo insostenibile di polpastrelli e un po' di spreco di tempo. 

L'altra possibilità sarebbe di eseguire il container in modo generico, "mountando" la cartella dei sorgenti dell'host dentro al container, in questo modo apache accede direttamente ai miei file sull'host e quindi non c'è la necessità di fare un build per ogni modifica:

```console
$ docker run -v "$PWD:/var/www/html" -p 80:80 rasteiner/common-php-apache
```

`$PWD` ovviamente viene sostituito da bash con la path della cartella attuale. 

Tutto bello! Funziona benissimo! A meno che l'applicazione non prevede di scrivere nella cartella "shareata". Ovviamente `ẁww-data` (l'utente di Apache all'interno del container) non ha nessun diritto di scrittura sulla **mia** cartella. Potrei risolvere dando a chiunque permessi di scrittura (`chmod -R 777 .`) e sappiamo tutti quanto questo sia una *buona idea*, almeno finché non veniamo presi a sassate dal sysadmin di turno.

La soluzione al problema dei permessi sembra essere quello di dare a `ẁww-data` lo stesso UID del nostro utente. In questo modo, se ho capito bene, fondamentalmente il kernel non capisce mica che `www-data` è qualcun'altro (e quindi è un po' come se Apache venisse eseguito con gli stessi diritti sui file come ce li ho io). Su macchine Linux, l'utente principale (cioè l'utente creato durante l'installazione di Linux) spesso ha UID 1000. 
Se nel nostro Dockerfile dello stack PHP Apache quindi inseriamo l'istruzione di settare l'uid di `ẁww-data` a `1000`, questo problema si risolve. Io mi son fatto un'[immagine apposta](https://hub.docker.com/r/rasteiner/phpapache-for-devs/):


```
FROM rasteiner/common-php-apache

RUN usermod -u 1000 www-data
```

In `~/.bashrc` quindi ho aggiunto un alias del genere:

```console
alias apache='docker run -p 80:80 -v "$PWD:/var/www/html" rasteiner/phpapache-for-devs'
```

Così posso giocare con qualsiasi sito `solo PHP` in questo modo:

```console
$ cd ~/applicazioneMia
$ apache
```

