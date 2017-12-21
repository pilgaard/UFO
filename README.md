# Forbedre sikkerheden på din MySQL database med nogle få ændringer #
## Introduktion ##

MySQL er et open-source database management system, og bliver brugt hele verdenen over. Ifølge iDatalabs, er MySQL en af de mest populære databasesystemer iblandt verdenens virksomheder, da 137.626 ud af 539.051 bruger MySQL, hvilket svarer til 24%.
Det er vigtigt at sikre MySQL, da data er meget værdifuld i nutidens verden, og hvis MySQL ikke bliver sikret, vil der være risiko for angreb.
Hvis vi ser på Revisionlegal’s indlæg omkring Data Breaches i 2017, kan vi se at selv de største firmaer døjer med sikkerhedsproblemer. F.eks var der 89 registrerede angreb i Januar 2017, hvor det største angreb ramte mange asiatiske firmaer, såsom “NetEase” og “Sina.com”. Dette angreb lækkede omkring 1,85 milliarder brugerkonti, hvis kontooplysninger blev sat til salg på the Dark Web.
Udover angreb på større firmaer, er selv de mindste applikationer i fare. Dette oplevede vi under udviklingen af vores LSD’er projekter, da en af grupperne var offer for en sikkerhedsfejl i MongoDB. Dette resulterede i at alt deres data blev krypteret, og at de skulle sende BitCoins til en addresse, for at få kyrpteringsnøglen til deres data

Hvis nogen får adgang til din data, kan det altså have konsekvenser i form af både tyveri eller gidseltagelse.
På grund af dette, vil vi forsøge at hjælpe, dig, læseren, med at øge sikkerheden i din MySQL ved at anvende nogle Best Practices indenfor MySQL sikkerhed.
## Bruger privilegier ##
En root user kan være meget farlig for hele din databases sikkerhed. 
Når den forkerte person får adgang til din database med root userens cerdentials har man adgang til alt, og er derfor ekstrem farlig for hele dit system's sikkerhed.
Ved at lave et par få ændringer til dine bruger privilegier kan du forbedre din sikkerhed markant. 
### Begræns adgang til databasen ###
I vores MySQL database har vi nogle forskellige brugere, som er sat op med forskellige indstillinger, og rettigheder.
Vi vil starte med at kigge på et par af dem, og hvordan vi kan ændre i dem for at forbedre sikkerheden.
Men først skal vi lige have adgang til databasen, det går vi vid at ssh ind på vores server 

`ssh root@your_droplet_ip`

Herefter kan vi logge ind på MySQL med vores root user

`mysql -u root -p`

Nu kan vi kigge på vores brugere, hvor de kan få adgang fra og om de har et password, det gør vi med den følgende kommando

`mysql> SELECT User,Host,authentication_string FROM mysql.user;`

![select users](/images/users.png)

Når vi kigger på vores brugere i vores database kan vi se, at alle brugere har et password, hvilket vi altid ønsker, dog kan vi se at vores admin bruger har et “%” udfra host, som er et wildcard, det betyder at brugeren kan benyttes fra enhver host adresse. Det er ikke det, vi ønsker. Vi vil i stedet for have at det er "localhost" eller ”127.0.0.1” 

Så hvis vi køre følgende kommando får vi denne besked:

![change output](/images/change.png)

`mysql> UPDATE mysql.user SET Host='127.0.0.1' WHERE Host="%";`
Vi kan nu se at der er blevet foretaget en ændring, og ved at køre vores select statement igen, vil vi kunne se at admin ikke længere har et procenttegn ud fra host.
På denne måde sikrer vi at alle databasens brugere kun kan tilgå databasen igennem localhost.
### Begræns brugernes rettigheder til databasen ###
Nu vil vi oprette en bruger til at administrere vores database til vores webservice og kun tildele brugeren de privilegier, der er nødvendige. Disse rettigheder vil variere fra system til system. I vores tilfælde ønsker vi kun at brugeren kan udføre SELECT og INSERT på vores lsd database. Det gør vi ved at oprette en ny bruger som hedder lsd-user. Vi vil kun have at brugeren kan tilgå databasen lokalt, og vi giver ham et password. 

`mysql> CREATE USER 'lsd-user'@'localhost' IDENTIFIED BY 'myPass';`

![createUser](/images/createUser.png)

Herefter skal vi tildele vores bruger nogle rettigheder, uden at gøre dette vil vores bruger ikke kunne gøre noget som helst.
Vi vil have at vores bruger at udføre select og insert statements på vores lsd database  

`mysql> GRANT SELECT,INSERT ON lsd.* to 'lsd-user'@'localhost';`

![grant](/images/grant.png)

Nu har vi fået tildelt rettigheder til vores lsd-user  og vi kan se dem ved at køre følgende kommando 

`mysql> SHOW GRANTS for 'lsd-user'@'localhost';`

![grants](/images/grants.png)

Når vi er færdig med at lave ændringer i privilegierne skal vi slutte af med flush

`FLUSH PRIVILEGES;`
Nu når vi har oprettet en bruger med begrænset rettigheder kan vi i stedet for vores root bruger benytte vores lsd bruger til at lave kald fra vores web service til vores database, på denne måde kan vi bedre styre hvilke handlinger der kan laves på databasen udefra,  hvilket er en stor forbedring i forhold til sikkerheden. Så hvis der er nogle der ønsker at foretage en handling på databasen som kun vores root user har adgang til, skal vedkommende først skaffe sig adgang til serveren. 
Vi kan også vælge at give vores root user et nyt navn, hvilket gør at en hacker der har fået adgang til serveren skal arbejde lidt mere for at få adgang til mysql, da vi ikke længere bruger standard navnet. det gør vi med følgende statement.
`rename user 'root'@'localhost' to 'newAdminUser'@'localhost';`

Og efterfølgende bruger vi flush.

`FLUSH PRIVILEGES;`

#### Hvad hvis min webservice er scaled ####

I tilfælde af at man har en app der er scaled eller vil have muligheden for at kunne scale, kan vi ikke begrænse adgangen til localhost.
I stedet for må vi begrænse adgangen til et privat netværk, hvis du ikke har styr på hvordan du gør det kan du finde en guide [her](https://www.digitalocean.com/community/tutorials/how-to-set-up-and-use-digitalocean-private-networking) som viser dig hvordan du får sat det op med digitalocean.

Vi har en privat ip der hedder 46.101.###.###, den kan findes ved at skrive ifconfig i serverens terminal. (Vi har udskiftede den sidste halvdel af ip adressen med hashtags da vi vil holde ip adressen til os selv.)

De to første oktetter udgør netværksadressen, og de to andre er til hosten. Hver gang du starter en ny droplet op i den nuværende region, vil den private IP-adresse se sådan ud 46.101.*. * hvor de to sidste oktetter er unikke for dropletten. Brug af et % wildcard i MySQL-hosten sikrer, at alle droplets på det private netværk kan tilsluttes.
Vi kan derfor opdatere vores bruger til at have adgang fra hele det private netværk frem for på det lokale netværk.

`mysql> UPDATE mysql.user SET Host='46.101.%.%' WHERE user="lsd-user";`
`FLUSH PRIVILEGES;`

Nu har vi sat op sådan at det kun er vores lsd-user som kan få adgang til databasen fra hele det private netværk, hvorimod alle andre brugere stadig skal have forbindelse fra localhost. 
## my.cnf ##
Tilføjelser/ændringer under sektionen “[mysqld]” i konfigurationsfilen “my.cnf” i stien: 
“C:\Program Files\MySQL\MySQL Server” (windows)  
 “/etc/mysql/my.cnf” (linux) 
### Begræns adgang til databasen ###
Såfremt det ikke er nødvendigt at tilgå databasen direkte igennem internettet, anbefales det at man begrænser eller fuldstændig lukker for adgangen til databasen fra andre kilder end localhost. For at begrænse adgangen kan man tilføje:

`skip-networking`

Med denne kommando stopper MySQL med at lytte på alle TCP/IP porte og herefter benyttes MySQL’s socket-baseret forbindelse til at opnå forbindelse fra localhost, men det er også muligt at tvinge MySQL til udelukkende at lytte til localhost ved at benytte: 

`bind-address=127.0.0.1`
### Slå “LOAD DATA LOCAL INFILE” kommando fra ###
Det er muligt at benytte sig af denne kommando til at opnå adgang til lokale filer, eksempelvis:

`mysql> LOAD DATA LOCAL INFILE '/etc/passwd' INTO TABLE table1`

Derfor anbefales det er slå den fra, for at begrænse angreb som eksempelvis SQL-injections, ved at tilføje:


	`set-variable=local-infile=0`

Ønsker man ikke at slå kommandoen fra, eksempelvis fordi man indlæser data fra .csv filer, så anbefales det at benytte en krypteret SSL forbindelse, ved at sætte ssl-mode til “VERIFY_IDENTITY” med:	

 	`--ssl-mode=VERIFY_IDENTITY`

Det er generelt en god idé at bruge en form for krypteret forbindelse, så man kan undgå session high-jacking eller lignende. Hvis MySQL ligger på en server og udelukkende kommunikerer med serveren via localhost, så er det ikke nødvendigt at bruge en krypteret forbindelse mellem MySQL og serveren. 
### Rettigheder ###
Det er vigtigt at sikre sig at skrive-rettighederne til “my.cnf” filen udelukkende tilhører root, ellers øges chancen for at uvedkommende får adgang MySQL serveren.

## Separat database og server ##
Vi har indtil videre forsøgt at gøre MySQL mere sikker ved bl.a. at begrænse kommunikationen til kun at ske via localhost, hvilket kræver at MySQL databasen er installeret på serveren og dermed i samme “miljø”. I nogle tilfælde vil det være mere optimalt at lave en løsning, hvor serveren og databasen ligger hver for sig og kommunikere via internettet. For at sikre sådan en løsning kan man bl.a.:

Ændre standardporten (3306) til forbindelser til databasen

Ved at ændre porten opnår man ikke som sådan en højere sikkerhed, men forsøger i stedet at “sløre” adgangen overfor diverse botnets og andre ondsindede entiteter, som kun checker på port 3306. Dette kaldes også port-obfuscating.

Specificere hvilke ip’er der kan forbinde til databasen

Ved hjælp af IPTables kan man “whiteliste” den eller de ip’er som brugere må forbinde fra, og på den måde udelukke alle med en ip der ikke er “whitelisted”

SSL

Det er væsentligt at kryptere forbindelsen mellem en server og databasen, når den ikke kun kommunikerer via localhost. For at tilføje SSL til vores forbindelse, skal vi igen ned i konfigurationsfilen “my.cnf” under sektionerne “mysqld” og “client”. Her skal der bl.a. specificeres et certifikat og en nøgle, så brugeren kan være sikker på hvem hosten(databasen) er og vice versa.

## Konklusion ##
Med disse relativt få ændringer vil MySQL være væsentlig mere sikker, og er et godt sted at starte. Disse ændringer er som sagt, Best Pracices, og vil sikrer imod “typiske” angreb, som man kan opleve, hvis man er bruger af MySQL. Ønsker man at sikre MySQL endnu mere, så er du blevet introduceret for my.cnf, og hvordan man kan tilføje nye sikkerhedsforanstaltning. For at måle præcis hvor meget mere sikker MySQL er blevet, kan man foretage før og efter penetrationstests. Disse resultater kan afsløre på hvilke områder, man kan sikre MySQL yderligere udover disse Best Practices.

## Kilder: ##
 
* https://idatalabs.com/tech/database-management-system 

* https://www.digitalocean.com/community/tutorials/how-to-secure-mysql-and-mariadb-databases-in-a-linux-vps

* https://www.digitalocean.com/community/tutorials/how-to-set-up-and-use-digitalocean-private-networking

* https://deliciousbrains.com/scaling-wordpress-dedicated-database-server/

* https://www.percona.com/blog/2013/06/22/setting-up-mysql-ssl-and-secure-connections/

* https://security.stackexchange.com/questions/115008/securing-mysql-server-from-remote-connection

* https://dev.mysql.com/doc/refman/5.7/en/

* https://revisionlegal.com/data-breach/2017-security-breaches/ 

* https://www.upguard.com/articles/top-11-ways-to-improve-mysql-security
