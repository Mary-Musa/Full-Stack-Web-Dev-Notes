# Week 25, Day 4: Nginx and TLS

By the end of today, a single Nginx container routes traffic to your shop, API, and admin dashboard with a real TLS certificate from Let's Encrypt. Customers visit `https://yourshop.co.ke` and everything just works.

**Prior concepts:** docker-compose with networks (Week 25 Day 3).

**Estimated time:** 3 hours

---

## Why Nginx

Nginx is a reverse proxy. It sits in front of your services, accepts incoming HTTP/HTTPS requests, and routes them to the right backend based on the URL path or hostname. Three reasons to use it:

1. **TLS termination.** Nginx handles the `https://` layer; your Node apps speak plain HTTP inside the Docker network. Simpler to configure Nginx for TLS once than every service individually.
2. **Path routing.** `/api/*` goes to the API, `/admin/*` goes to the admin service, `/` goes to the shop. One domain, many services.
3. **Security headers, rate limiting, caching, compression.** Nginx does all of these better than Express middleware would.

For a Kenyan SME shop on one server, Nginx is mandatory. Cloud platforms (Vercel, Netlify) bake it in; on your own server you run it explicitly.

---

## Adding Nginx To Compose

```yaml
# docker-compose.yml
services:
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/sites/:/etc/nginx/conf.d/:ro
      - ./nginx/certs:/etc/nginx/certs:ro
      - certbot_data:/var/www/certbot
    depends_on:
      - api
      - shop
    networks:
      - frontend

  # ... other services with no exposed ports
```

In production mode, only Nginx exposes ports 80 and 443. All other services are internal.

---

## A Minimal Nginx Config

```nginx
# nginx/nginx.conf
user nginx;
worker_processes auto;

events {
  worker_connections 1024;
}

http {
  include /etc/nginx/mime.types;
  default_type application/octet-stream;

  sendfile on;
  tcp_nopush on;
  keepalive_timeout 65;
  gzip on;
  gzip_types text/plain text/css application/json application/javascript;

  upstream shop_upstream {
    server shop:3000;
  }

  upstream api_upstream {
    server api:5000;
  }

  include /etc/nginx/conf.d/*.conf;
}
```

```nginx
# nginx/sites/mctaba.conf
server {
  listen 80;
  server_name mctaba.co.ke www.mctaba.co.ke;

  # ACME challenge for Let's Encrypt
  location /.well-known/acme-challenge/ {
    root /var/www/certbot;
  }

  # Redirect all other traffic to HTTPS
  location / {
    return 301 https://$host$request_uri;
  }
}

server {
  listen 443 ssl http2;
  server_name mctaba.co.ke www.mctaba.co.ke;

  ssl_certificate /etc/nginx/certs/live/mctaba.co.ke/fullchain.pem;
  ssl_certificate_key /etc/nginx/certs/live/mctaba.co.ke/privkey.pem;

  # Modern TLS config
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers HIGH:!aNULL:!MD5;
  ssl_prefer_server_ciphers on;

  # Security headers
  add_header Strict-Transport-Security "max-age=31536000" always;
  add_header X-Frame-Options "SAMEORIGIN" always;
  add_header X-Content-Type-Options "nosniff" always;

  # API routes
  location /api/ {
    proxy_pass http://api_upstream;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Request-Id $request_id;
  }

  # Webhook routes (no rewriting)
  location /webhook/ {
    proxy_pass http://api_upstream;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }

  # Everything else goes to the shop
  location / {
    proxy_pass http://shop_upstream;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }

  # Larger body for file uploads
  client_max_body_size 10M;
}
```

Walk through:

- HTTP (port 80) redirects everything to HTTPS except the Let's Encrypt challenge path.
- HTTPS (port 443) serves with TLS using certs from the mounted volume.
- `/api/*` routes to the API container.
- `/webhook/*` routes to the API (M-Pesa callbacks, WhatsApp webhooks).
- Everything else routes to the Next.js shop.
- Headers like `X-Forwarded-For` tell the backend the real client IP.

---

## Getting A Real Cert With Certbot

Let's Encrypt gives you free 90-day TLS certs. Certbot is the client that requests them.

```yaml
# Add to docker-compose.yml
services:
  certbot:
    image: certbot/certbot
    volumes:
      - ./nginx/certs:/etc/letsencrypt
      - certbot_data:/var/www/certbot
    command: sleep infinity   # keeps the container alive for manual runs

volumes:
  certbot_data:
```

Initial cert issuance:

```bash
docker-compose run --rm certbot certonly \
  --webroot -w /var/www/certbot \
  --email you@example.com \
  --agree-tos \
  --no-eff-email \
  -d mctaba.co.ke -d www.mctaba.co.ke
```

Certbot places files in a webroot path, Let's Encrypt fetches them to verify you control the domain, and the resulting certs land in `./nginx/certs/live/mctaba.co.ke/`.

Reload Nginx to pick up the new certs:

```bash
docker-compose exec nginx nginx -s reload
```

Visit `https://mctaba.co.ke`. Real HTTPS. Browser shows a green padlock.

### Renewal

Let's Encrypt certs expire after 90 days. Renew every 60 days via a cron job:

```bash
# Add to your OS cron or a scheduled Compose run
docker-compose run --rm certbot renew
docker-compose exec nginx nginx -s reload
```

Automate this or it will bite you at month 3.

---

## Rate Limiting In Nginx

```nginx
# In the http block
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/s;
limit_req_zone $binary_remote_addr zone=login_limit:10m rate=5r/m;

# In the server block
location /api/auth/login {
  limit_req zone=login_limit burst=3 nodelay;
  proxy_pass http://api_upstream;
}

location /api/ {
  limit_req zone=api_limit burst=20;
  proxy_pass http://api_upstream;
}
```

- `api_limit` allows 100 requests/second per IP; bursts up to 20 are absorbed into the bucket.
- `login_limit` is stricter: 5 requests/minute, max 3 burst. This protects the login endpoint from brute-force attacks.

Rate limiting in Nginx is cheaper and more reliable than in Express middleware because it happens before the request reaches your Node app.

---

## Caching Static Assets

Next.js static files can be cached aggressively:

```nginx
location /_next/static/ {
  proxy_pass http://shop_upstream;
  proxy_cache_valid 200 1y;
  add_header Cache-Control "public, max-age=31536000, immutable";
}

location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|webp)$ {
  proxy_pass http://shop_upstream;
  proxy_cache_valid 200 7d;
  add_header Cache-Control "public, max-age=604800";
}
```

Static assets from Next.js are fingerprinted (their URLs include content hashes), so they can cache forever. Images and other assets cache for a week.

---

## Checkpoint

1. `docker-compose up` starts Nginx on port 80 and 443.
2. Visiting `http://localhost` redirects to HTTPS (self-signed in dev).
3. `/api/health` is proxied to the API container.
4. `/` is proxied to the Next.js shop.
5. Security headers appear in the response (`curl -I https://localhost -k`).
6. Rate limit test: 200 rapid requests to `/api/auth/login` hit the rate limit.
7. On a real server, certbot issues a real Let's Encrypt cert and browsers trust it.

Commit:

```bash
git add .
git commit -m "feat: nginx reverse proxy with tls and security headers"
```

---

## What You Learned

- Nginx terminates TLS so Node apps stay plain HTTP inside.
- Path routing sends different URLs to different services.
- Let's Encrypt via certbot gives you free TLS.
- Rate limiting at the edge is faster than in Express.
- Static asset caching cuts repeat traffic to your app.

Tomorrow is the recap, and the weekend deploys the whole thing to a real server.
