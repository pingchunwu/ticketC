# TicketC Feature Documentation

TicketC (v2.1) is a personal iOS app for two people to exchange "tickets" (coupons/vouchers).
Each user can write a limited number of tickets, view the other person's tickets, and redeem
("check") the ones they've been given. The app also has switchable themes with background music, a
Christmas snowfall effect, and a hidden "DLC" that unlocks a Minion theme and a secret letter.

This document covers what each feature does and how it is implemented.

---

## 1. Tech stack and architecture

| Item | Details |
| --- | --- |
| Platform | iOS 13.0+, Swift 5, UIKit + Storyboards / XIBs |
| Architecture | MVVM (`ticketC/M`, `ticketC/VM`, `ticketC/VC`) |
| Networking | Alamofire 5.2 + SwiftyJSON 4 (`Utility/NetworkController.swift`) |
| Backend (current) | Google Apps Script web app that reads and writes a Google Sheet (`NetworkController.urlSheet`) |
| Backend (legacy) | Node.js / Express + MySQL server in `backend/` (`NetworkController.urlDatabase`) |
| Analytics | Firebase Analytics (`FirebaseApp.configure()` in `AppDelegate`) |
| Localization | `zh-Hant` (source strings) and `en` (`Utility/*/Localizable.strings`) |
| Local storage | `UserDefaults`, wrapped by `Utility/Tools.swift` |

### MVVM and data binding

- **`LiveData<T>`** (`Utility/LiveData.swift`) is a minimal observable. Setting `value` calls the
  single registered listener, and `observe(listener:)` calls the listener right away with the current
  value. A `LiveData` holds **one** listener, so a later `observe` call replaces the earlier one.
- **Models** (`M/`) hold raw state. For example, `ListPageM` holds the ticket lists and the quota.
- **ViewModels** (`VM/`) subclass `BaseVM`, which provides the shared `Tools`, `Config` and
  `NetworkController` singletons and a `connectionError: LiveData<Int?>`. ViewModels call the backend
  and publish the results through `LiveData`.
- **ViewControllers** (`VC/`) subclass `BaseVC`. They observe ViewModel `LiveData`, update the UI, and
  override `setStyle()` to apply the current theme.

### Global singletons

- **`Config.sharedInstance`** (`Utility/Config.swift`) holds the current theme (`currentStyle`), the
  theme palette (`styleColor`), the audio player, the embargo/DLC flag, the user names and today's
  date string.
- **`NetworkController.sharedInstance`**: the HTTP client. Requests time out after 20 seconds.
- **`Tools.sharedInstance`**: JSON-encodes and decodes `Bool` and `Style` values in `UserDefaults`.

### Users and test mode

`Config.setUsers()` hard-codes the two users of the app:

| `isTestMode` | `currentUser` (me) | `objectUser` (partner) |
| --- | --- | --- |
| `false` (default) | `Christina` | `Billy` |
| `true` | `testA` | `testB` |

Every ticket request sends one of these names as `user`.

---

## 2. Screen flow

```
LaunchScreen ("TicketC v2.1 – Tickets for Christina")
   │
   ▼
LoginPageVC ──(empty name / invalid name)──► InstructionVC (About, modal)
   │ valid name, or already logged in
   ▼
TabBarC (TBC, full-screen modal)
   ├── Tab 0: ListPageVC  "兌換券 / Tickets"
   └── Tab 1: OptionVC    "選項 / Options"
                 ├── 給寶貝的信 / Letter ─► BirthdayLetterVC   (DLC + Minion theme only)
                 ├── 更換主題 / Theme    ─► theme picker action sheet
                 ├── 關於 / About        ─► InstructionVC
                 └── 登出 / Log out      ─► back to LoginPageVC
```

Switching tabs uses a 0.3 s cross-dissolve transition (`TabBarC`'s `UITabBarControllerDelegate`).

---

## 3. Features

### 3.1 Login (`LoginPageVC`, `LoginPageVM`)

- The user types a "special name" in the text field and taps **Confirm**.
- `LoginPageVM.isLoginValid(username:)` posts `command=login` to the Sheet backend. The server
  decides whether the name is valid and returns `data.validLogin` (this check moved to the backend in
  commit `cfaeaaa`; the old client-side `validName` property is no longer used).
  - **Valid:** `checkedLogin = true` is saved to `UserDefaults`, and the tab bar opens.
  - **Invalid:** the About page (`InstructionVC`) opens instead.
- Tapping Confirm with an **empty** field opens the About page directly, which lets anyone see what
  the app is.
- **Auto-login:** a 0.1 s timer checks the saved `checkedLogin` flag on launch and skips the login
  page if the flag is `true`.
- Tapping outside the field or pressing Return dismisses the keyboard.
- Background music for the current theme starts here (`config.setMusic()`).
- A Firebase `select_content` event with `item_id = "login_page"` is logged when the page loads.

### 3.2 Hidden DLC unlock (embargo)

- Long-pressing the login icon for **5 seconds** calls `LoginPageVM.unlockEmbargo()`, which sets
  `config.embargo = false` and saves `embargo = false` to `UserDefaults`.
  - First time: an alert says "1/27 DLC unlocked – the update made for Mrs. Wu has been installed".
  - Later times: an alert tells the user to pick the **Minion (小小兵)** theme.
- While the embargo is on (the default), the theme picker leaves out the Minion theme and the Options
  list leaves out the Letter row.
- A date gate (unlock only after 2021/01/14) is still in `longPressed`, but it is commented out.

### 3.3 Tickets list (`ListPageVC`, `ListPageVM`, `ListPageM`)

The main tab. A segmented control switches between:

| Segment | Data source | Can edit? |
| --- | --- | --- |
| 0 – 我的兌換券 (My tickets) | `config.currentUser` | Yes |
| 1 – 寶貝的兌換券 (Partner's tickets) | `config.objectUser` | Read-only |

Each list has up to two table sections, each with a colored header that shows a count:

- **Section 0: Upcoming** (`UpcomingTicket`). Tickets not yet redeemed.
- **Section 1: Redeemed** (`PostTicket`). Tickets already used. This section is shown only when it is
  not empty.

Both ticket types subclass `BaseTicket` (`name`, `date`, `id`, `index`). `index` is the row the
ticket is drawn in.

#### Fetching (`getTicketDataFromSheet`)

- Runs every time the page appears, on pull-to-refresh, and when the segment changes.
- Sends `command=getTickets, user=<name>`. The response holds `quota` (the maximum number of tickets
  this user may create) and the user's `tickets`.
- Before filling the lists, the old ones are cleared (`removeTicktList`). This fixes duplicate rows
  on slow connections (commit `87f8174`).
- Tickets with `ticketDeleted == true` are skipped. `ticketChecked` decides whether a ticket goes to
  the Redeemed or the Upcoming list. Dates are cut down to `YYYY-MM-DD`.

#### Quota

- `remaining = maxTicketNum − upcoming.count − redeemed.count` (`ListPageM.setQuota()`).
- While `remaining > 0`, the label shows "還有 N 張可以填寫唷" ("N more tickets left to write"), and
  the first row of *My tickets* is a **new-ticket cell** (`NewTicketTVC`). When `remaining` reaches 0,
  the label and the new-ticket cell are hidden.

#### Creating a ticket

1. Type a name into the new-ticket cell and tap its confirm button.
2. An action sheet warns that a ticket cannot be edited once it is added.
3. `postNewTicketToSheet` sends `command=postNewTicket, ticketName, user`. The returned `id` and
   `ticketDate` are used to insert the new ticket at the top of Upcoming.

#### Redeeming ("checking") a ticket

1. Tap the check button on an upcoming ticket in *My tickets*.
2. Confirm in the dialog ("A ticket can only be used once").
3. `checkTicketToSheet` sends `command=checkTicket, id, user`. The ticket moves from Upcoming to the
   top of Redeemed.

#### Deleting a ticket

- Swipe-to-delete on an **upcoming** ticket in *My tickets* calls `deleteTicketToSheet`
  (`command=deleteTicket`). This is a soft delete: the backend marks the ticket deleted, and it is
  filtered out on the next fetch.
- Swiping the new-ticket cell, a **redeemed** ticket, or any of the partner's tickets is blocked with
  a playful alert ("These are our memories, how could you?").

#### Details and errors

- Tapping a row shows the ticket's creation date and status (waiting to be redeemed, or already
  redeemed).
- `connectionError` values from `Config` drive the error handling:

  | Constant | Value | Meaning | UI |
  | --- | --- | --- | --- |
  | `NO_ERROR` | 0 | Success | None |
  | `ERROR_NO_DATA` | 1 | Backend `status != 200` | Error alert |
  | `ERROR_NO_CONNECTION` | 2 | Request failed or returned empty | Error alert |
  | `WAITING_FOR_CONNECTION` | 3 | Reserved | Logged only |

### 3.4 Options (`OptionVC`, `OptionVM`)

A table of option rows. The **Letter** row appears only when the DLC is unlocked **and** the Minion
theme is active. The other rows are always shown.

- **Change theme:** opens a `UIPickerView` in an action sheet (see 3.5).
- **About:** opens `InstructionVC`.
- **Log out:** after a confirmation, saves `checkedLogin = false` and dismisses back to the login
  page.

The icon at the bottom changes with the theme: a panda, a Christmas tree, or a Minion image.

### 3.5 Themes (`Config`, `Utility/Color.swift`)

The `Style` enum is saved to `UserDefaults` under the key `style`. The app defaults to `defaultStyle`
the first time it runs. Each theme has a `BaseStyleColor` palette (`mainColor`, `secondColor`,
`backgroundColor`, `titleColor`, `infoTextColor`, `btnTextColor`):

| Theme | Picker name | Main / background | Music (looped) | Notes |
| --- | --- | --- | --- | --- |
| `defaultStyle` | 寶貝熊貓 (Baby Panda) | `#5856D6` on white | *October – time to love* | Panda icons |
| `xmasStyle` | 聖誕夜 (Christmas Eve) | `#BF0436` on black | *That's Christmas to Me* (Pentatonix) | Snowfall, light status bar, wreath/tree/"hohoho" icons |
| `minionStyle` | 小小兵 (Minion) | `#F2CD5E` on white | *Minions – Banana* | Shown only after the DLC unlock; Minion images; enables the Letter |

When a theme is picked, `config.setStyle()` saves it and rebuilds the palette, the screens restyle
themselves (`setStyle()` / `currentStyle` observers), the status bar is refreshed, and
`config.setMusic()` switches the music.

### 3.6 Snowfall effect (`BaseVC`)

Shown only in the Christmas theme, on every screen that subclasses `BaseVC`:

- A 0.1 s timer (added to the `.common` run loop mode, so snow keeps falling while you scroll) adds a
  new flake about 30% of the time.
- Each flake is a `CAShapeLayer` circle, 0–10 pt in size. Bigger flakes are brighter (lighter grey).
  Flakes are inserted at layer index 0–2, so they fall behind the table view (table views and cells
  have clear backgrounds).
- **The drift direction depends on the date:** on even days the snow drifts right, and on odd days it
  drifts left. The spawn area is widened 100 pt on the matching side.
- Flakes that fall off the bottom of the screen are removed. All snow is cleared when you leave the
  Christmas theme or the screen disappears.

### 3.7 About / Info page (`InstructionVC`, `InstructionVM`)

- Fetches the text with `command=getInfoText` and shows it (or "no info" if there is none).
- The look follows the theme: a bamboo icon, a "hohoho" icon, or a full-screen Minion background.

### 3.8 Secret letter (`BirthdayLetterVC`, `BirthdayLetterVM`)

- Available only from Options when the DLC is unlocked and the Minion theme is active.
- Fetches `command=getBirthdayLetter` and shows `header`, `body`, `footer` and `create_at` (date
  only).
- The body is drawn as underlined, justified text with 10 pt line spacing, in the theme's
  `infoTextColor`.
- The fetched values are kept in the view controller, so a later restyle does not wipe them (commit
  `a0486d1`).

### 3.9 Localization

The UI strings are written in Traditional Chinese in code and wrapped with `.localized`
(`String` extension in `BaseVC.swift`). `en.lproj/Localizable.strings` provides the English text.
The storyboards are also localized (`zh-Hant.lproj/Main.strings`, `LaunchScreen.strings`).

### 3.10 Analytics

Firebase is set up at launch from `GoogleService-Info.plist`. Right now the app logs only the login
page view event. It is meant to collect usage data for future updates.

---

## 4. Backend API

### 4.1 Google Sheet API (current)

All calls are `POST` to the Apps Script URL (`NetworkController.postToSheet`), with form-encoded
parameters. Responses have the shape `{ "status": 200, "data": { ... } }`.

| `command` | Extra params | `data` fields used |
| --- | --- | --- |
| `login` | `username` | `validLogin: Bool` |
| `getTickets` | `user` | `quota: String`, `tickets: [{ id, ticketName, ticketDate, ticketChecked, ticketDeleted }]` |
| `postNewTicket` | `ticketName`, `user` | `id`, `ticketDate` |
| `checkTicket` | `id`, `user` | `ticketName`, `ticketDate` |
| `deleteTicket` | `id`, `user` | (only `status` is checked) |
| `getInfoText` | – | `infoText` |
| `getBirthdayLetter` | – | `header`, `body`, `footer`, `create_at` |

The Apps Script source is not in this repository.

### 4.2 Legacy Express + MySQL server (`backend/index.js`)

This was the original backend (see `demonstration/ER Graph.png` for the schema: `users`, `tickets`,
`articles`). The iOS app no longer calls it (commit `05d6bd7` moved every endpoint to the Google
Sheet), but the code is kept, and the matching Swift calls are kept as commented-out code in each
ViewModel.

| Method & path | Purpose |
| --- | --- |
| `POST /getQuota` | Quota for `user` |
| `POST /getTickets` | `{ quota, tickets[] }` for `user`, ordered by `create_at` |
| `POST /postNewTicket` | Insert a ticket and return it |
| `POST /checkTicket` | Set `checked = true` |
| `POST /deleteTicket` | Soft delete (`deleted = true`) |
| `GET /getInfoText/:user` | The `articles` row with `title = "info"` |
| `GET /getBirthdayLetter/:user` | The `articles` row with `title = "birthdayLetter"` |
| `GET /renewAutoIncrement` | Maintenance: renumbers ticket ids |

To run it: `cd backend && npm install && npm start` (port `$PORT` or 3000).

> ⚠️ The legacy server builds SQL by joining strings together, so it is open to SQL injection, and
> its database credentials are hard-coded in the source. Do not expose it publicly without fixing
> both problems.

---

## 5. Building the app

```bash
pod install                 # SwiftyJSON, Alamofire, Firebase/Analytics
open ticketC.xcworkspace    # always open the workspace, not the .xcodeproj
```

- Set `Config.isTestMode = true` to use the `testA` / `testB` accounts.
- `ticketC/ViewController.swift` and `VC/TVC/UpcomingTVC copy.swift` are old, unused files and are
  not part of the build target.

## 6. Local storage keys (`UserDefaults`)

| Key | Type | Meaning |
| --- | --- | --- |
| `checkedLogin` | Bool | The user is logged in, so the login page is skipped |
| `embargo` | Bool | `false` after the DLC is unlocked (it is `true` when the key is missing) |
| `style` | `Style` | The chosen theme |
| `database` | Bool | Read by `BaseVM.isKeepUsingDatabase()` (legacy, not written anywhere) |
