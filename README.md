![alt text](https://dashboard.uberspace.de/static/img/logo-trans-2012.png)

# IPv4-to-IPv6-Only-Portmapper
## Erreichen von IPv6-Diensten (z.B. DS-Lite, Unitymedia, Kabel BW) über IPv4  


__Problem:__

* Mit Dual-Stack Lite (DS-Lite, z.B. mit Unitymedia, Kabel BW) sind keine IPv4-basierenden Portfreigaben mehr möglich  
* Dienste die an einem DS-Lite-Anschluss angeboten werden, können also nur aus IPv6 Netzen erreicht werden
* Eine ganze Menge Mobilfunk- und Hotelnetze bieten aber immer nur noch ausschließlich IPv4 an. 
* Daher ist z.B. ohne weiteres keine VPN-Verbindung nach Hause möglich, kein Zugriff auf ein NAS oder Smart Home Dienste :-(

Man könnte sich einen Dienst mieten, der das Mapping für einen übernimmt, z.B. bei [feste-ip.net](http://www.feste-ip.net/dslite-ipv6-portmapper/allgemeine-informationen/) - oder man benutzt:

_Bester Hoster wo gibt™_ - [Uberspace](https://uberspace.de)

__SOCAT to the rescue:__  

Uberspace unterstützt sowohl IPv4 als auch IPv6 und bietet sich damit als bridge an. [SOCAT](http://www.dest-unreach.org/socat/) ist ein Multipurpose relay (aka netcat auf crack) und läßt sich auch ohne root-Rechte kompilieren & nutzen. Das nehmen wir um einen listening port auf der Uberspace-ipv4 zu starten und von dort zu dem IPv6-Gerät (zuhause) zu bridgen.

__Annahmen__

* Wir gehen davon aus, dass Dein Dienst zuhause bereits funktioniert  
* Wir nehmen an, Du möchtest zuhause auf [SSH](https://de.wikipedia.org/wiki/Secure_Shell) Port 22 verbinden. Natürlich geht das ganze aber auch für OpenVPN (UDP Port 1194) o.ä.!     
* Deine Heim-Adresse  ist _zuhause.org_ bzw. IPv6 _[dead:beef:ca1f]_  
* Dein Uberspace-Host heisst _Nutella_ , also nutella.uberspace.de

__Einrichtung auf Deinem Uberspace__  

Zuerst brauchen wir einen offenen Port in der Firewall von Deinem Uberspace. Dafür haben die fleissigen [ubernauten](https://jonaspasche.com/app/about_us) ein script gebaut, welches uns einen [unprivilegierten Port](https://manual.uberspace.de/basics-ports.html) zuweist:

`uberspace port add`

Antwort:  
`Port 65324 will be open for TCP and UDP traffic in a few minutes.`  
Bestens. Wir haben also Port _65324_ erhalten. Den merken wir uns. Du wirst sehr wahrscheinlich einen anderen erhalten.

Wechsel in Dein Home-Verzeichnis:  
`cd ~`  
Jetzt SOCAT herunterladen:  
`wget http://www.dest-unreach.org/socat/download/socat-2.0.0-b9.tar.gz`

Achtet bitte darauf, dass Ihr aus Sicherheitsgründen immer eine [aktuelle Version](http://www.dest-unreach.org/socat/download/) ladet & benutzt!

Entpacken der heruntergeladenen Datei:  
`tar xfvz socat-2.0.0-b9.tar.gz`

In das Verzeichnis wechseln:  
`cd socat-2.0.0-b9`

Kompilieren:  
`./configure --prefix=/home/$USER`  
`make`  
`make install`

Wieder ins Home wechseln  
`cd ~`  
Das Installationsverzeichnis & das Archiv können jetzt weg:  
`rm -rf socat-2.0.0-b9`  
`rm socat-2.0.0-b9.tar.gz`  

Jetzt möchten wir das ganze noch als [Daemon](https://wiki.uberspace.de/system:daemontools) einrichten, damit es dauerhaft und automatisch läuft. 

Allgemeine Service Einrichtung:    
`uberspace-setup-svscan`

Der Daemon soll bin/socat mit den gewünschten Parametern starten, also den richtigen Ports und der Zieladresse.

Wichtig: Im Folgenden also statt _65324_ den Port eintragen, den das Uberspace-Script _Dir_ zugewiesen hat - und natürlich Deine richtige Zuhause-Adresse anstatt _zuhause.org_

Wir nutzen das Script "uberspace-setup-service" um das als Dienst Namens "socat" einzurichten:  
`uberspace-setup-service socat ~/bin/socat TCP4-LISTEN:65324,fork TCP6:zuhause.org:22` 

(Alternativ:Du könntest hier natürlich auch eine ipv6 Adresse angeben.. z.B. 
`uberspace-setup-service socat ~/bin/socat TCP4-LISTEN:65324,fork TCP6:[dead:beef:ca1f]:22`)

Antwort:  
`Creating the ~/etc/run-socat/run service run script`  
`Creating the ~/etc/run-socat/log/run logging run script` 
`Symlinking ~/etc/run-socat to ~/service/socat to start the service` 
`Waiting for the service to start ... 1 2 3 started!` 

`Congratulations - the ~/service/socat service is now ready to use!`  

Der Dienst ist damit eingerichtet. Jetzt starten wir den ganzen Salat mit:  
`svc -u ~/service/socat`  

Wir versichern uns noch , das der Dienst läuft mit:  
`ps aux | grep socat`  

`..`   
` /home/dein.uberspace.account/bin/socat TCP4-LISTEN:65324,fork TCP6:zuhause.org:22`  
`..` 

Sieht gut aus.  
Wenn Du jetzt aus dem ipv4-Netz zu Deinem Uberspace-Server auf Port 65324 verbindest, z.B. mit

`ssh user@nutella.uberspace.de -p 65324`  

landest Du direkt in Deinem Heimnetz auf ipv6 Port 22 :-)  
Wie oben geschrieben, das Ganze geht natürlich auch für OpenVPN o.ä., Du musst nur Port & Protokoll anpassen. 
