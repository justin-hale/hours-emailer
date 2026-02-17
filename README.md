# Weekly Hours Tracker

A lightweight, static web app for submitting weekly work hours via email. It runs entirely in the browser with no backend server — just a single HTML file hosted on GitHub Pages, with [EmailJS](https://www.emailjs.com/) handling email delivery.

## How It Works

1. A user visits the GitHub Pages URL and enters the site password.
2. They fill in hours for Monday–Friday (defaults to 8 each) and pick a week date.
3. Clicking **Send** fires an email through EmailJS to the configured recipient(s).

There is no database, no server, and no build step beyond the GitHub Actions workflow that injects configuration and deploys.

## Technology

| Layer | Technology | Purpose |
|---|---|---|
| **Hosting** | [GitHub Pages](https://pages.github.com/) | Serves the static site |
| **CI/CD** | [GitHub Actions](https://docs.github.com/en/actions) | Injects config variables and deploys on every push to `main` |
| **Email** | [EmailJS](https://www.emailjs.com/) | Sends email directly from the browser (no backend) |
| **Password** | SHA-256 (Web Crypto API) | Client-side password gate; hash is computed at build time |
| **Frontend** | Vanilla HTML / CSS / JS | Single-file app, no frameworks or bundlers |

### Password Protection

The plaintext password is **never** stored in the deployed site. During deployment, GitHub Actions hashes the `SITE_PASSWORD` variable with SHA-256 and injects only the hash into the page. When a user enters the password, the browser hashes their input using the Web Crypto API and compares it to the stored hash. Authentication persists for the browser tab via `sessionStorage`.

> **Note:** This is a convenience gate, not high-security authentication. The hash is visible in the page source, and a determined user could bypass it. It is appropriate for limiting casual access, not protecting sensitive data.

## Project Structure

```
hours-emailer/
├── index.html                  # The entire application (HTML + CSS + JS)
└── .github/
    └── workflows/
        └── deploy.yml          # GitHub Actions workflow for deployment
```

## External Services

### EmailJS

EmailJS lets the browser send email without a backend. You need three things configured in your EmailJS account:

| EmailJS Item | What It Is | Where to Find It |
|---|---|---|
| **Service ID** | Links to your email provider (Gmail, Outlook, etc.) | [EmailJS Dashboard > Email Services](https://dashboard.emailjs.com/) |
| **Template ID** | Defines the email layout; uses `{{to_emails}}`, `{{subject}}`, and `{{body}}` template variables | [EmailJS Dashboard > Email Templates](https://dashboard.emailjs.com/) |
| **Public Key** | Authenticates browser requests to EmailJS | [EmailJS Dashboard > Account > API Keys](https://dashboard.emailjs.com/) |

Your EmailJS email template should include these template variables:

- `{{to_emails}}` — recipient email address(es)
- `{{subject}}` — email subject line (e.g. "Hours Worked 02-17-2026")
- `{{body}}` — the formatted hours breakdown

### GitHub Pages

GitHub Pages must be enabled for the repository. The workflow uses the newer `actions/deploy-pages` approach (not the legacy branch-based method), which requires:

- **Settings > Pages > Source** set to **GitHub Actions**
- The workflow permissions listed in `deploy.yml` (`contents: read`, `pages: write`, `id-token: write`)

## Configuration

All configuration lives in **GitHub repository variables** (Settings > Secrets and variables > Actions > Variables tab). No secrets are needed — these are all non-sensitive except the password, which is hashed at build time before it reaches the deployed page.

| Variable | Description | Example |
|---|---|---|
| `EMAILJS_SERVICE_ID` | Your EmailJS service ID | `service_abc123` |
| `EMAILJS_TEMPLATE_ID` | Your EmailJS template ID | `template_xyz789` |
| `EMAILJS_PUBLIC_KEY` | Your EmailJS public/API key | `aBcDeFgHiJkLmN` |
| `RECIPIENT_EMAILS` | Comma-separated list of recipient email addresses | `boss@company.com,hr@company.com` |
| `SITE_PASSWORD` | Plaintext password (hashed at deploy time, never stored raw) | `mypassword123` |

### Changing the Password

1. Go to **Settings > Secrets and variables > Actions > Variables**.
2. Edit `SITE_PASSWORD` to the new plaintext value.
3. Re-run the deploy workflow (push to `main` or trigger manually via **Actions > Deploy to GitHub Pages > Run workflow**).

The workflow will hash the new password and deploy the updated page.

### Changing Recipients

Update the `RECIPIENT_EMAILS` variable and re-deploy. Multiple addresses should be comma-separated with no spaces.

## Deployment

Deployment happens automatically on every push to `main`. You can also trigger it manually:

1. Go to **Actions > Deploy to GitHub Pages**.
2. Click **Run workflow** > **Run workflow**.

### What the Workflow Does

1. Checks out the repository.
2. Reads the five repository variables and injects them into `index.html` via `sed` replacements.
3. Hashes `SITE_PASSWORD` with `sha256sum` and injects the hash (not the plaintext).
4. Uploads the directory as a GitHub Pages artifact and deploys it.

## Maintenance Checklist

- **EmailJS free tier** — limited to 200 emails/month. Monitor usage at [dashboard.emailjs.com](https://dashboard.emailjs.com/). Upgrade the plan if volume increases.
- **EmailJS service connection** — if the linked email provider changes its password or revokes access, re-authenticate at EmailJS Dashboard > Email Services.
- **GitHub Pages** — verify the site is live at `https://<username>.github.io/hours-emailer/`. If deployment fails, check the Actions tab for errors.
- **Password rotation** — update the `SITE_PASSWORD` variable and re-deploy whenever you need a new password. Let all users know the new password out-of-band.
- **EmailJS library version** — the app loads `@emailjs/browser@4` from the jsDelivr CDN. Major version bumps may require updating the `<script>` tag in `index.html`.

## Local Development

Since this is a single HTML file, you can open `index.html` directly in a browser. The password gate and EmailJS calls won't work because the placeholder tokens (`__EMAILJS_SERVICE_ID__`, etc.) haven't been replaced. To test locally:

1. Temporarily replace the placeholder values in the `CONFIG` object with real values.
2. Open `index.html` in a browser.
3. **Do not commit** the real values back to the repository.
