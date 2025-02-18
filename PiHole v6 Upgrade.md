# Das Upgrade selbst

```bash
sudo apt update
sudo apt upgrade -y
sudo pihole -up
```

Das `sudo pihole -up` bzw. das Gravity Update dauerte extrem lange und hat den kompletten Raspberry lahmgelegt (keine neuen SSH Sessions mehr möglich)

Der Raspberry Pi musste dann nach 30 Minuten vergeblichem warten Power gecyclet werden. Also abstecken und wieder anstecken.
Als er fertig gebootet war konnte festgestellt werden, dass das Update erfolgreich war. v6 lief bereits. Auch der lighttdp Webserver war weg.
Kontrolliert mit folgendem Command und durch Aufrufen der WebGUI.
```bash
sudo systemctl --type=service
```

Zur Sicherheit habe ich das natürlich kontrolliert und auch nochmal die drei Update Commands ausgeführt.
```bash
sudo apt update
sudo apt upgrade -y
sudo pihole -up
```

Evtl. werde ich beim zweiten PiHole einmal vor dem Upgrade die Blocklisten durchgehen und kontrollieren ob es welche gibt die `inaccessible during last run` sind und diese entfernen - dann erst upgraden. Ob es hilft wird sich zeigen.

# Nacharbeiten
## SSL Cert von Certbot wieder einspielen
### Vorgeschichte zu meinem Setup mit PiHole v5
Der neue integrierte Webserver von PiHole v6 hat nun standardmäßig HTTPS Support und ein eigenes SSL Cert. Das ganze Zertifikat kann man natürlich austauschen, wenn man möchte. Jedoch braucht der neue Webserver ein etwas anderes Format.

*Für meine Docker Services verwende ich den traefik Reverse Proxy der mir über eine DNS-01 Challenge bei Cloudflare die SSL-Zertifikate von Lets Encrypt holt und aktuell hält. (Hier ein Video dazu: [Title Unavailable \| Site Unreachable](https://www.youtube.com/watch?v=-hfejNXqOzA&t=23s)). Für Pi-Hole mache ich aber was anderes.*

Für meine pi-hole Instanzen wird das ganze mit certbot automatisiert. Vor dem Upgrade auf v6 war es mit lighttpd als Webserver nicht sonderlich kompliziert. Im groben so:
> **Pihole v5 certbot setup Vorgang:**
> 
> Genaue Details den Quellen entnehmen:
> https://certbot.eff.org/instructions?ws=other&os=pip&tab=wildcard
> https://certbot-dns-cloudflare.readthedocs.io/en/stable/
>
> Commands:
>```bash
>mkdir -p .secrets/certbot
>vim .secrets/certbot/cloudflare.ini
>chmod -R 600 .secrets/
>sudo python3 -m venv /opt/certbot/
>sudo apt install python3 python3-venv libaugeas0
>sudo python3 -m venv /opt/certbot/
>sudo /opt/certbot/bin/pip install --upgrade pip
>sudo /opt/certbot/bin/pip install certbot
>sudo ln -s /opt/certbot/bin/certbot /usr/bin/certbot
>sudo /opt/certbot/bin/pip install certbot-dns-cloudflare
>service lighttpd stop
>sudo certbot certonly --dns-cloudflare --dns-cloudflare-credentials .secrets/certbot/cloudflare.ini --dns-cloudflare-propagation-seconds 60 -d raspberrypi.gmpfeifer.at -d pi.gmpfeifer.at -d pihole.gmpfeifer.at
>sudo nano /etc/lighttpd/external.conf
>sudo service lighttpd restart
>echo "0 0,12 * * * root /opt/certbot/bin/python -c 'import random; import time; time.sleep(random.random() * 3600)' && sudo certbot renew -q" | sudo tee -a /etc/crontab > /dev/null
>```

### Anpassungen für PiHole v6
Damit das mit dem neuen integrierten Webserver in PiHole v6 wieder funktioniert mussten einige Anpassungen gemacht werden.

Am besten schaut man sich das neue zentrale Config File von PiHole an unter `/etc/pihole/pihole.toml` und man sieht was man machen kann um seine eigenen Zertifikate zu verwenden.

**Speziell diesen Hinweis:**


Certbot legt die Zertifikate hier ab:
```bash
/etc/letsencrypt/live/pihole.example.com/fullchain.pem
/etc/letsencrypt/live/pihole.example.com/privkey.pem
```

Also folgendes als root gemacht:
```bash
cat /etc/letsencrypt/live/raspi-vie.gmpfeifer.at/fullchain.pem /etc/letsencrypt/live/raspi-vie.gmpfeifer.at/privkey.pem > /etc/pihole/server.pem
```

Dann service restartet:
```bash
systemctl restart pihole-FTL
```


Dann Änderungen im `etc/pihole/pihole.toml` hier im Abschnitt `[webserver]` bei der Variable `domain`:
![[Pasted image 20250218231609.png]]
Und dann auch unter `[webserver.tls]` bei der Variable `cert`:
![[Pasted image 20250218223635.png]]
Speichern und schließen


Dann wollte ich noch, dass da ganze auch dauerhaft funktioniert. Mal sehen ob diese crontab Lösung robust genug ist dafür:
```bash
0  5	1 * * root /opt/certbot/bin/python -c 'import random; import time; time.sleep(random.random() * 3600)' && sudo certbot renew -q
15 5	1 * * cat /etc/letsencrypt/live/raspi-vie.gmpfeifer.at/fullchain.pem /etc/letsencrypt/live/raspi-vie.gmpfeifer.at/privkey.pem > /etc/pihole/server.pem
20 5	1 * * systemctl restart pihole-FTL
```
