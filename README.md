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
```

## 1) ACME-Webroot vorbereiten (einmalig)
```bash
sudo mkdir -p "$WEBROOT/.well-known/acme-challenge"
```
```bash
sudo chown -R www-data:www-data "$WEBROOT"
```
```bash
sudo tee /etc/nginx/snippets/letsencrypt.conf >/dev/null <<'EOF'
location ^~ /.well-known/acme-challenge/ {
    root /var/www/letsencrypt;
    default_type "text/plain";
    allow all;
    try_files $uri =404;
}
EOF
```
## 2) HTTP-only vHost erstellen

Wichtig: Noch kein SSL eintragen!
```bash
sudo tee "/etc/nginx/sites-available/$DOMAIN" >/dev/null <<EOF
server {
    listen 80;
    listen [::]:80;
    server_name $DOMAIN;

    include /etc/nginx/snippets/letsencrypt.conf;

    location = / {
        return 200 'http vhost ok for $DOMAIN';
        add_header Content-Type text/plain;
    }
}
EOF
```
```bash
sudo ln -sf "/etc/nginx/sites-available/$DOMAIN" "/etc/nginx/sites-enabled/$DOMAIN"
```
```bash
sudo nginx -t && sudo systemctl reload nginx
```

Test:
```bash
echo TEST123 | sudo tee "$WEBROOT/.well-known/acme-challenge/probe"
```
```bash
curl -i "http://$DOMAIN/.well-known/acme-challenge/probe"
```
## 3) Zertifikat ausstellen
```bash
sudo certbot certonly --agree-tos -n --email "$EMAIL" \
  --webroot -w "$WEBROOT" -d "$DOMAIN"
```

Prüfen:
```bash
sudo ls -l "/etc/letsencrypt/live/$DOMAIN/"
```
## 4) HTTPS-vHost aktivieren

Jetzt SSL eintragen und Reverse Proxy auf das Backend bauen.
```bash
sudo tee "/etc/nginx/sites-available/$DOMAIN" >/dev/null <<EOF
server {
    listen 80;
    listen [::]:80;
    server_name $DOMAIN;

    include /etc/nginx/snippets/letsencrypt.conf;
    return 301 https://\$host\$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name $DOMAIN;

    ssl_certificate     /etc/letsencrypt/live/$DOMAIN/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/$DOMAIN/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # ACME auch über 443 erlauben
    location ^~ /.well-known/acme-challenge/ {
        auth_basic off;
        root /var/www/letsencrypt;
        default_type "text/plain";
        try_files \$uri =404;
    }

    # Reverse Proxy
    location / {
        proxy_pass http://$UPSTREAM_IP:$UPSTREAM_PORT/;
        include proxy_params;
        proxy_redirect off;

        # interne Redirects anpassen
        proxy_redirect http://$UPSTREAM_IP:$UPSTREAM_PORT/ https://$DOMAIN/;
    }
}
EOF
```
```bash
sudo nginx -t && sudo systemctl reload nginx
```
## 5) Schnelltests
```bash
curl -I "http://$DOMAIN"          # → 301 auf https
```
```bash
curl -I "https://$DOMAIN" -k      # → 200 oder 302
```
## 6) (Optional) Security-Header und Auth

Nach erfolgreichem Test kannst du Security-Header und ggf. Basic-Auth
aktivieren. Beispiel:

```bash
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Content-Security-Policy "default-src 'self';" always;

    auth_basic "Zugang nur für Berechtigte";
    auth_basic_user_file /etc/nginx/.htpasswd;
```

User anlegen:
```bash
sudo htpasswd /etc/nginx/.htpasswd BENUTZER
```
## 7) Renewal testen
```bash
sudo certbot renew --dry-run
```
## 8) Neue Subdomain hinzufügen

Variablen (DOMAIN, UPSTREAM_IP, UPSTREAM_PORT) anpassen

Schritte 1–4 wiederholen

Fertig 🎉

## Troubleshooting

cannot load certificate
→ HTTPS-Block zu früh angelegt. Erst certonly machen, dann SSL eintragen.

Challenge gibt 401/403 zurück
→ Im HTTP-vHost darf kein auth_basic vor dem letsencrypt.conf-Include aktiv sein.

502 Bad Gateway
→ Backend-IP/Port prüfen, ggf. Firewall/Service down.

Doppelte Login-Abfrage
→ Basic-Auth nur nutzen, wenn die App keine eigene Auth hat.
