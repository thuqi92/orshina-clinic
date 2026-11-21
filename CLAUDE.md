# Orshina clinic app

Clinic management web app for **Orshina** (اورشينا), a dental + aesthetics clinic in Iraq.
UI language is Iraqi Arabic, right-to-left. Money is IQD (plain integers, shown as `150,000 د.ع`).

## Stack
- One static page: `public/index.html` (HTML + CSS + vanilla JS, no build step).
- Firebase compat SDK v10.12.2 is vendored in `public/vendor/` (CDNs may be blocked; keep it vendored).
- Firebase project `orshina-clinic`: Hosting, Firestore, Authentication (Email/Password provider).
- Deploy: push to `main` → `.github/workflows/deploy.yml` deploys hosting + `firestore.rules`
  using the `FIREBASE_SERVICE_ACCOUNT` repo secret. Never commit credentials or patient data.

## Departments
`dental` (الأسنان), `laser` (الليزر), `injection` (الحقن), `skin` (تنظيف البشرة). `general` is for expenses only.
Laser devices live in `settings/main.laserDevices` (`l1` Deka Again 1, `l2` Deka Again 2, `l3` Deka Motus, the old device kept for history).

## Login & roles
- Username + password. A username maps to the hidden address `<username>@orshina.clinic`.
- `users/{uid}`: `{username, role, empId}`. role = `owner | admin | operator | none`.
  - owner/admin: everything. operator: laser + skin care only (no money totals, no HR, no settings).
- The first run shows a setup screen that creates the owner (`meta/setup` marks it done).
- Staff accounts are created from the HR page through a secondary Firebase app instance (`createAccount`).
- **Security is enforced in `firestore.rules`.** Keep UI gating (`perm()` in the page) and the rules in sync.

## Data model (aggregated on purpose, to stay inside the free Spark quota)
Reads are kept low: the app never loads whole collections except the small pinned ones.
- `patients/{pid}`: one doc per client with everything in maps:
  `name, phone, birthYear, gender, allergies, conditions, notes, chunk, teeth{FDI:{s,note}}, visits{vid:{type,date,d,price,note,createdAt,by}}, payments{payid:{dept,amount,date,note,vid?}}, appts{aid:{date,time,svc,st}}`.
  Imported clients have ids `o<oldId>`, with `src:'eclinic'`.
- `index/{c00..}`: search index with ≤600 clients per doc:
  `items{pid:{n,p,l,dep[],al,dp{dept:due},nx{dept:nextDate}}}`. It is rebuilt from the patient doc on every patient write (`entryFor`).
- `ledger/{dept}_{YYYY-MM}_{a|b|c}`: monthly department books, sharded by day of month (1–10 a, 11–20 b, 21–31 c):
  `v{}` visits, `p{}` payments, `e{}` expenses. `ledger/hr_{YYYY-MM}_a`: `s{}` staff payments.
- `days/{YYYY-MM-DD}`: `appts{aid:{pid,n,time,svc,st,note}}`.
- `fixedpaid/{YYYY}`: fixed monthly costs (rent, license) already paid.
- `employees/{id}`: the public part `{name, job, dept, role, active, uid, username}`. `empPrivate/{id}` (admin only) holds `{salary, phone, start, notes}`.
- `settings/main`: `{laserDevices[], fixed[]}`.
- Deleting an entry inside a map is soft: the entry is set to `{del:true}`. Never delete patient financial history silently.

The page talks to Firestore through a tiny adapter (`DBA`). Its `update()` deep-merges nested maps using field paths, so concurrent adds of different visits don't clobber each other.

## Testing
Run the page against local emulators: `npx firebase-tools emulators:exec --project demo-orshina --only auth,firestore,hosting "<test>"`,
then open `http://127.0.0.1:5000/?emu=1` (the `?emu=1` switch only works on localhost).
