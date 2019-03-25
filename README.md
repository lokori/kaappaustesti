# Tervetuloa

## Aluksi

Päivitetään palvelimen paketit ja pakettitietokanta.

```
yum update
yum upgrade
```


## Käydään läpi palvelimen status 

LinEnum.sh ja vastaavat skriptit ajavat käytännössä joukon komentoja, joilla tutkitaan palvelimen konfiguraatiossa olevia virheitä. Hakkeri/hyökkääjä voi ajaa nämä komennot myös manuaalisesti, mutta ylläpitäjälle enumerointiskripti on kätevä työkalu konfiguraation nopeaan tarkastamiseen.

LinEnum: https://github.com/rebootuser/LinEnum/blob/master/LinEnum.sh

Skripti tarkastaa samoja asioita, joita tässä ohjeessa käsitellään muutenkin. Esimerkiksi:
* cron-jobeja
* 


## Tiedostojärjestelmä

```
df -h
cat /etc/fstab
```

1. suid/guid -binaarit

```
find / \( -perm -4000 -o -perm -2000 \) -print
find / -path -prune -o -type f -perm +6000 -ls
```

2. tiedostot joihin kaikilla on lukuoikeus

3. tiedostot joihin kaikilla on kirjoitusoikeus
```
find /dir -xdev -type d \( -perm -0002 -a ! -perm -1000 \) -print
```

4. omistajattomat tiedostot
```
find /dir -xdev \( -nouser -o -nogroup \) -print
```

5. /tmp /backup ja vastaavien hakemistojen oikeudet ja sisältö


## Tunnukset ja kirjautumiset

1. Rajoitetaan login niiltä pois jotka eivät tarvitse

```
pwner:x:1002:1003::/home/pwner:/sbin/nologin
```

2. Poistetaan root login mahdollisuus

/etc/ssh/sshd_config
```
PermitRootLogin no
```

3. Rajoitetaan login vain ssh-avaimilla kirjautumiseen, passwd login pois
/etc/ssh/sshd_config
```
PasswordAuthentication no
```

4. Rajoitetaan sudoa

```
cat /etc/sudoers
```

erityisesti ALL ALL NOPASSWD -tyyppisiä määrittelyitä voi pitää vaarallisina. Jos kehittäjä tai joku tarvitsee oikeuden ajaa yksittäinen komento root-oikeuksilla, annetaan oikeus ajaa vain se komento.

Tarkkuutta myös siinä voiko ajettavan komennon tulokseen vaikuttaa muuta kautta - esimerkiksi ympäristömuuttujien kautta tai kirjoittamalla johonkin tiedostoon muutoksia.

```visudo``` tarvittaessa tiedoston editointiin.

5. cron-ajojen tarkastaminen

```
crontab -l -u pwner
```

listaa käyttäjän ```pwner``` ajastetut ajot. Erityisesti root-käyttäjälle ajastetut komennot voivat olla turvallisuusriski ja aiheuttaa myös virhetilanteissa yllätyksiä.


6. Huonojen salasanojen automaattinen tarkastaminen
7. Kotihakemistojen (erityisesti /root) näkyvyydet
8. "Efektiiviset" sudot ja vaaralliset user groupit

Käyttäjätunnus esimerkiksi ryhmässä ```disk``` tai ```docker``` tarkoittaa sitä käytännössä että käyttäjällä on tosiasiallisesti root-oikeudet koneeseen.

Ryhmä ```video``` tarkoittaa että käyttäjä voi lukea näyttömuistia jne. 

Normaalisti palvelinympäristössä käyttäjillä ei pitäisi olla mitään ylimääräisiä ryhmiä, joten tämän pitäisi olla nopea tarkistaa.

## Yhteydet ja palvelut

1. Mitä yhteyksiä ja palvelinprosesseja on?

```
netstat -tlnup
```
```
sudo service --status-all
```


2. Sammutetaan turhat palvelut

Mikä tahansa palvelu, joka ei ole tarpeellinen, kannattaa sammuttaa. Hyökkäyspinta-alan minimointia.


3. Onko palvelu sidottu localhostiin jos sen ei ole tarkoitus kuunnella yhteyksiä ulkoa?

Jos palvelun on tarkoitus kuunnella yhteyksiä vain localhostista, se on syytä konfiguroida niin. Esimerkiksi tietokantapalvelin on usein sellainen että yhteyksiä on tarkoitus ottaa vain paikalliselta prosessilta (tai sisäverkosta), joten sen ei pidä kuunnella yhteyksiä ulkomaailmasta. Palomuuri lisäksi estää yhteyde ulkomaailmasta, mutta on parempi minimoida virhemahdollisuudet ja lisätä kerroksia.


4. Rajoitetaan palomuurista yhteyksiä sisään ja ulos

Listataan säännöt
```
sudo iptables -L
```

HUOM: iptables säännöillä on mahdollista tehdä vahinkoa itselleen:
* voit lukita itsesi vahingossa ulos koneesta.
* päivitysten hakeminen netistä voi muuttua mahdottomaksi.
* saatat häiritä jonkun olennaisen prosessin toimintaa (esim. ntpd)

iptables konfiguroinnin pääajatuksia:

* iptables jakaa liikenteen jonoihin (IN, OUT, FORWARD tai omat jonot)
* jokaisen jonon säännöt käsitellään järjestyksessä, eli muokatessa pitää olla tarkkana mihin kohtaan sääntö tulee.
* tyypillisesti ensin on säännöt hyväksytylle liikenteelle ja viimeisenä on yleinen sääntö "pudota kaikki paketit".
* DENY pudottaa paketin ja kuittaa lähettäjälle. DROP pudottaa paketin kuittaamatta. On makuasia kumpi on parempi.

Esimerkkejä säännöistä ja komennoista:

Salli SSH sisään tietyistä ip-osoitteista:
sudo iptables -A INPUT -p tcp -s 15.15.15.0/24 --dport 22 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 22 -m conntrack --ctstate ESTABLISHED -j ACCEPT

Salli http/https sisään:
sudo iptables -A INPUT -p tcp -m multiport --dports 80,443 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp -m multiport --dports 80,443 -m conntrack --ctstate ESTABLISHED -j ACCEPT

Salli paikallinen loopback:
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT


5. Fail2ban

Redhat/CentOS kohdalla pitää enabloida EPEL-repository, jossa on lisää paketteja standardivalikoiman lisäksi:
```
sudo yum install epel-release
```

tai AWS:n Redhat-palvelimen tapauksessa:

```
sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
```

Asennetaan fail2ban:
```
sudo yum install fail2ban
```

Käynnistetään:
```
sudo service fail2ban start
```

Asetustiedostoilla voidaan muokata toimintaa. Oletusasetuksilla fail2ban EI koske palomuuriin.
```
/etc/fail2ban/fail2ban.conf
/etc/fail2ban/jail.conf
```

muokataan ssh "jail" käyttöön
```
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
emacs /etc/fail2ban/jail.local
```
restart
```
sudo service fail2ban restart
```

7. Siirretään SSH eri porttiin

Voi aiheuttaa myös ongelmia, riippuu oman järjestelmän palomuureista ja reitityksistä. Mutta vähentää huomattavasti kolkuttelua jos SSH vastaa täysin epästandardissa portissa ja kone pudottaa muuten paketit, eikä vastaa niihin lainkaan.

(iptables säännöt pitää konfiguroida uudelleen jos on rajoittanut SSH-yhteyksiä)


## WWW-palvelin (Apache)

1. Tarkistetaan onko www rootin alla jotain turhaa vanhaa tavaraa

2. Konfiguroidaan HTTP pois Apachesta, vain HTTPS sallittu

RewriteEngine on
RewriteCond %{SERVER_NAME} =kovennus.com [OR]
RewriteCond %{SERVER_NAME} =www.kovennus.com
RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]


3. Konfiguroidaan Apachea tiukemmaksi

ServerTokens Prod 
ServerSignature Off 
TraceEnable Off 
Header always unset X-Powered-By

Oletusarvo hakemistolistauksille on sallia ne. Kielletään hakemistolistausten kautta tiedostojen etsiminen.
```
Options all -Indexes 
```


## Lopuksi

1. Konfiguroidaan SELinux käyttöön
2. Lokitus
3. Varmuuskopiot

