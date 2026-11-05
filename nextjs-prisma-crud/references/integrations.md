# Integrations

Every external service has one client or helper in the repo. Find it and call it; a second
sender for the same service is a bug waiting to diverge.

## Email

Usually Nodemailer over SMTP with a cached transporter. Credentials come from environment
variables (`SMTP_*` or similar) and never appear in logs, code, or reports. HTML templates
live where the existing ones live.

## WhatsApp

Providers vary. When reusing the wrapper, follow how it handles phone number normalization,
session selection, timeouts, media, errors, and message templates.

## File uploads (Cloudinary or similar)

- Accepted types and size limits are enforced on the server.
- Upload folder and resource type (image, raw/PDF) match the existing uploads.
- The returned URL is saved on the owning record, inside the same scoped write.
- The uploader is authorized for that record like any other write (see `security.md`).

## External APIs

Calls go through the repo's service or `lib` wrapper, from server code only, so auth headers
stay on the server. Live data uses `cache: "no-store"` (or the repo's equivalent).

## Cron jobs (Vercel or similar)

Before changing a scheduled job, establish:

- **Authorization**: a secret header or token (for example `CRON_SECRET`) that rejects any
  other caller.
- **Idempotency**: running it twice produces no duplicate emails, messages, or charges.
- **Timezone**: when it runs and which dates it computes.
- **Side effects**: every external effect it triggers.

Test it against a safe target, or describe the test for the user; production runs are the
user's call.

## Docker

Only when the repo already uses it or the task asks for local infrastructure, usually a local
PostgreSQL, MySQL, or MariaDB.
