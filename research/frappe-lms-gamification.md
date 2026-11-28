# Frappe LMS gamification: what exists, and where a custom app can extend it

Answers issue #4. Researched 2026-11-28.

**Sources read**
- `frappe/lms` @ `adb19c73a3d884b9dc84794475ece8330275b329` (2026-11-26, app version 2.45.2). Paths below are relative to that repo.
- `frappe/frappe` `develop` @ `6b2847e9d4f629e80de0ef4135b6f38dcc80e493` (17.0.0-dev), plus the file trees of the `version-15` and `version-16` branches.
- Duolingo first-party posts and the Sailer & Homner meta-analysis, as listed in §3. **Caveat:** this sandbox could not fetch `blog.duolingo.com` or `lennysnewsletter.com` directly, because DNS for both is blocked. The Duolingo claims below come from search-engine summaries of those first-party pages. Check them against the pages before anyone quotes a number.

---

## TL;DR

- **Badges: yes, and they are rule-based.** `LMS Badge` holds an auto-award rule that can point at *any* doctype. `LMS Badge Assignment` records each award. A `"*"` `on_change` hook evaluates the rules.
- **Streaks: yes, a minimal version.** `get_streak_info` computes current and longest streak on the fly. Weekends are skipped, nothing is stored, and there are no freezes. There is also an activity heatmap.
- **Points / XP, leaderboards, leagues, rewards, hearts: none.** Frappe's Energy Points and leaderboard existed up to `version-15` and are gone from `version-16` and `develop`. LMS `main` requires `frappe >=17.0.0-dev`, so they are not available.
- **Extension without a fork works well.** A custom app can add its own `doc_events`, which run alongside the ones LMS registers. It can also add `scheduler_events`, its own whitelisted APIs, and badge fixtures, and can add a sidebar link via `LMS Sidebar Item`. Two gotchas follow.
  - One progress path skips hooks: SCORM progress updates use `frappe.db.set_value`.
  - The LMS Vue SPA has no plugin slot. Custom UI has to be a separate page.

---

## 1. What upstream Frappe LMS ships

### 1.1 Badges: `LMS Badge` + `LMS Badge Assignment`

**`LMS Badge`** (`lms/lms/doctype/lms_badge/lms_badge.json`) has these fields: `title`, `image`, `description`, `enabled`, `reference_doctype` (Link → DocType), `event` (`New` | `Value Change` | `Manual Assignment`), `condition` (Code), `user_field` (the field on the reference doc that holds the user), `field_to_check`, and `grant_only_once`.

The engine lives in `lms/lms/doctype/lms_badge/lms_badge.py`:
- `hooks.py` registers `doc_events = {"*": {"on_change": ["lms.lms.doctype.lms_badge.lms_badge.process_badges"]}}`. **Every** document change on the site triggers it.
- `process_badges` returns early during patch, install, migrate, import or setup wizard. Otherwise it loads the enabled badges for `doc.doctype`, using `frappe.cache_manager.get_doctype_map`, and calls `apply_badge`.
- `apply_badge` runs each badge inside a DB savepoint. A badge error is rolled back and logged, and never fails the user's save.
- `rule_condition_satisfied` works as follows:
  - `condition` is a Python expression evaluated with `frappe.safe_eval(condition, None, {"doc": doc.as_dict()})`.
  - `New` fires only when there is no `doc_before_save`.
  - `Value Change` fires only when the condition flips from false to true.
  - `Manual Assignment` never auto-fires.
- `award()` skips the award if one already exists when the badge has `grant_only_once` or uses `Value Change` (it takes a row lock first). It then inserts an `LMS Badge Assignment` with `ignore_permissions=True`.
- `assign_badge(badge_name)` is whitelisted for POST. Moderator, Course Creator and Batch Evaluator can call it. For `Manual Assignment` badges, `condition` is a **JSON filter** run through `frappe.get_all` to bulk-award.

**`LMS Badge Assignment`** (`lms_badge_assignment.json` / `.py`) has `member`, `badge`, `issued_on`, plus fetched `badge_image`, `badge_description` and member fields. Its `validate` does three things:
- rejects duplicates when the badge has `grant_only_once`
- re-checks that *some* doc of `reference_doctype` owned by the member satisfies `condition`, but only if `condition` is set
- restricts Manual-event badges to admins

**Surfacing.** `lms.lms.api.get_badges(member)` (`lms/lms/api.py` ~L4542) feeds the profile page (`frontend/src/pages/ProfileAbout.vue`). Admins configure badges in Settings (`frontend/src/components/Settings/Badges/`).

**Limitation that matters.** A condition sees **one document** (`doc`). It cannot express anything cross-record, such as "completed 10 lessons", "7-day streak", "first course finished" or "top 10 this week". Those need code.

### 1.2 Streaks and the activity heatmap (`lms/lms/api.py`)

- `get_streak_info()` (~L3552) is whitelisted and only ever answers for `frappe.session.user`.
  - `fetch_activity_dates` collects the `creation` dates of `LMS Course Progress`, `LMS Quiz Submission`, `LMS Assignment Submission` and `LMS Programming Exercise Submission`.
  - `calculate_streaks` skips Saturdays and Sundays and walks consecutive weekdays.
  - `calculate_current_streak` returns the streak only if the last active day is the reference weekday or the day before it.
  - It returns `{current_streak, longest_streak}`.
- **Not present:** stored streak state, streak freezes or repair, timezone handling beyond server dates, configurable weekend rules, and streak milestones.
- UI: a 🔥 counter on Home (`frontend/src/pages/Home/Home.vue`) opens `Streak.vue`. The dialog explains that "weekends don't break your streak".
- `get_heatmap_data(member, base_days=200)` (~L2559) counts lesson completions plus quiz and assignment submissions per day, in a GitHub-style heatmap. It is visible to the learner, Moderators, and staff of the learner's course or batch.

### 1.3 Points, XP, leaderboards, leagues, rewards: none in LMS

`grep -i` across `lms/` for streak, leaderboard, energy, xp and points found no points, leaderboard or reward model. The only matches were the streak and heatmap code above.

**Frappe framework Energy Points:**
- The `version-15` branch has `frappe/social/doctype/energy_point_rule`, `energy_point_log`, `energy_point_settings` and `frappe/desk/leaderboard.py`.
- The `version-16` branch and `develop` have none of these files.
- `frappe/lms` `pyproject.toml` declares `[tool.bench.frappe-dependencies] frappe = ">=17.0.0-dev,<18.0.0"`.
- **So the framework's points and leaderboard cannot be reused on current LMS.** I did not determine whether Energy Points moved to a separate app.

### 1.4 Learning events a gamification layer can listen to

| Event | Doctype / trigger | Notes (source) |
|---|---|---|
| Lesson completed | `LMS Course Progress` insert with `status="Complete"` | Created by `save_progress` → `_save_progress` (`course_lesson.py` ~L570–690) or by `_record_graded_completion` after grading. One row per (member, lesson); `before_insert` rejects duplicates, so `after_insert` = "first completion". |
| SCORM lesson completed | `LMS Course Progress` | **Gotcha:** updating an *existing* SCORM row from `Partially Complete` to `Complete` uses `frappe.db.set_value` (`course_lesson.py` ~L650). **No doc_events fire.** Only the first insert fires them. |
| Course progress / course completed | `LMS Enrollment.progress` (Float), `on_update` | `_save_progress` now writes through `update_enrollment()` inside `batched_enrollment_updates()` specifically so `on_update` and webhooks fire once (see comment ~L675 and `test_save_progress.py`). `progress == 100` → course done. |
| Quiz attempt | `LMS Quiz Submission` `after_insert` | Fields: `score`, `score_out_of`, `percentage`, `passing_percentage`, `quiz`, `course`, `member`. Retakes create new rows, so dedupe on (member, quiz) for "first pass". A later grade that changes `percentage` fires `on_update`. |
| Assignment passed | `LMS Assignment Submission` `on_update` with `status` changing to `Pass` | Evaluator grades it. LMS itself reacts in `complete_lessons_on_pass`. |
| Programming exercise | `LMS Programming Exercise Submission` | Counted for streaks. |
| Certificate issued | `LMS Certificate` `after_insert` | LMS sends the certificate email here. |
| Enrolled | `LMS Enrollment` / `LMS Batch Enrollment` `after_insert` | Already hooked by LMS telemetry. |

---

## 2. Extension points for a separate custom app (no fork)

### 2.1 `doc_events` in the custom app's `hooks.py`
`frappe.model.document.Document.hook` → `composer` collects handlers from `frappe.get_doc_hooks()` for `doc.doctype` and for `"*"`. Handlers from every installed app are combined (`frappe/model/document.py` ~L2488). A custom app's handlers therefore run alongside LMS's, with nothing overridden:

```python
# gamify/hooks.py
doc_events = {
    "LMS Course Progress": {"after_insert": "gamify.events.on_lesson_complete"},
    "LMS Quiz Submission": {"after_insert": "gamify.events.on_quiz_submit",
                            "on_update": "gamify.events.on_quiz_regrade"},
    "LMS Assignment Submission": {"on_update": "gamify.events.on_assignment_update"},
    "LMS Enrollment": {"on_update": "gamify.events.on_enrollment_update"},  # progress==100
    "LMS Certificate": {"after_insert": "gamify.events.on_certificate"},
}
```
Recommendations:
- Keep handlers idempotent with an `XP Ledger` row keyed on (member, source doctype, source name, reason) and a unique index. Retakes, regrades and the "first insert vs later update" cases above make this necessary.
- Push heavy work to `frappe.enqueue(..., enqueue_after_commit=True)`.
- Copy LMS's savepoint pattern from `apply_badge` so a gamification bug never blocks a lesson save.

### 2.2 Server Scripts, Notifications and Webhooks (no app code)
`Document.run_method` calls, in order: the controller method plus hooks, `run_notifications`, `run_webhooks`, `run_server_script_for_doc_event` and `run_automations` (`frappe/model/document.py` ~L2069–2093).
- The `Webhook` doctype supports `after_insert`, `on_update`, `on_submit`, `on_cancel`, `on_trash`, `on_update_after_submit`, `on_change` and `workflow_transition` (`frappe/integrations/doctype/webhook/webhook.json`). These are queued as background jobs (`enqueue_webhook`). Use them if the XP engine lives outside Frappe.
- None of these fire for the SCORM `db.set_value` path described above.

### 2.3 `scheduler_events`
LMS already uses `hourly` and `daily` (`hooks.py` ~L231). A custom app can add its own, for example:
- `daily`: streak-at-risk reminders, freeze consumption, snapshotting streaks into a stored doctype
- `weekly` or `cron`: closing league weeks and promoting or demoting learners

### 2.4 Whitelisted APIs
- The custom app exposes its own `@frappe.whitelist()` endpoints, such as `get_my_xp`, `get_leaderboard` and `use_streak_freeze`.
- It can call LMS's `get_streak_info` / `get_heatmap_data` or re-implement the logic with freezes.
- `override_whitelisted_methods` (LMS itself uses it, `hooks.py` ~L261) can wrap an LMS method such as `lms.lms.api.get_streak_info` to add freeze-aware logic without forking. It is brittle across upgrades, so use it sparingly.
- `frappe.publish_realtime(event=..., user=member, after_commit=True)` can push "+10 XP" toasts. LMS already does this for `update_lesson_progress`.

### 2.5 Reuse the badge engine
- **Simple badges need no code.** Ship `LMS Badge` records as fixtures, for example on `LMS Certificate` with `New` and `doc.course == "X"`, or on `LMS Quiz Submission` with `Value Change` and `doc.percentage == 100`.
- **Cross-record badges** (N lessons, streak length, level reached) need custom-app code that inserts an `LMS Badge Assignment` directly. Give those badges `event` other than Manual and an empty `condition`. `validate_badge_criteria` then skips the re-check, and they still appear on the profile through `get_badges`.

### 2.6 UI
- The learner UI is a compiled Vue SPA (`frontend/`). I found no slot or hook for a third-party app to inject components into it.
- **Options:**
  - add an `LMS Sidebar Item` row (`item_type` = `Web Page` | `Route` | `External`) pointing at the custom app's own page (a leaderboard, XP history)
  - use realtime toasts
  - otherwise contribute upstream
- Treat in-SPA widgets as an open question.

### 2.7 Data model a custom app would add (suggested, not upstream)
- `XP Ledger`: member, points, reason, ref doctype/name, unique key
- `Learner Gamification`: member, total XP, level, current/longest streak, freezes owned, last active date, timezone
- `League Week`: week, tier, members, ranks

---

## 3. How leading gamified learning apps structure the core loop

All Duolingo numbers are Duolingo's own reported, mostly *relative* A/B results. They are not independently verified, and per the caveat above they were read via search summaries of the pages.

**Core loop.** Complete a lesson → earn **XP** → meet a daily goal and extend the **streak** → XP also ranks you on a weekly **league** leaderboard → return tomorrow.
- XP comes from lessons, Stories, practice and timed challenges, and feeds the weekly Leaderboards, where top learners advance a league ([Duolingo 101](https://blog.duolingo.com/duolingo-101-how-to-learn-a-language-on-duolingo/)).

**Streaks.**
- [Habit-research post](https://blog.duolingo.com/how-duolingo-streak-builds-habit/):
  - learners who reach a 7-day streak are reported 2.4× more likely to come back the next day
  - letting learners equip up to **two Streak Freezes** raised daily active learners by +0.38% (relative)
  - streak animations raised 7-day retention of brand-new learners by +1.7%
- [Improving the streak](https://blog.duolingo.com/improving-the-streak/): separating the streak from the daily goal raised Day-14 retention by 3.3% and daily learners on a streak by 10.5% (relative).
- [Streak Wager](https://blog.duolingo.com/how-streaks-keep-duolingo-learners-committed-to-their-language-goals/): staking in-game currency on 7 more days led to more returns and more lessons completed. No figure was captured.
- Freezes must be equipped in advance as "insurance", and Duolingo repairs streaks broken by outages ([protecting streaks](https://blog.duolingo.com/protecting-streaks-from-site-issues/)).
- Duolingo's retention PM said in an interview that the team has run 600+ streak experiments and focuses on the first week ([Lenny's Newsletter](https://lennysnewsletter.com/p/behind-the-product-duolingo-streaks)). Search summary only.

**Leagues** ([leaderboards post](https://blog.duolingo.com/duolingo-leagues-leaderboards/)):
- first tested in 2018 with 5 leagues; there are now 10, topped by Diamond
- weekly cohorts with promotion and demotion
- an iOS experiment increased lessons started and completed, and the feature was launched to all users. No figure was captured.
- Duolingo later raised XP for path lessons because leagues rewarded XP grinding. **Design lesson:** weight XP toward real learning, not easy farming.

**Hearts → Energy** ([Energy post](https://blog.duolingo.com/duolingo-energy/)):
- free users lost 1 heart per mistake
- Energy replaced Hearts as a battery that lessons drain and that refills over time and with correct answers, recharging fully in about a day
- this is a monetisation and pacing lever, not a learning one, and probably not worth copying for a course LMS

**Achievements / badges.** Duolingo has achievement badges ([post](https://blog.duolingo.com/achievement-badges/)).

**Learning-quality guardrail.** Duolingo's "Time Spent Learning Well" metric favours lessons that drive learning. It cites an external study finding completed lessons the strongest predictor of learning gains ([post](https://blog.duolingo.com/time-spent-learning-well/)).

**Research evidence.** Sailer & Homner (2020), *Educational Psychology Review* 32:77–112, [doi:10.1007/s10648-019-09498-w](https://link.springer.com/article/10.1007/s10648-019-09498-w):
- small positive effects of gamification on cognitive (g = .49) and motivational (g = .36) outcomes; the cognitive effect held in high-rigor studies
- for behavioural outcomes, game fiction and social interaction were significant moderators, and *competition combined with collaboration* worked best
- later meta-analyses call the evidence mixed ([Li, Ma & Shi 2023](https://pmc.ncbi.nlm.nih.gov/articles/PMC10591086/))

### Implications for this project
1. Make lesson completion the XP source, with first completion only. Add bonuses for passing a quiz, passing an assignment, a certificate and course completion. Don't award XP for raw attempts.
2. Store streaks in a doctype and add freezes, given Duolingo's freeze and first-week results. Keep LMS's weekend-skip as a setting rather than hard-coding it.
3. Leagues are last. They need enough concurrent learners per cohort to be meaningful, and the XP-grinding risk is real.
4. Prefer team or cohort challenges, which add collaboration on top of competition, in line with Sailer & Homner.

## Open questions
- Can a third-party Frappe app inject UI into the LMS SPA (Home 🔥 area, profile)? I found no mechanism.
- Should SCORM completion be fixed upstream to fire `on_update` (a small PR: use `doc.save()` instead of `db.set_value`)?
- Did Energy Points move to a separate app after `version-15`? Not verified.
