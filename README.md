# Nextcloud-auf-Ubuntu-Server24.02.2
#Festplatte

1. Festplatten anzeigen lassen (mmcblk0 = SD-Card, sda, sdb, sdc = USB-Festplatten)
   lsblk

2. Festplatte sda bearbeiten
   sudo fdisk /dev/sda
   p - anzeigen von bestehenden Partitionen
   d - Partition löschen (Vorsicht!!!)
   n - neue Partition anlegen
   p nach Eingabe von n bedeutet primary Partition anlegen
   w - Änderungen speichern

3. Partition sda1 mit Filesystem ext4 formatieren
   sudo mkfs -t ext4 /dev/sda1

4. UUID der Partition ermitteln
   sudo blkid /dev/sda1
   Beispielausgabe:
   /dev/sda1: UUID=123e4567-e89b-12d3-a456-426614174000 TYPE=ext4

5. Einhängepunkt unter /media erstellen
   sudo mkdir /media/nc

6. Partition sda1 einhängen
   sudo mount UUID=123e4567-e89b-12d3-a456-426614174000 /media/nc

7. Automatisches Einbinden konfigurieren
   sudo nano /etc/fstab
   Folgende Zeile unten anhängen:
   UUID=123e4567-e89b-12d3-a456-426614174000 /media/nc ext4 defaults 0 0
   nano mit strg+x verlassen und speichern

8. Änderungen testen
   sudo mount -a
   Überprüfen, ob die Partition gemountet wurde:
   df -h
9. sudo mkdir /media/nc/data

##Nextcloud Install

1. sudo nano nc-auto.sh
Einfügen:

#!/bin/bash

set -euo pipefail

log() {
    echo -e "\e[32m[INFO]\e[0m $1"
}

error() {
    echo -e "\e[31m[FEHLER]\e[0m $1" >&2
    exit 1
}

# Benutzereingaben
read -p "Nextcloud Admin-Benutzername: " ADMIN_USER
read -s -p "Nextcloud Admin-Passwort: " ADMIN_PASSWORD; echo
read -p "MariaDB Benutzername: " DB_USER
read -s -p "MariaDB Passwort: " DB_PASSWORD; echo
read -p "Pfad für Nextcloud-Daten (z. B. /mnt/nextcloud_data): " DATA_DIR

# Version auswählen
echo "Welche Nextcloud-Version soll installiert werden?"
echo "Einfach Enter drücken für die NEUESTE Version."
echo "Oder gib z. B. '27.1.4' ein für eine bestimmte Version:"
read -p "Version: " NC_VERSION

if [[ -z "$NC_VERSION" ]]; then
    NC_URL="http://download.nextcloud.com/server/releases/latest.tar.bz2"
    log "Neueste Version wird installiert."
else
    NC_URL="http://download.nextcloud.com/server/releases/nextcloud-${NC_VERSION}.tar.bz2"
    log "Version $NC_VERSION wird installiert."
fi

# Konstanten
NEXTCLOUD_DIR="/var/www/nextcloud"
DB_NAME="nextcloud"

log "System wird aktualisiert..."
apt update && apt upgrade -y

log "Benötigte Pakete werden installiert..."
apt install -y apache2 mariadb-server php php-mysql libapache2-mod-php \
php-gd php-json php-curl php-zip php-xml php-mbstring php-intl php-bcmath \
php-gmp php-imagick unzip wget ufw bzip2 php-cli

log "Apache-Module aktivieren..."
a2enmod rewrite headers env dir mime
systemctl restart apache2

log "Firewall freigeben..."
ufw allow OpenSSH
ufw allow 'Apache Full'
ufw --force enable

log "MariaDB absichern..."
mysql_secure_installation <<EOF
n
y
y
y
y
EOF

log "MariaDB-Datenbank und Benutzer erstellen..."
mysql -e "CREATE DATABASE IF NOT EXISTS $DB_NAME;"
mysql -e "CREATE USER IF NOT EXISTS '$DB_USER'@'localhost' IDENTIFIED BY '$DB_PASSWORD';"
mysql -e "GRANT ALL PRIVILEGES ON $DB_NAME.* TO '$DB_USER'@'localhost';"
mysql -e "FLUSH PRIVILEGES;"

log "Nextcloud herunterladen von: $NC_URL"
wget -O nextcloud.tar.bz2 "$NC_URL"

if ! file nextcloud.tar.bz2 | grep -q "bzip2 compressed data"; then
    error "Die Datei ist ungültig oder kein bzip2-Archiv."
fi

log "Nextcloud entpacken..."
tar -xjf nextcloud.tar.bz2
rm nextcloud.tar.bz2
rm -rf "$NEXTCLOUD_DIR"
mv nextcloud "$NEXTCLOUD_DIR"
chown -R www-data:www-data "$NEXTCLOUD_DIR"
chmod -R 755 "$NEXTCLOUD_DIR"

log "Datenverzeichnis erstellen..."
mkdir -p "$DATA_DIR"
chown -R www-data:www-data "$DATA_DIR"
chmod -R 750 "$DATA_DIR"

log "Apache Hauptkonfiguration anpassen (Root zeigt auf /var/www/nextcloud)..."
cat > /etc/apache2/sites-available/000-default.conf <<EOF
<VirtualHost *:80>
    DocumentRoot $NEXTCLOUD_DIR
    <Directory $NEXTCLOUD_DIR>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
EOF

systemctl reload apache2

log "Nextcloud installieren..."
sudo -u www-data php $NEXTCLOUD_DIR/occ maintenance:install \
  --database "mysql" --database-name "$DB_NAME" \
  --database-user "$DB_USER" --database-pass "$DB_PASSWORD" \
  --admin-user "$ADMIN_USER" --admin-pass "$ADMIN_PASSWORD" \
  --data-dir "$DATA_DIR"

log "Trusted Domain setzen..."
SERVER_IP=$(hostname -I | awk '{print $1}')
sudo -u www-data php $NEXTCLOUD_DIR/occ config:system:set trusted_domains 0 --value="$SERVER_IP"

log "Rechte setzen..."
chown -R www-data:www-data "$NEXTCLOUD_DIR"
chmod -R 755 "$NEXTCLOUD_DIR"

log "Nextcloud ist jetzt einsatzbereit unter: http://$SERVER_IP/"

echo -e "\e[34m[INFO]\e[0m Um Nextcloud  zu aktualisieren, führe folgenden Befehl aus:"
echo -e "\e[36mcd /var/www/nextcloud && sudo -u www-data php updater/updater.phar\e[0m"


2. sudo chmod +x nc-auto.sh
3. sudo ./nc-auto.sh

Updaten:
1. cd /var/www/nextcloud && sudo -u www-data php updater/updater.phar
