# AMANAH, PART 2: the commercial version (to sell to customers)

You are an expert product designer, senior engineer and security-minded architect. Build the app described below as a real product to SELL. It must include EVERYTHING in Part 1 (sections 1-10, included below in full) and then everything in Part 2 (sections 11 onward). Build Part 1 first, then extend it.

## HOW YOU MUST WORK (read first)
1. **Do not start coding yet.** First reply with: (a) a short plan, (b) the list of files and the folder structure, (c) the data model, (d) any real problems or conflicts you see in this description. Then WAIT for my approval.
2. After approval, build in the numbered **phases** at the end of this document. After each phase: run/test it, then give me a 5-line summary (what works, how to open it, what to test) and WAIT for me to say "continue".
3. Give COMPLETE files (not fragments) unless I ask for a patch. If a file is huge, split it into modules.
4. Do not invent features I did not ask for and do not remove anything described here. If something is unclear, make the most reasonable choice, state it in one line, and carry on.
5. If a requirement here is impossible on the platform (for example something that needs a server), tell me the closest honest alternative instead of faking it.
6. Keep chat replies short; put the work in the files.
7. If two sections of this document conflict, the LATER section wins.

## 1. What it is
**AMANAH** (Arabic/Amharic for "trust") is a private, offline-first **loan and money tracker** for people in Ethiopia. It records money the user **lent** and money the user **borrowed**, who owes what, repayments, due dates and reminders, and shows the user's net position at a glance. Everything is stored **on the phone only** (no server, no login, no ads). Currency: **Birr** (whole numbers shown with thousands separators, e.g. 15,000). Loans have **no interest** by design.

Target users: shop owners and ordinary people who lend and borrow informally and forget who owes what.

## 2. Platform and technical rules
- **Installable PWA**: one `index.html`, plain HTML + CSS + JavaScript (no frameworks, no build tools required for the user), `manifest.webmanifest`, service worker (network-first with offline fallback), icons (192, 512, maskable, apple-touch). Works fully offline. Hostable free on GitHub Pages.
- Mobile-first (design for 360 px width, also fine on laptops). Safe-area insets respected.
- Storage: `localStorage` (key `amanah.v1`), one JSON object. Handle a full or blocked storage gracefully (show a message, never crash).
- No install banner or "install app" button inside the app.
- Back button behaviour: phone/browser Back closes the top layer first (confirm dialog, then bottom sheet, then page, then returns to Home). When a loan is opened from a person's sheet, Back/X returns to that person's sheet, not to the list.
- Accessibility: tap targets at least 44 px, readable contrast in both themes, aria labels on icon buttons, Escape closes dialogs.
- All visible text goes through a translation function `T('English text', {placeholders})`.

## 3. Data model (one JSON object `db`)
```
db = {
  v: 1,
  settings: {lang, mode:'auto'|'light'|'dark', palette:'violet'|..., bgStyle:'charcoal'|'black'|'navy',
             cAccent, cLent, cOwe, textSize, pin (hash or ''), backupAt, remindLang:'en'|'am'|'om', ui:<version>},
  people:   [ {id, name, phone} ],
  loans:    [ {id, pid (person id), dir:'lent'|'borrowed', amount, ts (loan date), due (timestamp or 0),
               note, photo (compressed image data URL or ''), status:'open'|'forgiven'|'gift',
               payments:[ {id, ts, amount, note} ],
               methods:[ {m:'cash'|'telebirr'|'mpesa'|'cbebirr'|'cbe'|'abyssinia'|'other', label, amt} ] } ],
  activity: [ {id, ts, k (English message template), v (values)} ]   // newest actions history
}
```
Rules:
- Paid = sum of payments. Balance = max(0, amount - paid).
- A loan is **active** when status is open and balance > 0. Fully paid loans become "settled".
- **Forgiven** (only for lent loans): the person no longer owes it; history stays. **Gift**: not counted as debt.
- A person is reused by name (no duplicates). A person can have both lent and borrowed loans.
- Totals: *People owe you* = sum of active lent balances; *You owe* = sum of active borrowed balances; *Net* = owed to you minus you owe.
- Every action writes an activity entry (loan added, payment, payment undone, forgiven, gift, loan deleted, loan edited, settings or backup actions).

## 4. Navigation
Bottom bar, 5 items: **Home, People, Add (+) in a raised circular button, Activity, Settings**. Full pages (Notifications, a settings sub-page) open on top with a header and Back button and hide the bottom bar. Detail screens are **bottom sheets** (slide up, X button, scrim click closes).

## 5. Screens and behaviour

### 5.1 Home
- Top bar: title **AMANAH** (display font, uppercase), today's date under it, a **bell** button on the right with a red count badge (number of repayments needing attention) that opens Notifications.
- **Summary card (hero)**: a circular **ring** showing "% repaid" (paid / total of lent loans); to the right two stat blocks with a small progress bar each: **People owe you** (blue) and **You owe** (orange); below, a divider, then **Net position** (+/- amount, "Birr") and an **8-week trend** area chart of net position with the label "Last 8 weeks".
- If anything needs attention, an orange banner: "N repayments need attention" (opens Notifications).
- A two-option segmented switch: **I lent | I borrowed**.
- Search box "Search name or note" (filters as you type, keeps keyboard focus).
- Filter chips: **Open, Overdue, Due in 7 days, Settled**.
- Loan list (cards with rows): person name, note, balance on the right, and a small tag: *Overdue by N days* (red), *Due in N days* / *Due today* (amber), *Paid* (green), *Forgiven* / *Gift* (grey). Sorted by nearest due date, no-date loans last. Empty states with friendly text ("Nothing here. Tap + to add a loan.").
- No avatar circles with initials.

### 5.2 Add (+) and the loan form
- Tapping + opens a small sheet with two big choices: **I lent money** / **I borrowed money**.
- **Loan form** (sheet): Who (name; reuse an existing person if the name matches), Phone (optional, for reminders), Amount (Birr, numeric), Date (default today, cannot be in the future), **Repayment date** (optional) with quick chips **Tomorrow, In 7 days, In 14 days, In 30 days, In 60 days, No date** (these always count from TODAY, never from the loan date, which can be in the past), Note (optional, multi-line), **Payment method** (optional, see 5.3), **Add proof photo** (take or choose; compress to about 900 px JPEG before storing; preview with remove).
- Keyboard: Next on every field, Done on the last one, Enter inserts a new line only inside the note.
- Validation messages in plain language (e.g. "Enter the amount.").

### 5.3 Payment method (on each loan)
A collapsed row "Payment method (optional)" showing the choice underneath; tap to expand. Heading: lent = "How did you lend the money?", borrowed = "How did you receive the money?". Multi-select chips grouped: **Cash**; **Mobile wallets** (Telebirr, M-Pesa, CBE Birr); **Banks** (CBE, Bank of Abyssinia); **Other** (type a name). One method needs no amount. Two or more methods require an amount per method, with a live line "Allocated X of Y Birr" (red until equal to the loan amount); saving is blocked until it adds up. Shown in the loan details and in the CSV export.

### 5.4 Loan details (bottom sheet)
Person, direction, amount, balance, loan date, repayment date with status tag, note, payment methods, proof photo (tap to enlarge), **list of payments** (date, amount, note, each with Undo). Actions:
- **Record a payment received / Record a payment I made**: amount (default = balance, cannot exceed balance), date (default today), note. When the balance reaches 0 show "Fully paid. Mashallah!".
- **Remind** (lent loans): see 5.6.
- **Forgive the debt** and **Mark as a gift** (lent loans only; both ask for confirmation).
- **Edit** (all fields) and **Delete** (confirm).
Dangerous actions always use a confirm dialog with a red confirm button.

### 5.5 People
- Title "People" with the count. Search box (name or phone, live).
- Three tabs: **All** (default, everyone with any loan), **Owes me** (people with an active lent balance), **I owe** (people with an active borrowed balance). Each row: name, phone, amount, and label (owes you / you owe / settled). Sorted by amount. In the Owes me / I owe tabs the amount is that direction only. Empty states per tab.
- Tap a person: bottom sheet with name, total ("Owes you" / "You owe" / "All settled"), phone, a list of all their loans (type + amount, date, note, tag, balance at right; text must wrap before the amount column and never touch the card edges), buttons **Add loan** (pre-filled with that person) and **Edit** (rename, phone, with a safe delete).

### 5.6 Reminders
From a lent loan: choose **tone** (polite / firm) and **language** (English, Amharic አማርኛ, Afaan Oromoo). A message is generated from a template with {name}, {amt}, {due}; the user can edit it, then send via **WhatsApp** (`wa.me/<phone>?text=`), **Telegram** (`t.me/share/url`), **SMS** (`sms:`) or **Copy**. Phone numbers are normalised (Ethiopian numbers starting 09 become +2519...).

### 5.7 Activity (bottom tab)
A history of everything that happened, newest first, grouped by day (headings Today / Yesterday / date), each row = what happened + time. Filter chips: **Today (default), Yesterday, Choose a date, Date range, All**. "Choose a date" shows one date input; "Date range" shows From and To (swap automatically if reversed; both days inclusive). A line "N activities" and filter-specific empty states.

### 5.8 Notifications (page, opened from the bell or the banner)
Lists loans that are overdue or due within 3 days: red dot = overdue, amber = due soon; text "Abebe owes you 15,000 Birr" / "You owe Ahmed 1,200 Birr" with "Overdue by 13 days" / "Due in 2 days" / "Due today"; tap opens the loan. A note: alerts show inside the app when opened (real push notifications need a server and come later).

### 5.9 Settings (short menu, each row opens its own page, showing the current value under the title)
1. **Language**: reminder-message language (English / አማርኛ / Afaan Oromoo).
2. **Appearance and colors**: Theme **Automatic (default, follows the phone) / Light / Dark**; background style (Charcoal, Black, Navy); palette; custom colors with colour pickers (main accent, "owed to you" colour, "you owe" colour) and a reset; text size.
3. **Security**: **4-digit PIN** (set, change, turn off). Lock screen with a custom number pad; the app locks when reopened after 60 seconds in the background.
4. **Backup and data**: download a JSON backup (also copy/share), restore from a file (validated, confirm because it replaces everything), export loans to CSV/Excel, remove example data, erase all data (double confirm).
5. **About**: short privacy note: "Everything is saved on this phone only."

### 5.10 First run
Start empty (no fake data for real users). Optional: a welcome screen asking the user's name and a short animated AMANAH logo. A button to "Load example data" for a demo (all demo records flagged so they can be removed).

## 6. Visual design
- **Look**: modern fintech/fitness-app feel (similar spirit to MacroFactor): big clear numbers, rounded cards (about 20 px radius), generous spacing, calm.
- **Theme**: automatic light/dark. Dark = charcoal background (#1B1B1D), cards slightly lighter, thin borders; light = soft grey-white background, white cards. Accent colour **violet**. "Owed to you" = **blue**, "You owe" = **orange**, overdue = red, amber for due soon, green for paid/positive. Define every colour as a CSS variable with a light and a dark value; users can override accent and the two money colours.
- **Fonts**: **Unbounded** (wide display font) for the whole app and especially the title and big numbers; Noto Sans Ethiopic for Amharic. Scale sizes down slightly because Unbounded is wide (body about 13 px).
- Components: cards, list rows with a chevron, chips (selected = filled with the ink colour), segmented switch, bottom sheets, confirm dialogs, toasts, floating raised "+" button in the centre of the bar, area/ring SVG charts (no chart library).
- Motion: subtle (sheet slide, button press). Respect reduced-motion.

## 7. Languages
UI strings go through `T()`. English is the source; Amharic (አማርኛ, use "።") and Afaan Oromoo are added as dictionaries `DICT[lang][englishKey]`. Reminder templates exist in all 3 languages in polite and firm tones. A language selector for the whole UI is planned.

## 8. ADDITIONS FOR PERFECT DAILY USE (build after the base in sections 1-7)

### 8.1 Cash accounts and a real Net Worth (highest priority)
Goal: the owner tracks ALL his money, not only loans.
- Data: `db.accounts=[{id,name,type:'cash'|'bank'|'wallet'|'other',opening,ts}]`, `db.txns=[{id,acc,ts,kind:'in'|'out'|'transfer_in'|'transfer_out'|'adjust',amt,note,pair}]`. Balance is ALWAYS computed: opening + sum of txns (in and transfer_in add, out and transfer_out subtract, adjust adds its signed amount). Never store the balance.
- **Net worth = total of account balances + people owe you - you owe.**
- Home summary card: NET WORTH is the main number, with a small line "Cash X | Owed to you Y | You owe Z". Keep the ring, the two bars and the 8-week trend; the trend now shows net worth over time (cash at each point in time from the txns, plus the loan position at that time). With no accounts it behaves exactly like the base app.
- A "Your money" card on Home (under the summary): total cash and up to 3 accounts, "See all" opens the Accounts page; if there are none, a friendly card "Add your first account (cash, bank, Telebirr...)".
- **Accounts page** (full page): total, list of accounts with balances (negative in the warning colour), "Add account" (name, type chips Cash/Bank/Mobile wallet/Other, current balance). Tapping an account opens a sheet with its balance, recent transactions (newest first), and actions: **Add money, Take money out, Transfer, Set balance, Edit, Delete**.
- Transfer creates two linked txns so total cash is unchanged. **Set balance**: the user types the real balance and an 'adjust' txn is created for the difference. Deleting an account asks for confirmation; the other side of its transfers becomes a normal in/out txn so other accounts do not change. Any txn can be removed with confirmation.
- **Optional link with loans**: on a loan form and on "Record a payment", an optional "Which account?" selector (default: none). If chosen, the matching txn is created automatically (lending = money out, borrowing = money in, payment received = in, payment made = out) and is removed or adjusted if the loan or payment is deleted or edited.
- Every action is written to Activity with an English template key.
- Excel/CSV export of accounts: one row per txn (Date, Account, Type, Amount, Note, Balance after), opening balance first.

### 8.2 Expanded proof (on loans)
Replace the single photo with `proofs[]` = {id,type:'photo'|'pdf'|'file'|'ref'|'note',name,data,text,ts}. Migrate old single photos automatically. "Add proof" offers: Photo, PDF, Other document, Transaction ID / reference, Note. Up to 5 per loan; photos compressed; PDFs and documents up to 2 MB; allowed types image/*, application/pdf, text/plain, .doc/.docx; clear error messages for wrong type, size, limit or full storage. In loan details list them with icon, name, date; tap to open (photos in a viewer; files in a new tab through a Blob URL, never a data URL; text proofs with a Copy button); each can be removed with confirmation. Backup/restore must carry proofs (old backups with a single photo still restore).

### 8.3 Whole-app translation: English, Amharic (አማርኛ), Afaan Oromoo
- Source strings are English keys in `T('...')`. Dictionaries `DICT.am` and `DICT.om` map key to translation, with `{placeholder}` substitution. Fallback to English when a key is missing.
- Settings > Language: UI language and reminder language (separately). Remember the choice. Amharic uses "።". Month and weekday names are localised.
- Provide a `translate/strings-en.json` export of all strings plus a short prompt so a native speaker can translate and return JSON; add a validator that checks every key and every placeholder is present.
- Optional setting: show dates in the **Ethiopian calendar**.

### 8.4 First run, welcome and identity
- First run is EMPTY (no fake data). A 3-step welcome: language, your name, theme (Automatic/Light/Dark, default Automatic). Optional "Load example data" button for demo (flagged so it can be removed).
- A tasteful animated opening: the AMANAH wordmark animating in (letters rise/fade, about 1.2 s, skippable, shown on cold start only, disabled with reduced-motion), then "Welcome, [name]" greeting at the top of Home each new day.

### 8.5 Safety and trust
- **Backup reminder**: banner every 7 days since the last backup; one-tap backup; clear last-backup date in Settings.
- **Undo** (10 seconds) after deleting a loan, person, account or transaction.
- **Security**: choose a 4 or 6 digit PIN; PIN stored as a salted PBKDF2 hash (WebCrypto), never plain; lockout after 5 wrong tries (30 s, then growing); auto-lock after a configurable time in the background; **Hide amounts** eye toggle on Home that masks every number until tapped.
- Data is not sent anywhere. Say so in About.

### 8.6 Styled date picker (replace the phone's default date pickers everywhere)
A custom calendar bottom sheet that matches the app theme: month/year jump, today highlighted, min/max dates, quick chips (Today, Tomorrow, In 7/14/30 days, No date), range mode for Activity filters, localised month names (and Ethiopian calendar if enabled). Keyboard and screen-reader friendly.

### 8.7 Statements and sharing
Per-person statement: all loans, payments, balances, as formatted text and as a shareable image/PDF with the user's name and date. Share by WhatsApp/Telegram/SMS/copy.

### 8.8 Installment plans (optional per loan)
"Repayment plan": amount per week/month, number of instalments, first due date. The loan's next due date advances automatically as payments are recorded; reminders and notifications use the next instalment.

### 8.9 Insights
An "Insights" card on Home with a swipeable second chart: **Who owes you most** (top 5 horizontal bars) and a **Due soon timeline**. SVG only, theme-aware, readable in both themes.

### 8.10 Business summary import (for the owner's shop apps)
Settings > Backup and data > "Import business summary": accepts a small JSON {businessName, date, stockValue, cash, receivables} and records it as a line under "Businesses" in Net Worth (optional, can be added by hand too).

## 9. ACCEPTANCE CHECKLIST (all must pass)
1. Loans: add lent/borrowed, part-payments, undo, forgive, gift, settled; ring, bars, net and trend update instantly.
2. Date chips always count from TODAY; transaction dates are preserved.
3. People: search, All / Owes me / I owe tabs with correct per-direction amounts; Back from a loan returns to the person sheet.
4. Accounts: add, add money, take out, transfer (total cash unchanged), set balance, delete; Net Worth = cash + owed - owe; trend correct.
5. Loan <-> account link creates, updates and removes txns correctly.
6. Payment methods with split amounts must add up to the loan amount.
7. Proofs: all types, limits and errors; old photos migrate; backup and restore keep them.
8. Whole UI works in English, Amharic and Afaan Oromoo; missing keys fall back to English; no placeholder is lost.
9. PIN, lockout, hide amounts, backup reminder and undo work; the PIN is never stored in plain text.
10. Both themes are readable (contrast), nothing overflows at 360 px, tap targets are at least 44 px, works offline, installs to the home screen.
11. Old saved data from earlier versions loads without loss (write migrations and test them).

## 10. PHASES (PART 1)
1. Foundation: file structure, storage, migrations, helpers, theme system (Automatic/Light/Dark + custom colours), `T()` and the bottom bar.
2. Loans: model, add/edit/delete, payments, forgive/gift, loan details, Home with hero and lists.
3. People, search and tabs; person sheet.
4. Activity, Notifications and Settings menu with sub-pages.
5. Reminders in 3 languages with WhatsApp/Telegram/SMS/copy.
6. Payment methods and expanded proofs.
7. Cash accounts, Net Worth, loan-account link, Insights.
8. Security, backup/restore/export, undo, backup reminder, hide amounts.
9. Custom date picker, whole-app translation, Ethiopian calendar option, welcome and animation.
10. PWA: manifest, icons, offline service worker, install behaviour; final QA against section 9; write a short README.

## 11. PART 2: WHAT MAKES IT SELLABLE

### 11.1 Product definition
- Name: AMANAH, tagline idea "Your money and what you own, private and in your pocket." Audience: individuals and small business owners in Ethiopia (some with several businesses), mostly on low-to-mid Android phones, often on slow or expensive data.
- Principles: (1) **private by design**: the company must not be able to read user data; (2) **offline first**: everything works without internet, sync is optional; (3) **fast and light**; (4) **honest and simple**.
- Propose a plan structure (for example Free with limits, and Pro) and a price idea suitable for Ethiopia, and explain the reasoning briefly. Ask me to confirm before building billing.

### 11.2 Asset manager (the growth of Amanah beyond cash)
- Asset types: Cash and bank, Receivables (loans owed to you, automatic from loans), Vehicles, Land and property, Equipment, Stock and inventory, Investments, Other. Liabilities: loans you owe, plus custom liabilities.
- Each asset: name, type, value (with currency), acquired date and price, optional depreciation (straight-line, per year), photos and documents (receipt, title, contract), notes, owner/business tag.
- **Value history**: each revaluation is recorded with a date; charts show value over time.
- **Net worth dashboard**: total, by category (donut or stacked bar), by business, trend (8 weeks, 12 months), with the loan position included.
- **Businesses**: the user can define several businesses (for example three shops) and see each one's assets and the combined total. Importing a summary from the owner's separate shop apps (JSON file or QR code with stock value, cash, receivables) updates that business's line.

### 11.3 Accounts, login and licensing
- Optional account (sign up with phone number or email, one-time code, no passwords to remember; allow staying fully local without an account).
- **Trial and license**: 14-day free trial, then Free limits or a paid plan. License validated on the server and cached on the device; the app keeps working offline for at least 7 days after the last check, and never locks the user out of EXPORTING their own data, even when expired.
- Payments: research and propose options usable in Ethiopia (for example mobile money or a local payment gateway) plus a manual option where the owner approves a payment code. Never store card or wallet credentials in the app.

### 11.4 Encrypted cloud sync (optional, for several devices and phone loss)
- End-to-end encrypted: the app encrypts data on the device (AES-GCM, key derived from a user passphrase with PBKDF2 or Argon2) BEFORE upload. The server stores only ciphertext. Explain clearly in the UI that a forgotten passphrase cannot be recovered, and offer a printable recovery code.
- Offline-first queue, per-record last-write-wins with timestamps, conflict list for edits on two devices, sync status indicator, manual "Sync now", and a "delete all my cloud data" button.
- Suggested stack to evaluate: Supabase (Postgres, Auth, Row Level Security) or Firebase; justify your choice and give the schema and security rules. Keep the client in the same vanilla JS style (ES modules), unless you can justify something lighter for low-end phones.

### 11.5 Notifications
- Web Push with a service worker for due and overdue reminders, scheduled server-side from minimal metadata (or locally scheduled where the platform allows). Notification text must not reveal amounts unless the user enabled it. Quiet hours and per-type settings. Fall back to the in-app Notifications page.

### 11.6 Security suite
- Everything in 8.5, plus: **local data encrypted at rest** with a key derived from the PIN or passphrase; **biometric unlock** where available (WebAuthn); **decoy mode**: a second PIN opens a believable fake dataset (editable by the user, no sign that it is fake) while the real data stays hidden; **quick hide** (a gesture or button that instantly covers the screen); privacy screen when the app goes to the app switcher where possible; encrypted backup files with a passphrase; session list and "log out other devices" when sync is on.
- Provide a short threat model (who the user fears: people forcing them to show their wealth, theft of an unlocked phone, curious relatives, a hacked server) and show how each feature answers it.

### 11.7 Shared ledger (optional)
Invite the other person by phone number; both sides see and confirm the same loan; payments need confirmation from the other side; dispute flag; read-only statement link. The server only relays encrypted records.

### 11.8 Reports
Monthly summary, annual summary, who owes you most, aging of overdue loans (0-7, 8-30, 31+ days), cash-flow by account, exportable PDF and Excel with the user's name and date.

### 11.9 Multi-currency
Default Ethiopian Birr. Per-account and per-asset currency (USD, EUR, SAR, AED and others), manual exchange rates with date, totals shown in a base currency.

### 11.10 Admin and operations (privacy-safe)
A small admin dashboard: number of users, active users, plan and license status, sync health, error counts. No access to decrypted user data. Privacy-friendly, opt-in diagnostics only. Feature flags and a remote "minimum supported version" check with an update prompt.

### 11.11 Packaging and distribution
- Keep the PWA as the primary product, and package it for Android as a Trusted Web Activity or with Capacitor/PWABuilder; prepare Play Store assets (icons, screenshots, short and long description in 3 languages), versioning, update flow and migrations. Explain what iOS can and cannot do for a PWA.
- Landing page (single page, 3 languages) explaining the product, the privacy promise, pricing, install steps, FAQ and a WhatsApp/Telegram support link.

### 11.12 Trust, legal and support
Privacy policy and terms in plain language and 3 languages, account and data deletion, data export, clear statement that the company cannot read user data, in-app help and FAQ, a "Contact support" button (WhatsApp/Telegram), crash-safe behaviour (never lose data; always keep the last good copy).

### 11.13 Quality bar
- Unit tests for money maths, balances, net worth, migrations and encryption round-trips; end-to-end tests for the main flows; accessibility pass; performance budget for a low-end Android phone on a slow connection (target: first load under 3 seconds on 3G, JavaScript under about 200 KB gzipped, smooth scrolling with 2,000 loans).
- Every data-structure change ships with a migration and a test using real old data.

## 12. ACCEPTANCE CHECKLIST (PART 2, in addition to section 9)
1. Net worth dashboard is correct across accounts, loans, assets and businesses, including currency conversion.
2. Encrypted sync: two devices converge; the server holds only ciphertext (show proof); wrong passphrase cannot read data.
3. Licence: trial, expiry, offline grace; export always works.
4. Decoy PIN shows the fake dataset and never leaks the real one in activity, search, notifications or backups.
5. Push notifications arrive and respect quiet hours and privacy settings.
6. Shared ledger confirmations and disputes work between two accounts.
7. PDF/Excel reports match the on-screen numbers.
8. Android package installs, updates, and keeps data across updates.
9. Tests pass; performance budget met on a low-end device profile.

## 13. PHASES (PART 2, after Part 1 is complete)
11. **Architecture**: write the architecture document, full data schema (local and server), the threat model, and the plan and pricing proposal. WAIT for approval.
12. **Refactor and foundations**: modules, migrations, test setup, encryption helpers, performance budget.
13. **Asset manager and businesses**: asset types, value history, dashboard, business summary import.
14. **Security suite**: encryption at rest, biometric, decoy mode, quick hide, encrypted backups.
15. **Backend and sync**: auth, schema, row-level security, encrypted sync, conflict handling, cloud delete.
16. **Licensing and payments**: trial, plans, licence checks, offline grace, payment options.
17. **Notifications**: web push, scheduling, privacy settings.
18. **Shared ledger** (can be postponed if I say so).
19. **Reports and multi-currency**.
20. **Admin dashboard, landing page, legal pages, in-app help**.
21. **Packaging**: Android build, store listing assets, update flow.
22. **Final QA** against sections 9 and 12, a launch checklist, and a short operator guide for me (how to deploy, back up, support customers, handle a lost passphrase request).

## FINAL STEP: REVIEW THIS PROMPT BEFORE YOU BUILD
Before writing any code, review THIS document critically and reply with a SHORT report:
1. **Contradictions or unclear parts** (quote the line and propose a fix).
2. **Missing features or edge cases** that a real user of a loan and money tracker in Ethiopia would expect (for example: wrong dates, negative balances, duplicate names, very large numbers, time zones, low-end Android phones, slow connections, storage limits, data loss).
3. **Risks** (security, privacy, legal, technical limits of a PWA, browser storage limits) and how to reduce each.
4. **Suggested additions** that are closely related and would clearly improve the product, ranked by value and effort. Do not add them yet.
5. **A corrected, improved version of this prompt** if you found real problems (only the changed sections).
Then ask me which suggestions to accept and WAIT. Do not build until I approve.
