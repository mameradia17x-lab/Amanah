# AMANAH: full description (use this to rebuild the app from scratch)

> HOW TO USE THIS FILE: paste it into any AI app builder or coding assistant.
> Tell it: "Build this app. First show me a short plan and the list of files, wait for my approval, then build step by step. Keep answers brief."
> Build in this order: 1) data + loans + home, 2) people, 3) payments + reminders, 4) activity + notifications, 5) settings, PIN, backup, 6) polish, languages, offline install.

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

## 8. Planned next (not built yet; design so they fit later)
Cash accounts and a real Net Worth (accounts + owed to you - you owe) with transfers and balance history; richer proof (PDF, transaction ID, several files); styled date picker; installment plans; stronger security (longer PIN, lockout, decoy PIN that opens a fake dataset, hide-amounts mode); cloud sync and real push notifications; asset types (vehicles, land, shop stock) and import of summaries from the shop app.

## 9. Acceptance checklist
1. Adding a lent and a borrowed loan updates the ring, both bars, net position and the trend instantly.
2. Part-payments reduce the balance; Undo restores it; paying everything marks it Paid.
3. "Tomorrow" and the other date chips are always relative to today, even when the loan date is months ago.
4. People tabs show correct amounts per direction; a person with both kinds appears in both tabs.
5. Reminder opens WhatsApp with the edited message in the chosen language.
6. Activity shows every action with correct day grouping and filters.
7. PIN locks and unlocks; backup then restore on a clean install gives identical data.
8. Works offline after the first load; installs to the home screen; no horizontal scroll at 360 px; both themes readable.
