# Deployment Overview and Selection Guide

> Last Updated: 2026-02-27

If you have not decided which deployment method to use, read this page first, then jump to the detailed guide.

## 1. Fast Start for Beginners (Recommended)

You can directly run the community one-click deployment script (supports Docker Compose / binary mode, plus update check and HTTPS ACME setup):

```bash
curl -fsSL https://raw.githubusercontent.com/dujiao-next/community-projects/main/scripts/dujiao-next-one-click-deploy/deploy.sh | bash
```

Project details:  
`https://github.com/dujiao-next/community-projects/tree/main/scripts/dujiao-next-one-click-deploy`

## 2. How to Choose a Deployment Method

| Method | Difficulty | Best For | Key Characteristics | Guide |
| --- | --- | --- | --- | --- |
| One-Click Script (Community) | Low | First-time deployment, quick setup users | Menu-driven flow, supports Docker/binary, includes update check and HTTPS | Community script README |
| Docker Compose | Medium | Users who need standardized and repeatable deployment | Container isolation, clear upgrade/rollback path, automation-friendly | [Docker Compose Deployment](/en/deploy/docker-compose) |
| aaPanel Manual Deployment | Low-Medium | Users already running aaPanel | GUI-oriented operations, suitable for panel-based maintenance | [aaPanel Deployment](/en/deploy/aapanel) |
| Manual Deployment (Build from source) | High | Advanced customization and secondary development | Highest control and flexibility | [Manual Deployment](/en/deploy/manual) |

## 3. Pre-Deployment Checklist

- Prepare a Linux server and domain(s) that resolve to your public IP (same-origin or split domains both work)
- Plan your ports (at minimum API port + web ports)
- Set strong random keys in `config.yml`:
  - `jwt.secret`
  - `user_jwt.secret`
- Choose your data stack:
  - Lightweight: SQLite + Redis
  - Recommended for production: PostgreSQL + Redis
- Choose one default admin initialization method:
  - Environment variables: `DJ_DEFAULT_ADMIN_USERNAME` / `DJ_DEFAULT_ADMIN_PASSWORD`
  - `config.yml`: `bootstrap.default_admin_username` / `bootstrap.default_admin_password`

## 4. Recommended Paths

- New user: start with the one-click script; if you already use aaPanel, go directly to the aaPanel guide.
- Long-term operations with stable repeatability: use Docker Compose.
- Deep customization or local build workflow: use manual deployment.

## 5. After Deployment

1. Verify service status:
   - API health endpoint: `/health`
   - User/Admin pages accessible
2. Change the admin password immediately after first login.
3. Configure payment settings and callback URLs (see [Payment Configuration & Callback Guide](/en/payment/guide)).
4. Enable HTTPS (the one-click script can handle this with ACME).

