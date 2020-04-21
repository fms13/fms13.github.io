= Freifunk as KVM Guest on a Debian 10 Host =

Dieses Tutorial beschreibt, wie man ein Freifunk-Image in einer virtuellen Maschine auf Debian 10 als Hostsystem installiert.

Dabei kann der interne WLAN-Adapter als Access Point eingesetzt werden, wenn das Hostsystem über eine weitere Netzwerkschnittstelle an das Internet angebunden ist.

Als Virtualisierungs-Lösung wird der Einsatz von KVM beschrieben und die Einrichtung via libvirt.

Die folgenden Schritte werden beschrieben:

  - Installation der benötigten Pakete
  - Konfiguration des Systems
  - Installation des ffmuc x86 64-Bit Images (gluon)
  - Installation und Konfiguration von gluon
  - Anpassen der Netzwerkschnittstellen
  - Nutzen des internen WLAN-Adapters als Access Point

== Installation der benötigten Pakete ==

Um KVM, libvirt und die Tools zum Verwalten von Netzwerkbrücken zu installieren, sind folgende Befehle auszuführen:

  apt-get install qemu-kvm libvirt-clients libvirt-daemon-system
  apt-get install virtinst virt-top
  apt-get install bridge-utils

== Konfiguration des Systems ==

Damit ein Benutzer KVM konfigurieren kann, muss er der Gruppe libvirt angehören:

  adduser <Benutzername> libvirt

Jetzt werden drei Netzwerkbrücken eingerichtet:

  - ffmuc-setup
  - ffmuc-clients
  - ffmuc-mesh

Die folgenden Zeilen in /etc/network/interfaces erledigt das. Die Datei muss als root editiert werden.

  # Bridge fuer das Einrichten von gluon:
  auto ffmuc-setup
  iface ffmuc-setup inet static
      bridge_ports none
      address 192.168.1.2
      broadcast 192.168.1.255
      netmask 255.255.255.0
      gateway 192.168.1.1
      bridge_stp off # deaktiviere das sapnning tree protocol
      bridge_waitport 0 # keine Verzoegerung bevor ein Port verfuegbar wird
      bridge_fd 0 # kein Forwarding Delay

  # Bridge für das Anbinden der Clients:
  auto ffmuc-clients
  iface ffmuc-clients inet manual
      bridge_ports none
      bridge_stp off # deaktiviere das sapnning tree protocol
      bridge_waitport 0 # keine Verzoegerung bevor ein Port verfuegbar wird
      bridge_fd 0 # kein Forwarding Delay

  # Bridge fuer die gluon-Meshfunktion:
  auto ffmuc-mesh
  iface ffmuc-mesh inet manual
      bridge_ports none
      bridge_stp off # deaktiviere das sapnning tree protocol
      bridge_waitport 0 # keine Verzoegerung bevor ein Port verfuegbar wird
      bridge_fd 0 # kein Forwarding Delay

Die Bridge ffmuc-setup bekommt die IP-Adresse 192.168.1.2 um gluon zu konfigurieren. Es muss sichergestellt
sein, dass kein anderes Interface auf dem System diese IP-Adresse bereits nutzt, z.B. mit dem Befehl

  ip addr

Die Netzwerkkonfiguration muss per

  systemctl restart networking.service

oder mit einem Neustart des Systems neu geladen werden.

Die eingerichteten Bridges können angezeigt werden mit

  brctl show

was ergibt:

  bridge name     bridge id               STP enabled     interfaces
  ffmuc-clients           8000.3cf8627f3488       no              vnet0
  ffmuc-mesh              8000.fe5400b00e26       no              vnet2
  ffmuc-setup             8000.000000000000       no

== Installation und Konfiguration von gluon ==

Als nächstes kann das gluon-Image von https://firmware.ffmuc.net/ heruntergeladen werden:

  - Schritt 1: Wähle Dein Routermodell
    - Aus der Liste "manuelle Auswahl" des Routermodells wählen und "x86" aus der Liste wählen.
    - Aus der Liste "Bitte Modell wählen" den Eintrag "generic 64bit" selektieren.
  - Schritt 2: Wähle Deine Installationsart
    - "Erstinstallation" auswählen.
    - "stable" auswählen

Die heruntergeladene Datei in y.B. ~/Downloads/ entpacken und in /home/vm-images/ ablegen.

Dazu als Benutzer ausführen:

  gunzip ~/Downloads/gluon-ffmuc-v2019.1.3-x86-64.img.gz
  mkdir /home/vm-images/
  mv ~/Downloads/gluon-ffmuc-v2019.1.3-x86-64.img /home/vm-images/
  chgrp -R libvirt /home/vm-images
  chmod -R g+w /home/vm-images

Jetzt installieren wir das Image um es mit Qemu zu starten, dazu als Benutzer aufrufen:

  virt-install --connect qemu:///system --name ffmuc-offloader --memory 64 --vcpus=1 \
      --virt-type kvm --os-type linux --os-variant generic \
      --import --disk /home/vm-images/gluon-ffmuc-v2019.1.3-x86-64.img,format=raw,bus=sata \
      --network=bridge:ffmuc-setup \
      --network=bridge:ffmuc-clients \
      --network=bridge:ffmuc-mesh \
      --noautoconsole

Damit die VM beim Systemstart gestartet wird, muss dieses Kommando aufgerufen werden:

  virsh autostart <name>

Die Liste der verfügbaren VMs kann man sich anzeigen lassen mit

  virsh -c qemu:///system list

worauf diese Ausgabe erscheinen sollte:

   Id   Name              State
  ---------------------------------
   1    ffmuc-offloader   running

Jetzt kann gluon in der VM konfiguriert werden wie jeder andere gluon-Instanz, s. https://ffmuc.net/wiki/doku.php?id=knb:gui

Dazu einen Webbrowser starten und http://192.168.1.1 in der Adresszeile eingeben.

== Anpassen der Netzwerkschnittstellen ==

Nach dem Speichern der Konfiguration der gluon-Instanz wird sie heruntergefahren und das Netzwerk für die
finale Konfiguration umkonfiguriert.

  virsh -c qemu:///system shutdown ffmuc-offloader

Gluon soll via NAT über das lokale Netzwerk eine Verbindung in das Internet aufbauen können. Dazu richten wir
für KVM das Default-Netzwerk ein:

  virsh -c qemu:///system net-autostart default
  virsh -c qemu:///system net-start default

Jetzt werden die Netzwerkschnittstellen für gluon umkonfiguriert. Aus der Setup-Konfiguration

  - ffmuc-setup
  - ffmuc-clients
  - ffmuc-mesh

wird

  - ffmuc-clients
  - default
  - ffmuc-mesh

Als root editiert man dazu /etc/libvirt/qemu/ffmuc-offloader.xml und sucht die drei <interfaces>-Einträge.

Dann ersetzt man die alte Konfiguration

    <interface type='bridge'>
      <mac address='52:54:00:da:6f:0b'/>
      <source bridge='ffmuc-setup'/>
      <model type='e1000'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
    <interface type='bridge'>
      <mac address='52:54:00:3c:f0:d5'/>
      <source bridge='ffmuc-clients'/>
      <model type='e1000'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
    </interface>
    <interface type='bridge'>
      <mac address='52:54:00:b0:0e:26'/>
      <source bridge='ffmuc-mesh'/>
      <model type='e1000'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
    </interface>

durch die neue Konfiguration

    <interface type='bridge'>
      <mac address='52:54:00:3c:f0:d5'/>
      <source bridge='ffmuc-clients'/>
      <model type='e1000'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
    <interface type='network'>
      <mac address='52:54:00:da:6f:0b'/>
      <source network='default'/>
      <model type='e1000'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
    </interface>
    <interface type='bridge'>
      <mac address='52:54:00:b0:0e:26'/>
      <source bridge='ffmuc-mesh'/>
      <model type='e1000'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
    </interface>

Die <mac>...</mac>-Einträge sehen bei jeder Installation anders aus. Wichtig ist das Anpassen
der

  slot='0x03'

bzw.

  slot='0x04'

Einträge für ffmuc-clients und das default-Netzwerk.

Nach dem Speichern muss die Konfiguration neu eingelesen werden mit

  systemctl reload libvirtd.service

Nach dem Starten der gluon-VM testen wir die Internetverbindung der Instanz. Dazu aktivieren wir eine Konsole
mit

  virsh -c qemu:///system console ffmuc-offloader

Damit hat man Zugang zum gluon-Terminal. Nach dem Drücken von Return kann man die Internetverbindung z.B. mit

  ping ffmuc.net

testen. Als erfolgreiche Ausgabe erscheint z.B.

  root@ffmuc-5254008b1e37:/# ping ffmuc.net
  PING ffmuc.net (2606:4700:10::6816:4941): 56 data bytes
  64 bytes from 2606:4700:10::6816:4941: seq=0 ttl=59 time=21.524 ms
  64 bytes from 2606:4700:10::6816:4941: seq=1 ttl=59 time=10.657 ms

== Nutzen des internen WLAN-Adapters als Access Point ==

Um den internen WLAN-Adapter als Access Point für das Client-Netzwerk zu nutzen, installieren wir hostapd mit

  aptitude install hostapd

Dann wird /etc/hostapd/hostapd.conf editiert und sollte diesen Inhalt bekommen:

  bridge=ffmuc-clients
  interface=wlp1s0

  ssid=muenchen.freifunk.net/muc_cty
  channel=11
  hw_mode=g
  ieee80211n=1
  ieee80211d=1
  country_code=DE
  wmm_enabled=1

Die SSID für den aktuellen Standort kann der Liste auf https://ffmuc.net/wiki/doku.php?id=knb:gui entnommen werden.

Um den hostapd-Daemon im System zu aktivieren, muss

  systemctl unmask hostapd
  systemctl start hostapd.service

aufgerufen werden. Die Einrichtung von hostapd via /etc/default/hostap ist veraltet und wird in einer künftigen Debian-Version entfernt.

Das WLAN-Interface sollte jetzt an ffmuc-client angekoppelt sein, was man mit

  brctl show

prüfen kann. Die Ausgabe ist z.B.

    bridge name     bridge id               STP enabled     interfaces
    ffmuc-clients           8000.3cf8627f3488       no              vnet0
                                                            wlp1s0
    ffmuc-mesh              8000.fe5400b00e26       no              vnet2
    ffmuc-setup             8000.000000000000       no
    virbr0          8000.525400a88af3       yes             virbr0-nic
                                                            vnet1


Als Ausgangspunkt wurden folgende Beschreibungen genutzt:

  - https://psi.cx/2017/ff-router-als-vm/
  - https://psi.cx/2017/unifi-ff-setup/
  - www.freifunk-dueren.de/lte-offloader-router-in-einem-geraet-mit-webserver-fuer-dienste-im-freifunk-netz/
