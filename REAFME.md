# Streamlit admin dashboard

This is a separate, read-only Streamlit dashboard for the existing Woldia Smart Campus MySQL database. It does not replace or modify the PHP portal.

## Files

- `app.py` — dashboard and password gate
- `requirements.txt` — Python dependencies
- `Dockerfile` — container build
- `docker-compose.yml` — local or server deployment
- `.env.example` — environment variable template (copy to `.env`)
- `.streamlit/secrets.toml.example` — Streamlit Community Cloud secrets template
- `.dockerignore` and `.gitignore` — exclude local secrets and generated files

## Deploy with Streamlit Community Cloud

Streamlit Community Cloud deploys from a GitHub repository; it does not deploy this ZIP directly.

1. Extract the ZIP and put the contents of `streamlit_dashboard` in a GitHub repository (or in a folder in an existing repository).
2. Push the repository to GitHub. Keep `.streamlit/secrets.toml` and `.env` out of Git.
3. In Streamlit Community Cloud, choose **Create app**, select the repository and branch, and set the app file path to `app.py` (or `streamlit_dashboard/app.py` if you kept the folder in a larger repository).
4. Open the app's **Settings → Secrets**. Copy the contents of `.streamlit/secrets.toml.example`, replace every placeholder, and save it there. Do not commit the secrets file.
5. Deploy or reboot the app.

The database must be reachable from Streamlit Community Cloud. A MySQL server at `localhost` or a private XAMPP installation on your PC is not accessible to the hosted app. Use a securely hosted MySQL instance with network access restricted to trusted sources and a dedicated account with `SELECT` permission only on the four dashboard tables. Do not expose a development database or the XAMPP `root` account to the public internet.

## Deploy with Docker Compose

1. Install Docker Engine and the Docker Compose plugin on the deployment machine.
2. Copy `.env.example` to `.env`.
3. Edit `.env`:
   - Set `DASHBOARD_PASSWORD` to a long, unique password.
   - Set `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, and `DB_PASSWORD` for the MySQL instance.
   - For XAMPP running on the same Windows machine as Docker Desktop, `host.docker.internal` is commonly the database host. If MySQL is on another server, use that server's reachable hostname or IP.
4. From this directory, build and start the dashboard:

   ```sh
   docker compose up --build -d
   ```

5. Open `http://localhost:8501` (or the configured `STREAMLIT_PORT`). Check startup and health with:

   ```sh
   docker compose ps
   docker compose logs -f
   ```

Stop it with `docker compose down`.

## Database permissions and access

The dashboard only runs `SELECT COUNT(*)` queries on `users`, `complaints`, `announcements`, and `student_clearance`. Use a dedicated MySQL account with `SELECT` permission on those tables, rather than the XAMPP `root` account. Ensure the MySQL server accepts connections from Docker and that its firewall only permits trusted sources.

The password gate is separate from the PHP portal's login and role checks. It is a shared dashboard password, not per-user authentication. For public internet access, put the service behind HTTPS and an identity-aware reverse proxy or other organization-approved access control. Do not commit `.env` or expose port 8501 directly to the public internet.

The database schema and SQL dump are not included in this project folder; the existing MySQL database must be available to the container.