# NGINX + Let's Encrypt für mehrere Subdomains

Diese Anleitung beschreibt Schritt für Schritt, wie man neue Dienste/Subdomains
hinter NGINX absichert.  
Besonderheit: Das Zertifikat wird **zuerst** per Webroot ausgestellt,
danach der HTTPS-vHost aktiviert.  
Dadurch vermeidet man den typischen Fehler  
`cannot load certificate .../fullchain.pem`.

---

## Voraussetzungen
- Server mit Debian/Ubuntu + NGINX
- Ports 80 und 443 offen
- DNS-A-Record zeigt auf deine öffentliche IP
- `sudo` oder root vorhanden
- Certbot installiert (`sudo apt install certbot python3-certbot-nginx`)

---

## 0) Variablen anpassen
```bash
DOMAIN="DOMAIN.EU"   # neue Subdomain
UPSTREAM_IP="192.168.xxx.xx"                  # interne Ziel-IP
UPSTREAM_PORT="80"                            # interner Ziel-Port
WEBROOT="/var/www/letsencrypt"                # Challenge-Verzeichnis
EMAIL="admin@$(hostname -f)"                  # Kontakt-Mail für Certbot

## 1) ACME-Webroot vorbereiten (einmalig)

sudo mkdir -p "$WEBROOT/.well-known/acme-challenge"
sudo chown -R www-data:www-data "$WEBROOT"

sudo tee /etc/nginx/snippets/letsencrypt.conf >/dev/null <<'EOF'
location ^~ /.well-known/acme-challenge/ {
    root /var/www/letsencrypt;
    default_type "text/plain";
    allow all;
    try_files $uri =404;
}
EOF
