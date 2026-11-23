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
Dental is run as a separate business. The main day, home and month totals cover `CLK` (laser, injection, skin) plus general expenses and salaries.
Dental income, cash/card, expenses, net and debts appear in their own box (`dentalBox`).
Laser devices live in `settings/main.laserDevices` (`l1` Deka Again 1, `l2` Deka Again 2, `l3` Deka Motus, the old device kept for history).

## Login & roles
- Username + password. A username maps to the hidden address `<username>@orshina.clinic`.
- `users/{uid}`: `{username, role, empId}`. role = `owner | admin | operator | none`.
  - owner: everything.
  - admin (shown as إدارة): all clinic work, client files and debts, expenses, salaries (HR), companies and stock, and the day accounts (`vFinanceDay`, the cash received that day).
    No monthly money figures: no month income, expenses or net, no الصافي anywhere, the finance page is forced to day mode, and department pages hide month money.
    No log or trash, and no account or role management. In `perm()` these are `money` (owner only), `exp` (owner + admin), `owner` and `admin`.
    The monthly hiding is UI-only, because the ledger docs must stay readable. The log and users rules are enforced server-side (owner only).
  - operator: laser + skin care only (no money totals, no HR, no settings).
- The first run shows a setup screen that creates the owner (`meta/setup` marks it done).
- Staff accounts are created from the HR page through a secondary Firebase app instance (`createAccount`).
- **Security is enforced in `firestore.rules`.** Keep UI gating (`perm()` in the page) and the rules in sync.

## Data model (aggregated on purpose, to stay inside the free Spark quota)
Reads are kept low: the app never loads whole collections except the small pinned ones.
- `patients/{pid}`: one doc per client with everything in maps:
  `name, phone, birthYear, gender, allergies, conditions, notes, ref{src,detail}, chunk, teeth{FDI:{s,note}}, visits{vid:{type,date,d,price,note,createdAt,by}}, payments{payid:{dept,amount,date,note,vid?,m?}}, appts{aid:{date,time,svc,st}}`.
  Payment method: cash is the default and stores nothing; card payments carry `m:'card'` (in the patient doc and the ledger `p` entry). Day and month accounts split cash and card (`cashCard`).
  Imported clients have ids `o<oldId>`, with `src:'eclinic'`.
  `ref` records how the client heard about the clinic (شلون سمعتي عن عيادتنا). `src` is social | relative | people | walkin | other; `detail` holds the page name or relative's name.
  It is required for every new client, including ones created from the picker. The index mirrors it as `rs`/`rd`, and `refPanel` counts new clients (index `c`) per day and month.
- `index/{c00..}`: search index with ≤600 clients per doc:
  `items{pid:{c?,n,p,l,dep[],al,dp{dept:due},nx{dept:nextDate},rs?,rd?}}`. It is rebuilt from the patient doc on every patient write (`entryFor`).
- `ledger/{dept}_{YYYY-MM}_{a|b|c}`: monthly department books, sharded by day of month (1–10 a, 11–20 b, 21–31 c):
  `v{}` visits, `p{}` payments, `e{}` expenses. `ledger/hr_{YYYY-MM}_a`: `s{}` staff payments.
- `days/{YYYY-MM-DD}`: `appts{aid:{pid,n,time,svc,st,note}}`.
- `fixedpaid/{YYYY}`: fixed monthly costs (rent, license) already paid.
- `employees/{id}`: the public part `{name, job, dept, role, active, uid, username}`. `empPrivate/{id}` (admin only) holds `{salary, phone, start, notes}`.
- `settings/main`: `{laserDevices[], fixed[]}`.
- Deleting an entry inside a map is soft: the entry is set to `{del:true}`. Never delete patient financial history silently.
- Deleting a patient is soft too: `patients/{pid}.del=true` (plus `delAt`, `delBy`), and the index entry gets `del:true`.
  Only the owner can purge a file for good (rules: patients delete = owner). Deleted employees get `deleted:true` and stay in `S.empAll`, so their names still resolve.
- `suppliers/{sid}` (admin only) holds the companies (الشركات):
  - Fields: `name, kind (materials|maintenance|other), phone, notes, deleted?, tx{id:{type:'opening'|'buy'|'service'|'pay', date, desc, amount, items[{name,qty,unit,price,itemId,mvId}], expPath?, expId?, del?}}`.
  - Balance owed = opening + buy + service − pay. `opening` (دين سابق) is debt from before the app: no stock change and no expense.
  - A company payment is also written as a general expense (`cat:'شركات'`, with `sid`/`stx` links). Deleting either one removes the other.
- `stock/{itemId}` (admin only) holds the stock (المخزن):
  - Fields: `name, kind, unit, min, deleted?, mv{id:{q:+in/−out, t:'in'|'use'|'adj', date, ref, note?, del?}}`. Quantity = sum of live `q`.
  - A buy line adds an `in` movement.
  - An injection session with `d.stk` adds a `use` movement (`d.stkMv`, `d.stkQ`). Editing, deleting or restoring the session updates that movement.
- `home/*` (owner only, enforced in the rules) holds the owner's private home expenses (مصاريف البيت, the last tab after السجل, route `#house`):
  - `home/settings`: `{cats{id:{n,ic,c,o,subs[],del?}}, fixed{id:{n,cat,sub,amount,o,del?}}}`. If the doc is missing, the page uses the defaults `HCAT0`/`HFIX0`, and the first settings write saves them.
  - `home/m_{YYYY-MM}`: one doc per month, `x{id:{date,amount,cat,sub,note,fx?,at,del?}}`. `fx` links a payment to a fixed monthly item (shown as paid for that month). Paying a different amount becomes the item's new amount.
  - Logged with kind `home` (restorable from the trash) or `homeset` (settings changes).
- `log/{YYYY-MM-DD}/by/{uid}` is the activity log: `e{id:{at,who,act,kind,label,ref,before?}}`.
  - `act` is add | edit | delete | restore | purge.
  - Every data-changing action calls `logAct(...)`.
  - The rules make it append-only: each person writes only to their own doc and can add exactly one new entry per write. No one can change or delete entries from the app. Only the owner can read it.
  - The السجل page reads 1–31 days and restores deleted items from the `ref` stored in each delete entry (`restoreEntry`).

The page talks to Firestore through a tiny adapter (`DBA`). Its `update()` deep-merges nested maps using field paths, so concurrent adds of different visits don't clobber each other.

## Testing
Run the page against local emulators: `npx firebase-tools emulators:exec --project demo-orshina --only auth,firestore,hosting "<test>"`,
then open `http://127.0.0.1:5000/?emu=1` (the `?emu=1` switch only works on localhost).
