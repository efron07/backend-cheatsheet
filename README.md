# Backend Cheat Sheet — Node.js, Express, PostgreSQL & MongoDB (PharmaCare Edition)

Backend fundamentals, Node.js, Express, API design, PostgreSQL on Neon (SQL + DBA),
MongoDB with Mongoose, auth, caching, jobs, security, testing, observability, architecture
and deployment — built around the PharmaCare pharmacy API that serves the Next.js frontend.

**Live site:** https://efron07.github.io/backend-cheatsheet/

## Edit
Pages live in `docs/`, one per section. Push to `main` and GitHub Actions republishes.

## Preview locally
```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve        # http://127.0.0.1:8000
```

MkDocs is pinned to 1.6.x on purpose: MkDocs 2.0 breaks Material-based sites.
