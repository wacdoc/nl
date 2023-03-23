# Bouw uw eigen SMTP-server voor het verzenden van e-mail

## preambule

SMTP kan rechtstreeks diensten afnemen van cloudleveranciers, zoals:

* [Amazon SES-SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Ali cloud e-mail push](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

U kunt ook uw eigen mailserver bouwen - onbeperkt verzenden, lage totale kosten.

Hieronder laten we stap voor stap zien hoe je een eigen mailserver kunt bouwen.

## Server selectie

De zelfgehoste SMTP-server vereist een openbaar IP-adres met poorten 25, 456 en 587 open.

Veelgebruikte publieke clouds hebben deze poorten standaard geblokkeerd en het is misschien mogelijk om ze te openen door een werkopdracht uit te geven, maar het is uiteindelijk erg lastig.

Ik raad aan om te kopen bij een host die deze poorten open heeft staan ​​en het opzetten van omgekeerde domeinnamen ondersteunt.

Hier raad ik [Contabo](https://contabo.com) aan.

Contabo is een hostingprovider gevestigd in München, Duitsland, opgericht in 2003 met zeer concurrerende prijzen.

Als u Euro als aankoopvaluta kiest, is de prijs goedkoper (een server met 8 GB geheugen en 4 CPU's kost ongeveer 529 yuan per jaar, en de initiële installatiekosten zijn een jaar gratis).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

Geef bij het plaatsen van een bestelling `prefer AMD` , en de server met AMD CPU zal betere prestaties leveren.

In het volgende zal ik de VPS van Contabo als voorbeeld nemen om te demonstreren hoe u uw eigen mailserver kunt bouwen.

## Ubuntu-systeemconfiguratie

Het besturingssysteem hier is Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Als de server op ssh `Welcome to TinyCore 13!` (zoals weergegeven in de onderstaande afbeelding), betekent dit dat het systeem nog niet is geïnstalleerd. Koppel SSH los en wacht een paar minuten om opnieuw in te loggen.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

Wanneer `Welcome to Ubuntu 22.04.1 LTS` verschijnt, is de initialisatie voltooid en kunt u doorgaan met de volgende stappen.

### [Optioneel] Initialiseer de ontwikkelomgeving

Deze stap is optioneel.

Voor het gemak heb ik de installatie en systeemconfiguratie van ubuntu-software in [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) gezet.

Voer de volgende opdracht uit om met één klik te installeren.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Chinese gebruikers, gebruik in plaats daarvan de volgende opdracht en de taal, tijdzone, etc. worden automatisch ingesteld.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo maakt IPV6 mogelijk

Schakel IPV6 in zodat SMTP ook e-mails met IPV6-adressen kan verzenden.

bewerk `/etc/sysctl.conf`

Wijzig of voeg de volgende regels toe

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Vervolg met [de contabo-tutorial: IPv6-connectiviteit toevoegen aan uw server](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Bewerk `/etc/netplan/01-netcfg.yaml` , voeg een paar regels toe zoals weergegeven in de onderstaande afbeelding (het standaardconfiguratiebestand van Contabo VPS heeft deze regels al, verwijder het commentaar).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

Vervolgens `netplan apply` om de gewijzigde configuratie van kracht te laten worden.

Nadat de configuratie is gelukt, kunt u `curl 6.ipw.cn` gebruiken om het ipv6-adres van uw externe netwerk te bekijken.

## Kloon de configuratierepository ops

```
git clone https://github.com/wactax/ops.soft.git
```

## Genereer een gratis SSL-certificaat voor uw domeinnaam

Voor het verzenden van e-mail is een SSL-certificaat vereist voor codering en ondertekening.

We gebruiken [acme.sh](https://github.com/acmesh-official/acme.sh) om certificaten te genereren.

acme.sh is een open source geautomatiseerde tool voor het ondertekenen van certificaten,

Voer het configuratiemagazijn ops.soft in, voer `./ssl.sh` uit en er wordt een `conf` map gemaakt in **de bovenste map** .

Zoek uw DNS-provider via [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , bewerk `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Voer vervolgens `./ssl.sh 123.com` uit om `123.com` en `*.123.com` certificaten voor uw domeinnaam te genereren.

Bij de eerste uitvoering wordt [acme.sh](https://github.com/acmesh-official/acme.sh) automatisch geïnstalleerd en wordt een geplande taak voor automatische verlenging toegevoegd. Je kunt `crontab -l` zien, er is zo'n regel als volgt.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

Het pad voor het gegenereerde certificaat is zoiets als `/mnt/www/.acme.sh/123.com_ecc。`

Certificaatvernieuwing roept `conf/reload/123.com.sh` script aan, bewerk dit script, u ​​kunt opdrachten toevoegen zoals `nginx -s reload` om de certificaatcache van gerelateerde applicaties te vernieuwen.

## Bouw een SMTP-server met chasquid

[chasquid](https://github.com/albertito/chasquid) is een open source SMTP-server geschreven in Go-taal.

Als vervanging voor de oude mailserverprogramma's zoals Postfix en Sendmail, is chasquid eenvoudiger en gemakkelijker te gebruiken, en het is ook gemakkelijker voor secundaire ontwikkeling.

Start `./chasquid/init.sh 123.com` wordt automatisch geïnstalleerd met één klik (vervang 123.com door uw verzendende domeinnaam).

## E-mailhandtekening DKIM configureren

DKIM wordt gebruikt om e-mailhandtekeningen te verzenden om te voorkomen dat brieven als spam worden behandeld.

Nadat de opdracht met succes is uitgevoerd, wordt u gevraagd om het DKIM-record in te stellen (zoals hieronder weergegeven).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Voeg gewoon een TXT-record toe aan uw DNS (zoals hieronder weergegeven).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Servicestatus en logboeken bekijken

 `systemctl status chasquid` Servicestatus bekijken.

De status van normaal bedrijf is zoals weergegeven in de onderstaande afbeelding

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` of `journalctl -xeu chasquid` kan het foutenlogboek bekijken.

## Omgekeerde domeinnaam configuratie

De omgekeerde domeinnaam is om het IP-adres te kunnen herleiden tot de corresponderende domeinnaam.

Door een omgekeerde domeinnaam in te stellen, kan worden voorkomen dat e-mails als spam worden geïdentificeerd.

Wanneer de e-mail is ontvangen, voert de ontvangende server een omgekeerde domeinnaamanalyse uit op het IP-adres van de verzendende server om te bevestigen of de verzendende server een geldige omgekeerde domeinnaam heeft.

Als de verzendende server geen omgekeerde domeinnaam heeft of als de omgekeerde domeinnaam niet overeenkomt met het IP-adres van de verzendende server, kan de ontvangende server de e-mail als spam herkennen of weigeren.

Ga naar [https://my.contabo.com/rdns](https://my.contabo.com/rdns) en configureer zoals hieronder weergegeven

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

Denk er na het instellen van de omgekeerde domeinnaam aan om de voorwaartse resolutie van de domeinnaam ipv4 en ipv6 naar de server te configureren.

## Bewerk de hostnaam van chasquid.conf

Pas `conf/chasquid/chasquid.conf` aan naar de waarde van de omgekeerde domeinnaam.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Voer vervolgens `systemctl restart chasquid` uit om de service opnieuw te starten.

## Back-up conf naar git-repository

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

Ik maak bijvoorbeeld als volgt een back-up van de conf-map naar mijn eigen github-proces

Maak eerst een privémagazijn

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Voer de conf-directory in en dien deze in bij het magazijn

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Afzender toevoegen

loop

```
chasquid-util user-add i@wac.tax
```

Kan afzender toevoegen

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Controleer of het wachtwoord correct is ingesteld

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

Na het toevoegen van de gebruiker wordt `chasquid/domains/wac.tax/users` bijgewerkt, vergeet niet om het in te dienen bij het magazijn.

## DNS voegt SPF-record toe

SPF (Sender Policy Framework) is een e-mailverificatietechnologie die wordt gebruikt om e-mailfraude te voorkomen.

Het verifieert de identiteit van een e-mailafzender door te controleren of het IP-adres van de afzender overeenkomt met de DNS-records van de domeinnaam die het beweert te zijn, waardoor wordt voorkomen dat fraudeurs valse e-mails verzenden.

Door SPF-records toe te voegen, kan zoveel mogelijk worden voorkomen dat e-mails als spam worden herkend.

Als uw domeinnaamserver het SPF-type niet ondersteunt, voegt u gewoon een TXT-type record toe.

De SPF van `wac.tax` is bijvoorbeeld als volgt

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF voor `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Merk op dat ik hier `include:_spf.google.com` heb, dit komt omdat ik `i@wac.tax` later zal configureren als het verzendadres in het Google-postvak.

## DNS-configuratie DMARC

DMARC is de afkorting van (Domain-based Message Authentication, Reporting & Conformance).

Het wordt gebruikt om SPF-bounces vast te leggen (misschien veroorzaakt door configuratiefouten, of iemand anders doet zich voor als u om spam te verzenden).

TXT-record `_dmarc` toevoegen,

De inhoud is als volgt

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

De betekenis van elke parameter is als volgt

### p (Beleid)

Geeft aan hoe om te gaan met e-mails die niet slagen voor verificatie door SPF (Sender Policy Framework) of DKIM (DomainKeys Identified Mail). De parameter p kan worden ingesteld op een van de volgende drie waarden:

* geen: er wordt geen actie ondernomen, alleen het verificatieresultaat wordt teruggekoppeld naar de afzender via het e-mailrapportagemechanisme.
* Quarantaine: plaats de mail die niet door de verificatie is gekomen in de map spam, maar zal de mail niet direct afwijzen.
* afwijzen: weiger e-mails die niet geverifieerd zijn direct.

### fo (faalopties)

Specificeert de hoeveelheid informatie die door het rapportagemechanisme wordt geretourneerd. Het kan worden ingesteld op een van de volgende waarden:

* 0: Rapporteer validatieresultaten voor alle berichten
* 1: Rapporteer alleen berichten die de verificatie niet doorstaan
* d: Rapporteer alleen mislukte domeinnaamverificaties
* s: rapporteer alleen SPF-verificatiefouten
* l: Rapporteer alleen DKIM-verificatiefouten

### rua & ruf

* rua (Reporting URI for Aggregate reports): E-mailadres voor het ontvangen van geaggregeerde rapporten
* ruf (Reporting URI for Forensic reports): e-mailadres om gedetailleerde rapporten te ontvangen

## Voeg MX-records toe om e-mails door te sturen naar Google Mail

Omdat ik geen gratis zakelijke mailbox kon vinden die universele adressen ondersteunt (Catch-All, kan alle e-mails ontvangen die naar deze domeinnaam worden verzonden, zonder beperkingen op voorvoegsels), heb ik chasquid gebruikt om alle e-mails door te sturen naar mijn Gmail-mailbox.

**Als u uw eigen betaalde zakelijke mailbox heeft, wijzigt u de MX niet en slaat u deze stap over.**

Bewerk `conf/chasquid/domains/wac.tax/aliases` , stel doorstuurmailbox in

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` geeft alle e-mails aan, `i` is het voorvoegsel van het e-mailadres van de verzendende gebruiker die hierboven is gemaakt. Om e-mail door te sturen, moet elke gebruiker een regel toevoegen.

Voeg dan het MX-record toe (ik verwijs hier direct naar het adres van de reverse domeinnaam, zoals weergegeven in de eerste regel in onderstaande figuur).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

Nadat de configuratie is voltooid, kunt u andere e-mailadressen gebruiken om e-mails te verzenden naar `i@wac.tax` en `any123@wac.tax` om te zien of u e-mails kunt ontvangen in Gmail.

Zo niet, controleer dan het chasquid-logboek ( `grep chasquid /var/log/syslog` ).

## Stuur een e-mail naar i@wac.tax met Google Mail

Nadat Google Mail de mail had ontvangen, hoopte ik natuurlijk te antwoorden met `i@wac.tax` in plaats van i.wac.tax@gmail.com.

Ga naar [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) en klik op "Nog een e-mailadres toevoegen".

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Voer vervolgens de verificatiecode in die is ontvangen in de e-mail waarnaar is doorgestuurd.

Ten slotte kan het worden ingesteld als het standaard afzenderadres (samen met de optie om met hetzelfde adres te antwoorden).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

Op deze manier hebben we de oprichting van de SMTP-mailserver voltooid en gebruiken we tegelijkertijd Google Mail om e-mails te verzenden en te ontvangen.

## Stuur een testmail om te controleren of de configuratie is gelukt

Voer `ops/chasquid` in

Voer `direnv allow` om afhankelijkheden te installeren (direnv is geïnstalleerd in het vorige initialisatieproces met één toets en er is een hook aan de shell toegevoegd)

ren dan

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

De betekenis van de parameters is als volgt

* gebruiker: SMTP-gebruikersnaam
* pass: SMTP-wachtwoord
* aan: ontvanger

U kunt een testmail sturen.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Het wordt aanbevolen om Gmail te gebruiken om testmails te ontvangen om te controleren of de configuraties succesvol zijn.

### TLS-standaardversleuteling

Zoals te zien is in de onderstaande afbeelding, is er dit kleine slotje, wat betekent dat het SSL-certificaat met succes is ingeschakeld.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Klik vervolgens op "Toon originele e-mail"

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

Zoals te zien is in de onderstaande afbeelding, geeft de oorspronkelijke e-mailpagina van Gmail DKIM weer, wat betekent dat de DKIM-configuratie is gelukt.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Controleer Ontvangen in de koptekst van de oorspronkelijke e-mail en u kunt zien dat het afzenderadres IPV6 is, wat betekent dat IPV6 ook met succes is geconfigureerd.
