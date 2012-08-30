---
layout: post
title: FTP-only backup
---

A közelmúltban szükségem volt egy olyan backup megoldásra, ami akkor is működik, ha a tárhelyre csak
ftp elérésem van - a phpmyadmin is csak a szolgáltató saját adminfelületén keresztül érhető el.

A háttér az, hogy jóatyám céges joomlás oldalát felnyomták. Mivel nem saját kód, nem volt verziózva.
A szolgáltató backupolt, de ez nem letölthető, hogy összehasonlítsd korábbi változattal, hanem vissza
lehet állni rá.

Páróra játék eredménye [ez a két script](https://gist.github.com/3530787).

A megoldás egy php és egy bash scriptből áll. A php maga mellé egy file-ba menti a teljes DB dumpot,
a felhasznált függvény [David Walsh blogjáról](http://davidwalsh.name/backup-mysql-database-php) való.
Ez a mappa .htaccess-el védve van, csak jelszó ismeretében érhető el (tudom, hogy nem az igazi, de nem
találtam jobb megoldást). Cirka egymegás dumpot csinál egy másodpercen belül.

A lényegi munkát a shellscript végzi. A környezet: a mappa, amiben dolgozik, egyaránt egy checkoutolt
git repo, valamint a teljes ftp tartalmat letöltő wget parancs kimenete. A lépések:

1. a .git-en kívül minden törlése
1. adatbázis backup triggerelése
1. teljes ftp tartalom mentése
1. ha az előbbi meghiúsult, értesítő levél és kilépés
1. ha nem változott semmi file, kilépés
1. minden változás commitolása, pusholása
1. értesítő levél

A központi szerver egy saját hostolású [GitLab](http://gitlabhq.com/), így weben keresztül is
vissza lehet nézni, mikor mi változott. Gittel természetesen az adott időpontra visszaállás is könnyen
megoldható.
