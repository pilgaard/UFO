# Forbedre sikkerheden på din MySQL database med nogle få ændringer #
## Introduktion ##

MySQL er et open-source database management system, som bliver brugt hele verdenen over. Dette skyldes at det konsistent har høj performance, pålidelighed og er nem at bruge.
Som mange andre produkter, der kommer out-of-the-box, er sikkerhed ikke en af første overvejelser når man installerer MySQL. Da førsteprioriteten er at få systemet op at køre, så virksomheden kan få værdi af det nye databasesystem.
Hvis MySQL ikke bliver sikret i et produktionsmiljø, kan det have voldsomme konsekvenser for din data, da man aldrig kan vide hvem der får adgang til din data, hvilket med næsten garanti vil udvikle sig til for eksempel ransomware eller data theft.
Det er derfor vigtigt at både MySQL og det miljø som MySQL findes i er sikret, så ovenstående kan undgås.

Der er mange forskellige metoder man kan bruge for at sikre MySQL og sin server. I denne blog vil vi forsøge at dække nogle best practices man kan anvende, for at sikre sin database så godt som muligt.
## Bruger privilegier ##
En root user kan være meget farlig for hele din databases sikkerhed. 
Når den forkerte person får adgang til din database med root userens cerdentials har man adgang til alt, og er derfor ekstrem farlig for hele dit system's sikkerhed.
Ved at lave et par få ændringer til dine bruger privilegier kan du forbedre din sikkerhed markant. 
### Begræns adgang til databasen ###
I vores MySQL database har vi nogle forskellige brugere, som er sat op med forskellige indstillinger, og rettigheder.
Vi vil starte med at kigge på et par af dem, og hvordan vi kan ændre i dem for at forbedre sikkerheden.

Først vil vi kigge på vores brugere, hvor de kan få adgang fra og om de har et password, det gør vi med den følgende kommando

`mysql> SELECT User,Host,authentication_string FROM mysql.user;`

![select users](/images/users.png)

Når vi kigger på vores brugere i vores database kan vi se, at alle brugere har et password, hvilket vi altid ønsker. Dog kan vi se at vores admin bruger har et “%” udfra host, som er et wildcard. Det betyder at brugeren kan benyttes fra enhver host adresse. Det er ikke det, vi ønsker. Vi vil i stedet for have at det er "localhost" eller ”127.0.0.1” 

Så hvis vi køre følgende kommando får vi denne besked:

![change output](/images/change.png)

`mysql> UPDATE mysql.user SET Host='127.0.0.1' WHERE Host="%";`
Vi kan nu se at der nu er blevet foretaget en ændring. Hvis vi kører vores select statement igen vil vi kunne se, at admin ikke længere har et procenttegn ud fra host.
Nu har vi sørget for at vores database kun kan tilgås lokalt, og man skal derfor have adgang til serveren før man kan få adgang til databasen.
### Begræns brugernes rettigheder til databasen ###
Efterfølgende vil vi oprette en bruger til at administrere vores database til vores webservice og tildele brugeren kun de privilegier, der er nødvendige. Disse rettigheder vil variere fra system til system. 
I vores tilfælde vil vi kun have at vores bruger kan udføre SELECT og INSERT på vores lsd database.
Det gør vi ved først at oprette en ny bruger som hedder lsd-user. Vi vil kun have at brugeren kan tilgå databasen lokalt, og vi giver ham et password 

`mysql> CREATE USER 'lsd-user'@'localhost' IDENTIFIED BY 'myPass';`

![createUser](/images/createUser.png)

Herefter skal vi tildele vores bruger nogle rettigheder. Uden at gøre dette vil vores bruger ikke kunne gøre noget som helst.
Vi vil have at vores bruger at udføre select og insert statements på vores lsd database  

`mysql> GRANT SELECT,INSERT ON lsd.* to 'lsd-user'@'localhost';`

![grant](/images/grant.png)

Nu har vi fået tildelt rettigheder til vores lsd-user  og vi kan se dem ved at køre følgende kommando 

`mysql> SHOW GRANTS for 'lsd-user'@'localhost';`

![grants](/images/grants.png)
Nu når vi har oprettet en bruger med begrænset rettigheder, kan vi i stedet for vores root bruger, benytte vores lsd bruger til at lave kald fra vores web service til vores database, på denne måde kan vi bedre styre hvilke handlinger der kan laves på databasen udefra.   
## my.cnf ##
Tilføjelser/ændringer under sektionen “[mysqld]” i konfigurationsfilen “my.cnf” findes i stien: 
“C:\Program Files\MySQL\MySQL Server” (windows)  
 “/etc/mysql/my.cnf” (linux) 
### Begræns adgang til databasen ###
Såfremt det ikke er nødvendigt at tilgå databasen direkte igennem internettet, anbefales det at man begrænser eller fuldstændig lukker for adgangen til databasen fra andre kilder end localhost. For at begrænse adgangen kan man tilføje:

`skip-networking`

Med denne kommando stopper MySQL med at lytte på alle TCP/IP porte, og herefter benyttes MySQL’s socket-baseret forbindelse til at opnå forbindelse fra localhost. Det er også muligt at tvinge MySQL til udelukkende at lytte til localhost ved at benytte: 

`bind-address=127.0.0.1`
### Slå “LOAD DATA LOCAL INFILE” kommando fra ###
Det er muligt at benytte sig af denne kommando til at opnå adgang til lokale filer, eksempelvis:

`mysql> LOAD DATA LOCAL INFILE '/etc/passwd' INTO TABLE table1`

Derfor anbefales det er slå den fra, for at begrænse angreb som eksempelvis SQL-injections, ved at tilføje:


	`set-variable=local-infile=0`

Ønsker man ikke at slå kommandoen fra, eksempelvis fordi man indlæser data fra .csv filer, så anbefales det at benytte en krypteret SSL forbindelse, ved at sætte ssl-mode til “VERIFY_IDENTITY” med:	

 	`--ssl-mode=VERIFY_IDENTITY`

Det er generelt en god idé at bruge en form for krypteret forbindelse, så man kan undgå session high-jacking eller lignende. 
### Rettigheder ###
Det er vigtigt at sikre sig at skrive-rettighederne til “my.cnf” filen udelukkende tilhører root, ellers øges chancen for at uvedkommende får adgang MySQL serveren.

 Kilder:  
https://www.digitalocean.com/community/tutorials/how-to-secure-mysql-and-mariadb-databases-in-a-linux-vps

https://dev.mysql.com/doc/refman/5.7/en/

http://www.hexatier.com/mysql-database-security-best-practices-2/

https://www.upguard.com/articles/top-11-ways-to-improve-mysql-security  

http://www.informationisbeautiful.net/visualizations/worlds-biggest-data-breaches-hacks/
