# Google Analytics → "Popular essays" on the homepage

The homepage has a **Most read / Popular essays** section driven by
`_data/popular.yml`. A scheduled GitHub Action
(`.github/workflows/update-popular.yml`, weekly + manual) regenerates
that file from the Google Analytics Data API and commits it; the commit
triggers a GitHub Pages rebuild. No analytics credentials ever reach
the browser — GA is read server-side, at build time.

## Why a service account, not a "GA token"

GA4 has no permanent read token:

- The **Measurement Protocol API secret** only *sends* events; it cannot
  read reports.
- An **OAuth access token** lasts ~1 hour; a refresh token is tied to a
  personal Google account and breaks on password/2-Step changes.

The durable route is a **Google service account** with read-only access
to the one GA property. Its JSON key lives only as a GitHub Actions
secret.

## One-time setup (≈5 minutes, in your own accounts)

1. **Google Cloud Console** (console.cloud.google.com, signed in as you)
   - Create (or pick) a project, e.g. `rammehta-blog-analytics`.
   - Enable **Google Analytics Data API** for that project.
   - IAM & Admin → Service Accounts → Create: name it `ga-reader`.
   - On the service account: Keys → Add key → JSON. **Save the downloaded
     JSON — treat it like a password. Do not paste it into chat or commit it.**
   - Copy the service account's email (`ga-reader@<project>.iam.gserviceaccount.com`).
2. **Google Analytics** (analytics.google.com) → Admin → Property →
   Property access management → add that service-account email as
   **Viewer**. (Property = the one whose measurement ID is `G-GBZBCHDDVD`.)
3. **Find the numeric Property ID**: GA → Admin → Property settings →
   "Property ID" (a number like `123456789`). This is *not* the
   `G-GBZBCHDDVD` measurement ID — the API needs the number.
4. **GitHub repo** `rammehta1899.github.io` → Settings → Secrets and
   variables → Actions:
   - Secret: `GA_SERVICE_ACCOUNT_KEY` = the full contents of the JSON key file.
   - Variable: `GA_PROPERTY_ID` = the numeric property ID from step 3.
5. Actions tab → "Update popular posts (Google Analytics)" → Run workflow.
   It writes the top post paths + view counts (last 90 days) to
   `_data/popular.yml` and commits; the homepage then shows Popular essays.

## Notes

- Revoking access = delete the service account key (or remove the
  Viewer grant in GA). Nothing else on the site depends on it.
- The Action is a no-op until `GA_PROPERTY_ID` is set, so merging the
  workflow early is safe.
- Window and count are env settings in the workflow (`GA_WINDOW_DAYS=90`,
  script default top 8; the homepage displays up to 6).
