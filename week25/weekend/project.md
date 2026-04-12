# Week 25 Weekend: Deploy To A Real Server

Take everything and put it on a real Linux VPS. A DigitalOcean droplet, a Linode, a Contabo VPS -- any Ubuntu 22.04 server with 2GB RAM and a public IP.

**Estimated time:** 4-6 hours.

**Deadline:** Monday morning.

---

## What You Need

- A VPS with Ubuntu 22.04.
- A domain name (yours or a free subdomain from freedns.afraid.org).
- SSH access.
- Docker and docker-compose installed on the server.

---

## Steps

1. Provision the VPS.
2. Point your domain's A record to the VPS IP.
3. SSH in; install Docker (`apt install docker.io docker-compose`).
4. Clone your repo.
5. Create a production `.env` with real keys.
6. `docker-compose -f docker-compose.yml up -d`.
7. Initial certbot run for the domain.
8. Reload Nginx.
9. Verify `https://yourdomain.co.ke` loads the shop.
10. Run a real test purchase end to end using a real phone.

---

## Deliverables

- Live URL.
- A `DEPLOY.md` file describing every step, in order, with commands.
- Screenshot of the shop live.
- Screenshot of a successful sandbox M-Pesa test payment.
- `uptime` command output showing the server has been up for at least 12 hours.

---

## Grading Rubric (50 pts)

| Area | Points |
|---|---|
| Live site at a real URL | 20 |
| Working HTTPS with Let's Encrypt | 10 |
| End-to-end purchase works | 10 |
| DEPLOY.md quality | 5 |
| No regressions | 5 |

---

## Hints

**Smallest viable server.** 2 GB RAM is enough for dev-scale traffic. Do not pay for more before you need it.

**UFW for firewall.** After setup, close every port except 22 (SSH), 80 (HTTP), 443 (HTTPS).

```bash
sudo ufw allow 22
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable
```

**Secrets management.** For the Marathon, a gitignored `.env` is fine. For real production, use a secrets manager. Make a note in DEPLOY.md that you will upgrade later.

**Monitoring.** Install `htop` and check CPU/memory while running the site. If you are near limits, tune resource limits in compose.

**Cost.** A Contabo or Hetzner VPS runs around $5/month and is fine for this course. DigitalOcean is $6 for the cheapest droplet. If you cannot spend money, use Oracle Cloud's always-free tier.

Next week is CI/CD -- automating what you did by hand today.
