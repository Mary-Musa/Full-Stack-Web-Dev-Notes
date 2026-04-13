# Week 25 - Day 4 Assignment

## Title
Nginx Reverse Proxy And TLS

## Overview
Day 4 puts an Nginx in front of your API. Nginx handles TLS termination, compression, static assets, and forwards everything else to the Node backend. You add a self-signed cert for local dev and understand the real prod setup.

## Learning Objectives Assessed
- Configure Nginx as a reverse proxy
- Terminate TLS at Nginx
- Forward headers correctly (X-Forwarded-For, X-Forwarded-Proto)
- Understand where Let's Encrypt fits in production

## Prerequisites
- Days 1-3 completed

## AI Usage Rules

**Ratio this week:** 30% manual / 70% AI
**Habit:** AI writes infra, you verify boundaries. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Nginx config boilerplate.
- **NOT ALLOWED FOR:** The security headers (you decide your policy).
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Add Nginx to compose

**What to do:**
```yaml
nginx:
  image: nginx:1.27-alpine
  volumes:
    - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    - ./nginx/certs:/etc/nginx/certs:ro
  ports:
    - "80:80"
    - "443:443"
  depends_on: [api]
```

**Expected output:**
Nginx container starts.

### Task 2: Self-signed cert and config

**What to do:**
```bash
mkdir -p nginx/certs
openssl req -x509 -newkey rsa:2048 -nodes -keyout nginx/certs/key.pem \
  -out nginx/certs/cert.pem -days 365 -subj "/CN=localhost"
```

`nginx/nginx.conf`:

```nginx
events {}
http {
  upstream api { server api:3000; }
  server {
    listen 80;
    return 301 https://$host$request_uri;
  }
  server {
    listen 443 ssl;
    ssl_certificate /etc/nginx/certs/cert.pem;
    ssl_certificate_key /etc/nginx/certs/key.pem;

    location / {
      proxy_pass http://api;
      proxy_set_header Host $host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_set_header X-Real-IP $remote_addr;
    }
  }
}
```

**Expected output:**
`curl -k https://localhost` hits your API. HTTP redirects to HTTPS.

### Task 3: Trust the proxy in Express

**What to do:**
```javascript
app.set("trust proxy", 1);
```

So `req.ip` reflects the real client, not the Nginx container IP.

**Expected output:**
Logged IPs are correct.

### Task 4: Security headers

**What to do:**
Install `helmet`:

```bash
npm install helmet
```

```javascript
app.use(helmet());
```

In `headers-notes.md`, write 4-6 sentences:
- What does `Strict-Transport-Security` do?
- What does `X-Content-Type-Options: nosniff` prevent?
- Why is `Content-Security-Policy` worth the hassle?

**Expected output:**
`headers-notes.md` committed.

### Task 5: Production path

**What to do:**
In `prod-tls.md`, write 5-7 sentences:
- Self-signed in dev, Let's Encrypt in prod. Why?
- How does `certbot` renew certs?
- What is `acme-companion` in compose land?
- What happens at 89 days if you never renew?

Your own words.

**Expected output:**
`prod-tls.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Serve static files directly from Nginx.
- Enable gzip compression.
- Use HTTP/2.

## Submission Requirements

- **What to submit:** Repo with nginx config, notes, `AI_AUDIT.md`.
- **Deadline:** End of Day 4

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Nginx in compose | 15 | Starts cleanly. |
| TLS + HTTP->HTTPS | 25 | Redirect + cert work. |
| Trust proxy + headers forwarded | 20 | IPs correct. |
| Security headers with helmet | 15 | In place. |
| Headers notes | 10 | Three questions answered. |
| Prod TLS notes | 10 | Four questions answered. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Not trusting the proxy.** IP-based rate limiting then sees only the Nginx IP.
- **Forgetting the HTTP -> HTTPS redirect.** Clients land on plain HTTP forever.
- **Using the default Nginx config unchanged.** It bites in subtle ways.

## Resources

- Day 4 reading: [Nginx and TLS.md](./Nginx%20and%20TLS.md)
- Week 25 AI boundaries: [../ai.md](../ai.md)
- Nginx proxy docs: https://nginx.org/en/docs/http/ngx_http_proxy_module.html
