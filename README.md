# Quick Log – an offline Android app built from a single HTML file

Quick Log is a small personal data-entry app for four Excel/Google Sheets datasets: **Expenses**, **Training**, **Learning** and **Job applications**. It lets you add rows quickly on your phone, with no internet connection, and then copy them in the exact column order of your spreadsheet.

This document is a full manual. It explains:

1. [What the project is and how the pieces fit together](#1-the-big-picture)
2. [The repository structure](#2-repository-structure)
3. [The app code (`www/index.html`) in detail](#3-the-app-code-wwwindexhtml-in-detail)
4. [How a web page becomes an Android app (Capacitor)](#4-how-a-web-page-becomes-an-android-app-capacitor)
5. [How GitHub Actions builds the APK (`build.yml` line by line)](#5-how-github-actions-builds-the-apk)
6. [Step-by-step: from files to an installed app](#6-step-by-step-from-files-to-an-installed-app)
7. [Building on your own computer instead](#7-building-on-your-own-computer-instead)
8. [Updating the app later](#8-updating-the-app-later)
9. [Using the app day to day](#9-using-the-app-day-to-day)
10. [Customizing: add a field or a new dataset](#10-customizing-add-a-field-or-a-new-dataset)
11. [Where your data lives and how to back it up](#11-where-your-data-lives-and-how-to-back-it-up)
12. [Troubleshooting](#12-troubleshooting)
13. [Limitations and security notes](#13-limitations-and-security-notes)
14. [Glossary](#14-glossary)

---

## 1. The big picture

The project has three layers. Understanding them makes everything else easy.

```
┌──────────────────────────────────────────────┐
│ 1. THE APP                                   │
│    www/index.html                            │
│    HTML + CSS + JavaScript. Forms, autofill, │
│    IDs, local storage, copy-to-Excel.        │
└──────────────────────────────────────────────┘
                     │ is placed inside
                     ▼
┌──────────────────────────────────────────────┐
│ 2. THE ANDROID WRAPPER (Capacitor)           │
│    A native Android project whose only job   │
│    is to open a WebView (a mini browser)     │
│    and load the files from www/.             │
└──────────────────────────────────────────────┘
                     │ is compiled into
                     ▼
┌──────────────────────────────────────────────┐
│ 3. THE APK                                   │
│    One installable file (app-debug.apk).     │
│    Contains the wrapper AND your web page,   │
│    so it works with no internet at all.      │
└──────────────────────────────────────────────┘
```

**Who builds the APK?** Compiling an Android app needs Java, the Android SDK and Gradle. Instead of installing all of that on your computer, this project uses **GitHub Actions**: GitHub rents a temporary Linux computer, runs the build commands written in `build.yml`, and gives you the finished APK to download.

**The full flow in one line:**

> You commit files → GitHub starts a runner → it downloads your files and tools → it generates the Android project → it copies your page into it → Gradle compiles it → the APK is saved as an artifact → you download it and install it on your phone.

---

## 2. Repository structure

```
.github/
  workflows/
    build.yml          ← the build instructions for GitHub Actions
www/
  index.html           ← the entire app
capacitor.config.json  ← app name, app ID, web folder
package.json           ← the tools (Capacitor) to download
README.md              ← this manual
```

Important rules:

- The `.github/workflows/build.yml` path must be **exactly** this, and it must be at the **top level of the repository**. GitHub ignores workflow files anywhere else. If your files live inside a subfolder (for example `quicklog-apk/`), the workflow still has to be in the top-level `.github/workflows/`, and it must be told to work inside the subfolder (see [section 5.4](#54-variant-when-your-files-are-in-a-subfolder)).
- The folder `.github` starts with a dot, so it is a "hidden" folder. The GitHub web uploader often skips hidden folders. The workaround is **Add file → Create new file** and typing `.github/workflows/build.yml` as the file name; typing `/` creates the folders.
- The `android/` folder does **not** appear in the repository. It is generated during the build (section 4.4).

---

## 3. The app code (`www/index.html`) in detail

The whole app is one file with three parts: `<style>` (appearance), HTML (structure) and `<script>` (behavior). There are no libraries and no build step, which is why the file works unchanged inside the Android wrapper.

### 3.1 The `<head>`

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```

Tells the phone's browser to use the real screen width instead of pretending to be a wide desktop page. Without it, everything would look tiny. `viewport-fit=cover` lets the page use the full screen including the notch area, and the CSS keeps content clear of it with `env(safe-area-inset-top)`.

### 3.2 The CSS

The styles are kept short. The key ideas:

- **CSS variables** (`--bg`, `--card`, `--tx`, `--mu`, `--ac`, `--bd`) hold the colors (background, card, text, muted text, accent, border) in one place.
- **Dark mode**: a `@media (prefers-color-scheme: dark)` block redefines those variables. When the phone is in dark mode, the same elements automatically pick up the dark colors.
- **Layout**: `.tabs` is a row of four equal buttons. `.card` is a white (or dark) rounded box. `.row` is a flex row used to put two fields side by side (for example Quantity and Unit).
- **Font size 16px on inputs**: on phones, inputs below 16px make the browser zoom in when you tap them. 16px avoids that.

### 3.3 The HTML skeleton

The body has only a few elements. Everything else is generated by JavaScript:

| Element | Purpose |
|---|---|
| `<div class="tabs" id="tabs">` | Filled with the four tab buttons |
| `<div id="form">` | Filled with the form fields of the current dataset |
| `<button id="save">` | Saves the current form as a new row |
| `<b id="lt">` | Title line: "Expenses – 12 entries (3 not copied yet)" |
| `<input id="base">` | "Last ID number already in my Excel" |
| `<button id="copyNew">`, `<button id="copyAll">` | Copy rows to the clipboard |
| `<textarea id="out">` | Shows the copied text (fallback for manual copy) |
| `<div id="list">` | The most recent entries, each with a ✕ delete button |

### 3.4 The JavaScript

#### Helper functions

```js
const today=()=>new Date().toISOString().slice(0,10);   // "2026-11-06"
const nowT=()=>new Date().toTimeString().slice(0,5);    // "18:40"
const $=id=>document.getElementById(id);                // short for getElementById
```

Dates are stored internally as `YYYY-MM-DD` (what `<input type="date">` uses) and converted to `DD/MM/YYYY` only when copying.

#### The dataset definitions: the heart of the app

All four datasets are described in one object, `D`. The app code does not know anything specific about "expenses" or "training"; it just reads this description and builds everything from it. Example (shortened):

```js
const D = {
  exp: {
    label: 'Expenses',        // tab name
    id: 'Expense ID',         // name of the ID column
    pre: 'EXP-',              // ID prefix
    pad: 4,                   // zero padding → EXP-0001
    auto: 'Item',             // the field used to trigger autofill
    show: ['Date','Item','Total Price (€)'],   // what to display in the list
    f: [ /* the fields, in the same order as your Excel columns */ ]
  },
  tr: {...}, ln: {...}, jb: {...}
};
```

Each field in `f` is an object with these optional properties:

| Property | Meaning |
|---|---|
| `n` | Field name = the Excel column header |
| `t` | Input type: `date`, `time`, `number`, `url`, or `area` (multi-line text). Default is text |
| `d` | A function returning the default value (for example today's date) |
| `o` | A list of options, which makes the field a dropdown |
| `l` | `1` = remember past values and suggest them while typing (a `<datalist>`) |
| `af` | `1` = this field is **auto-filled** from the last entry with the same key |
| `calc` | A function that computes the value from the other fields (read-only) |
| `half` | `1` = show this field side by side with its neighbor |
| `step` | `'any'` on numbers so decimals like 1.196 are accepted |

Two examples:

```js
// Total Price is computed, never typed:
{n:'Total Price (€)', t:'number', step:'any', half:1,
 calc: r => r['Quantity'] && r['Unit Price (€)']
          ? +(r['Quantity'] * r['Unit Price (€)']).toFixed(5) : ''}

// Duration is computed from start and end time; it handles passing midnight:
{n:'Duration (min)', t:'number',
 calc: r => {
   if (!r['Start Time'] || !r['End Time']) return '';
   let a = r['Start Time'].split(':'), b = r['End Time'].split(':');
   let m = (+b[0]*60 + +b[1]) - (+a[0]*60 + +a[1]);
   return m < 0 ? m + 1440 : m;     // +1440 minutes = next day
 }}
```

The order of the `f` array **is the column order** of the copied text, so it must match your spreadsheet.

#### State and storage

```js
let cur='exp', rows={}, sent={}, base={};
const ld=k=>{try{return JSON.parse(localStorage.getItem(k))}catch(e){return null}};
const sv=(k,v)=>{try{localStorage.setItem(k,JSON.stringify(v))}catch(e){}};
for (const k in D) { rows[k]=ld('ql_'+k)||[]; sent[k]=ld('qs_'+k)||0; base[k]=ld('qb_'+k)||0 }
```

- `cur`: which tab is open.
- `rows[k]`: the saved entries of dataset `k`, an array of objects.
- `sent[k]`: how many rows of that dataset were already copied.
- `base[k]`: the "last ID already in my Excel" number.
- `localStorage` is a small key/value store inside the WebView. Values must be text, so everything goes through `JSON.stringify`/`JSON.parse`. The `try/catch` means the app keeps working (just without saving) if storage is unavailable.
- Keys are `ql_exp` (rows), `qs_exp` (copied count), `qb_exp` (base number), and the same for `tr`, `ln`, `jb`.

#### Building the form: `draw()`

`draw()` runs every time you switch tabs or save. It:

1. Draws the tab buttons (`tabs()`), highlighting the current one.
2. Loops over the current dataset's fields and builds an HTML string: a `<select>` if the field has options, a `<textarea>` for `area`, otherwise an `<input>`. Fields flagged `half` are wrapped in a flex `.row`.
3. For fields flagged `l`, it adds a `<datalist>` built from all values previously saved in that field (duplicates removed with `new Set`). That is the "suggest while typing" behavior.
4. Inserts the HTML into `#form`, then attaches event listeners: every input recalculates computed fields, and the key field (`auto`) triggers autofill when it changes.
5. Refreshes the list of recent entries and the "last ID" box.

#### Reading and calculating

```js
function read(){ const r={}; D[cur].f.forEach((f,i)=>r[f.n]=$('f'+i).value); return r }
function recalc(){ const r=read(); D[cur].f.forEach((f,i)=>{ if(f.calc) $('f'+i).value=f.calc(r) }) }
```

Each input gets the id `f0`, `f1`, … by its position. `read()` collects the current values into an object keyed by field name; `recalc()` runs every `calc` function and writes the result back into its read-only input.

#### Autofill: the "lazy" part

```js
function autofill(){
  const d=D[cur], key=$('f'+d.f.findIndex(f=>f.n===d.auto)).value;
  const last=[...rows[cur]].reverse().find(r=>r[d.auto]===key);   // most recent entry with same key
  if(!last) return;
  d.f.forEach((f,i)=>{ if(f.af && f.n!==d.auto && !$('f'+i).value) $('f'+i).value=last[f.n]||'' });
  recalc();
}
```

When you pick "Bananas" in Expenses, the app finds the most recent saved "Bananas" row and copies its category, subcategory, unit, unit price and store into the empty fields. It never overwrites something you already typed.

#### Generating IDs

```js
function nextId(){
  const d=D[cur]; let n=base[cur]||0;
  rows[cur].forEach(r=>{
    const m=parseInt(String(r[d.id]).slice(d.pre.length),10);   // "EXP-0045" → 45
    if(m>n) n=m;
  });
  return d.pre + String(n+1).padStart(d.pad,'0');                // 46 → "EXP-0046"
}
```

The next ID is one more than the **highest** of two things: the number you typed in "Last ID already in my Excel", and the highest ID among the rows saved in the app. This means that if your spreadsheet ends at EXP-0045, the app starts at EXP-0046 instead of colliding with existing rows.

#### Saving

```js
$('save').onclick = () => {
  recalc(); const r = read();
  if (!r[d.auto]) { /* show "Fill in: Item" */ return }
  r[d.id] = nextId();
  rows[cur].push(r);
  sv('ql_'+cur, rows[cur]);
  draw();
};
```

The key field (Item, Activity, Learning Material, Company) is the only required field. The row gets its ID, is added to the array, written to storage, and the form is redrawn (which resets it to defaults).

#### Copying to Excel

```js
function tsv(rs, head){
  // one line per row; ID first, then each field; values separated by TAB
}
```

Spreadsheets split pasted text into columns at **tab characters**, so the rows are produced as tab-separated text (TSV). Dates are converted from `YYYY-MM-DD` to `DD/MM/YYYY` by `fmt()`. Tabs and line breaks inside values (for example in Notes) are replaced by spaces so they cannot break the columns.

- **Copy new rows** copies `rows[cur].slice(sent[cur])`, only the rows not copied before, with no header, then sets `sent[cur]` to the new total. Paste it under the last row of your sheet.
- **Copy all rows** copies everything with a header line, and does not change the "already copied" counter.

Clipboard handling:

```js
try { await navigator.clipboard.writeText(text) }
catch(e) { o.select(); document.execCommand('copy') }   // older fallback
```

The modern clipboard API is tried first; if it is blocked, the text is selected in the visible text box so you can copy it manually.

#### Deleting an entry

The ✕ button removes that row from the array, adjusts the "copied" counter so it never exceeds the number of rows, and saves.

---

## 4. How a web page becomes an Android app (Capacitor)

### 4.1 The idea

An Android app can contain a **WebView**: an embedded browser window with no address bar. If the app's only content is "open this local web page in a full-screen WebView", then your HTML file **is** the app. Capacitor is the open-source framework that creates this Android shell and connects it to your web files.

### 4.2 `package.json`

```json
{
  "name": "quicklog",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "@capacitor/android": "^7.0.0",
    "@capacitor/cli": "^7.0.0",
    "@capacitor/core": "^7.0.0"
  }
}
```

This lists three packages that `npm install` downloads from the npm registry:

- `@capacitor/core`: the runtime that lives inside the app.
- `@capacitor/android`: the native Android template and code.
- `@capacitor/cli`: the command-line tool (`npx cap …`) that generates and syncs the Android project.

The `^7.0.0` means "any 7.x version".

### 4.3 `capacitor.config.json`

```json
{
  "appId": "com.gui.quicklog",
  "appName": "Quick Log",
  "webDir": "www"
}
```

- `appId`: the app's unique identifier. Android uses it to tell apps apart. If you install an APK with the **same** appId over an existing one, Android treats it as an update and keeps the data.
- `appName`: the name shown under the icon on your phone.
- `webDir`: the folder whose contents become the app. Must contain `index.html`.

### 4.4 What the Capacitor commands do

| Command | What it does |
|---|---|
| `npm install` | Downloads the packages in `package.json` into `node_modules/` |
| `npx cap add android` | Generates a complete native Android project in a new `android/` folder from Capacitor's template, using your appId and appName |
| `npx cap sync android` | Copies everything in `www/` into `android/app/src/main/assets/public/` and updates the native project's plugin list |
| `npx cap open android` | (Local use only) Opens the project in Android Studio |

When the finished app starts, the native code creates the WebView and loads `index.html` from the app's own assets, served internally from the address `https://localhost`. No network is used. That's why the app works in airplane mode, and why `localStorage` and the clipboard API behave like on a normal secure web page.

### 4.5 What Gradle does

**Gradle** is Android's build system. When you run `./gradlew assembleDebug`, it:

1. Downloads the Android SDK components and libraries the project needs (first run only).
2. Compiles the Java/Kotlin code of the wrapper.
3. Packages the compiled code, the resources, and your `www` files into a single archive.
4. **Signs** it. A "debug" build is signed with an automatically created test key, which is enough for installing on your own phone but not for publishing on the Play Store.
5. Writes the result to `android/app/build/outputs/apk/debug/app-debug.apk`.

### 4.6 What an APK is

An APK (Android Package) is a zip-like file that Android can install. Inside is the compiled wrapper, the Android manifest (name, permissions), and a copy of your `index.html`. Your app code is inside the file, which is why no internet is needed to run it.

---

## 5. How GitHub Actions builds the APK

### 5.1 Concepts

| Term | Meaning |
|---|---|
| **GitHub Actions** | GitHub's built-in automation system |
| **Workflow** | One automation, defined by one YAML file in `.github/workflows/` |
| **Event / trigger** | What starts a workflow (a commit, a button press, a schedule…) |
| **Job** | A group of steps that run on one machine |
| **Runner** | The temporary virtual machine GitHub rents for a job. It starts empty and is deleted afterwards |
| **Step** | One command (`run:`) or one reusable action (`uses:`) |
| **Action** | A ready-made, shareable step, such as "install Node.js" |
| **Artifact** | A file saved from the runner so you can download it after the job ends |

Because the runner is wiped at the end, anything you want to keep, such as the APK, **must** be uploaded as an artifact.

### 5.2 The workflow file, line by line

```yaml
name: Build APK
```
The name shown in the **Actions** tab.

```yaml
on:
  push:
  workflow_dispatch:
```
The triggers. `push` starts the workflow every time you commit to the repository. `workflow_dispatch` adds a manual **Run workflow** button.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
```
One job called `build`, running on a fresh Ubuntu Linux runner.

```yaml
    steps:
      - uses: actions/checkout@v4
```
Downloads your repository's files onto the runner. Without this, the runner would be empty.

```yaml
      - uses: actions/setup-node@v4
        with:
          node-version: 22
```
Installs Node.js 22, which runs `npm` and the Capacitor CLI.

```yaml
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 21
```
Installs Java 21 (the Temurin build), which Gradle and the Android build tools need.

```yaml
      - run: npm install
      - run: npx cap add android
      - run: npx cap sync android
```
Downloads Capacitor, generates the native Android project, and copies `www/` into it (section 4.4).

```yaml
      - run: cd android && chmod +x gradlew && ./gradlew assembleDebug
```
Enters the generated folder, makes the Gradle launcher executable (`chmod +x`), and compiles the debug APK (section 4.5).

```yaml
      - uses: actions/upload-artifact@v4
        with:
          name: QuickLog-apk
          path: android/app/build/outputs/apk/debug/app-debug.apk
```
Saves the APK as an artifact named `QuickLog-apk` so it appears on the run's page.

### 5.3 The complete file (files at the top of the repository)

```yaml
name: Build APK
on:
  push:
  workflow_dispatch:
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 21
      - run: npm install
      - run: npx cap add android
      - run: npx cap sync android
      - run: cd android && chmod +x gradlew && ./gradlew assembleDebug
      - uses: actions/upload-artifact@v4
        with:
          name: QuickLog-apk
          path: android/app/build/outputs/apk/debug/app-debug.apk
```

### 5.4 Variant: when your files are in a subfolder

If `package.json`, `www/` and the other files are in a subfolder (for example `quicklog-apk/`), keep the workflow at the top-level `.github/workflows/build.yml` and add two things: a `defaults` block so every `run:` command works inside the subfolder, and the subfolder in the artifact path.

```yaml
name: Build APK
on:
  push:
  workflow_dispatch:
jobs:
  build:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: quicklog-apk
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 21
      - run: npm install
      - run: npx cap add android
      - run: npx cap sync android
      - run: cd android && chmod +x gradlew && ./gradlew assembleDebug
      - uses: actions/upload-artifact@v4
        with:
          name: QuickLog-apk
          path: quicklog-apk/android/app/build/outputs/apk/debug/app-debug.apk
```

Note that `defaults.run.working-directory` applies only to `run:` steps; the `path:` of `upload-artifact` is relative to the repository root, so it needs the subfolder name.

---

## 6. Step by step: from files to an installed app

### 6.1 Create the repository
1. Create a free account at github.com.
2. Click **New repository**, give it a name (for example `QuickLogAPP`). Private is fine.

### 6.2 Upload the project files
1. In the repository, click **Add file → Upload files**.
2. Drag in `package.json`, `capacitor.config.json`, `README.md` and the `www` folder. Commit.
3. Check that `www/index.html` exists. If the folder did not upload, use **Add file → Create new file**, type `www/index.html`, paste the app's HTML, and commit.

### 6.3 Create the workflow file
1. **Add file → Create new file**.
2. In the name box type `.github/workflows/build.yml` (typing `/` creates each folder).
3. Paste the YAML from section 5.3 (or 5.4 if your files are in a subfolder).
4. Click **Commit changes**.

### 6.4 Run the build
1. Open the **Actions** tab. **Build APK** should be listed on the left. If you see a notice that workflows are disabled, click **I understand my workflows, go ahead and enable them**.
2. The commit from step 6.3 normally starts a run automatically (yellow circle = running). If not, click **Build APK → Run workflow → Run workflow**.
3. Wait about 4–6 minutes. A green check means success; a red X means failure (see [Troubleshooting](#12-troubleshooting)).
4. Click the run to see its steps. You can open each step to read its log.

### 6.5 Download the APK
1. On the finished run's page, scroll to **Artifacts**.
2. Download **QuickLog-apk**. It's a zip containing `app-debug.apk`.
3. Unzip it.

### 6.6 Install on the phone
1. Move `app-debug.apk` to the phone (USB cable, Google Drive, email, Bluetooth…).
2. Open it from the phone's file manager.
3. Android will ask for permission to install apps from this source ("Install unknown apps"). Allow it for that app only.
4. Tap **Install**. If Play Protect warns about an unrecognized app, choose to install anyway; this is normal for debug APKs not from the Play Store.
5. Open **Quick Log**. It works offline from now on.

---

## 7. Building on your own computer instead

This does the same steps as the workflow, with your machine acting as the runner.

**One-time setup**
1. Install Node.js 22 LTS from nodejs.org.
2. Install Android Studio from developer.android.com/studio and open it once so it downloads the Android SDK. It also includes a suitable JDK.

**Build**
```bash
npm install
npx cap add android
npx cap sync android
npx cap open android      # opens Android Studio
```
In Android Studio wait for the Gradle sync, then **Build → Build Bundle(s) / APK(s) → Build APK(s)** and click **locate**. Or from the command line:

```bash
cd android
./gradlew assembleDebug          # Windows: gradlew.bat assembleDebug
```
The APK is at `android/app/build/outputs/apk/debug/app-debug.apk`.

---

## 8. Updating the app later

1. Edit `www/index.html` (on GitHub, open the file and click the pencil icon, edit, commit).
2. The commit triggers a new build automatically. Download the new APK as in section 6.5.
3. Install it **over** the existing app. Because the `appId` is unchanged, Android treats it as an update and **keeps your saved entries**.

Do **not** uninstall first, because uninstalling deletes the app's stored data (section 11).

---

## 9. Using the app day to day

1. Choose a tab and fill in the form. Typing a previously used item, activity, material, etc. suggests it and auto-fills the related fields.
2. Tap **Save entry**. The ID is assigned automatically.
3. First time only: in each tab, type the last ID number already in your spreadsheet (for example `45` for `EXP-0045`) into **Last ID number already in my Excel**. The next entry will then be 46.
4. When you want the entries in your spreadsheet, tap **Copy new rows for Excel**, open the file, click the first empty cell in column A, and paste.
5. If you also add rows by hand in the spreadsheet, update the "last ID" box so the numbers don't collide.

Notes on pasting:
- Dates are copied as `DD/MM/YYYY`. Check that your spreadsheet reads them as real dates.
- Decimals are copied with a dot (`0.99`). If your spreadsheet uses a comma as the decimal separator (for example Portuguese regional settings), numbers may paste as text; use Find & Replace (`.` → `,`) on that column, or change `fmt()` in the code to output a comma.

---

## 10. Customizing: add a field or a new dataset

All changes happen in the `D` object in `www/index.html`.

**Add a field** (for example "Payment method" to Expenses): add a line to the `f` array **at the position where the column sits in your spreadsheet**:

```js
{n:'Payment Method', o:['Card','Cash','MB Way'], af:1},
```

**Change a dropdown's options**: edit the `o` array.

**Add a whole new dataset** (for example "Sleep"): add a new key to `D`:

```js
sl: {
  label:'Sleep', id:'Sleep ID', pre:'SL', pad:3, auto:'Date',
  show:['Date','Hours'],
  f:[
    {n:'Date', t:'date', d:today},
    {n:'Hours', t:'number', step:'any'},
    {n:'Notes', t:'area'}
  ]
}
```

The new tab, form, ID counter, storage and copy buttons all appear automatically, because the code builds everything from `D`. The storage keys (`ql_sl`, `qs_sl`, `qb_sl`) are created on first use because the startup loop runs over every key of `D`.

After editing, commit the file and download the new APK (section 8).

---

## 11. Where your data lives and how to back it up

- Entries are stored in the WebView's `localStorage`, inside the app's private storage on the phone. They are not sent anywhere.
- They are kept when you install an updated APK with the same `appId`.
- They are **deleted** if you uninstall the app, clear its data in Android settings, or reset the phone.
- **Backup = paste into your spreadsheet.** After each pasting session your data is safe in the file. Use **Copy all rows (with header)** at any time to export everything the app holds.
- The debug APK and a differently signed APK cannot update each other; if you ever switch the way you build, copy your rows out first.

---

## 12. Troubleshooting

| Problem | Cause and fix |
|---|---|
| **Actions tab shows no workflow** | The workflow file is not at `.github/workflows/build.yml` in the top level of the repository. Check the path. |
| **`.github` folder won't upload** | Hidden folder; use **Add file → Create new file** and type the full path. |
| **Run fails at `npm install`: "no package.json"** | Files are in a subfolder; use the variant in section 5.4. |
| **Run fails at `npx cap sync`: web directory not found** | `www/index.html` is missing or misplaced. |
| **Run fails at Gradle with a Java version error** | Keep `java-version: 21` with Capacitor 7. |
| **Artifact missing on the run page** | The build failed before the upload step, or the `path:` is wrong (check the subfolder prefix). |
| **Phone: "App not installed"** | A previously installed copy was signed with a different key. Uninstall it first (this deletes data; copy your rows out first). |
| **Phone: can't open the APK** | Allow "Install unknown apps" for the app you opened the file from. |
| **Copy button does nothing** | The text appears in the box below the buttons; long-press → Select all → Copy. |
| **Pasted numbers or dates are text** | See the notes in section 9 about decimals and date formats. |
| **IDs collide with my spreadsheet** | Set **Last ID number already in my Excel** for that tab. |

To see exactly why a build failed: **Actions → click the failed run → click the red step → read the last lines of the log**.

---

## 13. Limitations and security notes

- The app has no connection to Google Drive or Excel. The link between phone and spreadsheet is the manual copy/paste.
- It is a **debug** build: signed with a test key and intended for personal use. Publishing on Google Play would need a release build with your own signing key.
- Data is not encrypted and is not synced between devices.
- Anyone with access to the unlocked phone and the app can read the entries.
- A private GitHub repository keeps your code private; the APK contains no secrets, only the app code.

---

## 14. Glossary

- **APK**: the installable file format of Android apps.
- **Artifact**: a file saved from a GitHub Actions run so it can be downloaded afterwards.
- **Capacitor**: framework that wraps a web app into a native mobile app.
- **Commit**: saving a change to the repository.
- **Gradle**: Android's build tool that compiles and packages the app.
- **JDK / Java**: required by Gradle and Android tools.
- **localStorage**: small browser storage used by the app to keep entries.
- **Node.js / npm**: JavaScript runtime and its package manager, used to run Capacitor.
- **Runner**: the temporary virtual machine on which a GitHub Actions job runs.
- **TSV**: tab-separated values; text where columns are separated by tab characters, which spreadsheets split into cells when pasted.
- **WebView**: a browser component embedded inside an Android app.
- **Workflow**: an automation defined in a YAML file under `.github/workflows/`.
- **YAML**: the indentation-based text format used for workflow files.
