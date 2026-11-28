# Ship the new TechieGigs app as an update to the existing store listings

TechieGigs already has a live app on the App Store and Play Store. The new **Branded App** reuses the same bundle ID / package name and ships as an update, so existing **Learners** upgrade in place and the listings keep their ratings, reviews and installs. Two listings can't be merged later, so a fresh listing was rejected.

## Consequences

We need the current Android signing key or Play App Signing access (an upload-key reset via Google if lost), and the cutover must carry over existing **Learner** accounts so the update doesn't log everyone out into an empty app.
