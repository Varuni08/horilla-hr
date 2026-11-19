# Horilla on Vercel — fictional-data POC only

This fork contains **experimental** Vercel container configuration. It is meant
for manual exploration of Horilla with **fictional employees and fake payroll**,
not for real employee records, production payroll, or a public HR service.

## Architecture and intentional omissions

- Vercel Container Function: Horilla's Django + Gunicorn web app.
- External managed PostgreSQL: required for consistent logins and saved data.
- Static CSS/JS: collected into the container image at build time.
- Redis: omitted initially; the application uses local-memory caching without it.
- Scheduler: **not running**. Payslip automation, leave resets, reminders,
  expiring contracts, and other scheduled functions do not run.
- Uploads/media: **not configured**. The Vercel function filesystem is read-only;
  do not test file uploads until private object storage is configured and tested.
- Email/external integrations: not configured. Do not use real recipient addresses.

The existing Compose setup is unchanged and is still the documented deployment
path for the full application. Do not use these POC files as a production recipe.

## 1. Create a managed PostgreSQL database

Create a **new, empty PostgreSQL database exclusively for this demo**, for
example using a supported managed provider such as Neon. Use an SSL-enabled
connection string and keep it private; never add credentials to the repository.

## 2. Run the schema migrations once with GitHub Actions

1. In this GitHub repository, open **Settings → Secrets and variables → Actions**.
2. Add a *repository secret* called `HORILLA_DEMO_DATABASE_URL` whose value is
   the PostgreSQL connection string from step 1.
3. Open **Actions → Migrate Horilla demo database → Run workflow** on branch
   `2.0`. The manual workflow builds the same container and executes only
   `python manage.py migrate --noinput` against that database.
4. Wait for the workflow to finish successfully before using the live app.

The workflow has no timer and runs only when you explicitly trigger it.
It does not create employees, administrators, or fake payroll automatically.

## 3. Import the repository into Vercel

1. Open https://vercel.com/new and sign in to the Vercel account you want.
2. Import your fork `Varuni08/horilla-hr`, branch `2.0`, project root `.`.
3. Make sure Vercel selects its **Container** runtime based on `vercel.json`
   and `Dockerfile.vercel`. Leave the build settings at their defaults.
4. Set the following **Production** environment variables *before* deployment:

   | Name | Value |
   | --- | --- |
   | `DATABASE_URL` | Same SSL PostgreSQL connection string as step 1 |
   | `SECRET_KEY` | New, long, random secret unique to this demo |
   | `DB_INIT_PASSWORD` | Different, strong random password for first setup |
   | `DEBUG` | `False` |
   | `HORILLA_ENV` | `production` |
   | `ALLOWED_HOSTS` | **Only** the actual Vercel project hostname, no `https://` |
   | `CSRF_TRUSTED_ORIGINS` | `https://` followed by that same exact hostname |
   | `TIME_ZONE` | `Asia/Kolkata` |
   | `HORILLA_REQUIRE_SCHEDULER` | `0` |

   To generate two independent random secrets locally, run this command twice:

   ```bash
   python -c "import secrets; print(secrets.token_urlsafe(64))"
   ```

   **Never use placeholder keys or `ALLOWED_HOSTS=*`.** Horilla deliberately
   rejects insecure production settings. Choose a project hostname before
   setting `ALLOWED_HOSTS` and `CSRF_TRUSTED_ORIGINS`. If your final hostname
   changes, update both environment variables and redeploy.

5. Deploy. On Vercel's Hobby plan, the app is for personal/noncommercial
   experiments only; a company or commercial POC may require a paid plan.

## 4. Initialize and verify

1. Visit `https://YOUR-HOSTNAME/health/`, then the home page.
2. On a new database Horilla shows **Initialize Database**. Enter the
   `DB_INIT_PASSWORD` from Vercel and create the first admin account.
3. Sign in, create a fictional company and fictional employee profile.
4. Check pages for Employee, Attendance, Leave, and Timesheets/Projects.
5. Do **not** enter real people, salaries, IDs, addresses, or documents.

## Limitations and troubleshooting

- If a build fails, read the Vercel **Build Logs**. This Dockerfile relies on
  the `horilla/horilla-hr:2.4.0` base image, which must remain compatible with
  the checked-out source release. Update both together after upstream changes.
- `DisallowedHost` means `ALLOWED_HOSTS` does not match the request hostname.
- CSRF failures often mean `CSRF_TRUSTED_ORIGINS` lacks the deployed HTTPS URL.
- Missing tables usually mean the manual migration workflow did not finish.
- File uploads and scheduled jobs are *deliberately unsupported* in this POC.
- `/ready/` may report the missing scheduler. That is expected in this POC,
  but must not be accepted in a full production launch.
- Vercel may scale containers and restart them; never use local SQLite for this
  deployment. The external database is essential for persistent demo state.

**Status:** These deployment files are prepared but no Vercel build or live
deployment has been verified yet. Review Vercel logs and test the routes before
calling the application deployed.
