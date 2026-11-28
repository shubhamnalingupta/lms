# Frappe LMS as backend, one Expo app for Android, iOS and web

We use Frappe LMS (Python/Frappe/MariaDB) as the backend, including auth and data, and keep it synced with **Upstream** so we inherit its fixes. All three learner surfaces (Android, iOS, learn.techiegigs.com) are one Expo/React Native codebase talking to the Frappe API, so the custom gamified UI is written once. Supabase was rejected: replacing Frappe's database and auth would mean rewriting the backend and losing **Upstream** updates.

## Consequences

Frappe's own Vue web UI is replaced for learners, so **Upstream** UI improvements don't reach us; only backend ones do.
