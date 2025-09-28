# NGINX + Let's Encrypt (Multi-Domain Reverse Proxy)

Diese Anleitung beschreibt Schritt für Schritt, wie man für neue Dienste
(eigene Subdomains → interne IPs) per **NGINX Reverse Proxy + Let's Encrypt**
eine abgesicherte HTTPS-Verbindung einrichtet.  
Jede neue Seite (z. B. `DOMAIN.EU`) bekommt ein **eigenes Zertifikat**.

---

## Voraussetzungen

- Linux Server (Debian/Ubuntu)
- Root/Sudo Zugriff
- Port 80 + 443 offen
- DNS A-Record zeigt auf Server-IP
- `nginx`, `certbot` und `python3-certbot-nginx` installiert

```bash
sudo apt update
sudo apt install nginx certbot python3-certbot-nginx -y

1. Basis: Let's Encrypt Webroot vorbereiten
sudo mkdir -p /var/www/letsencrypt/.well-known/acme-challenge
sudo chown -R www-data:www-data /var/www/letsencrypt

Snippet erstellen:
bash
sudo nano /etc/nginx/snippets/letsencrypt.conf


Inhalt:
bash
location ^~ /.well-known/acme-challenge/ {
    root /var/www/letsencrypt;
    allow all;
}
