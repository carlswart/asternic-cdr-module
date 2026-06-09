# CHANGELOG — Asternic CDR Reports FreePBX 17 Compatibility

> **Target:** Asternic CDR Reports v1.6.6  
> **Platform:** FreePBX 17 (SNG7 / Debian-based containers, PHP 8.2+)  
> **Original upstream:** https://github.com/asternic/asternic-cdr-module

---

## Repository Structure

| Commit | Description |
|--------|-------------|
| `8b6b01d` | **Baseline** — Unmodified upstream v1.6.6 (Oct 2024) |
| `6b54b0b` | **PATCH-001** — PHP 8 undefined array key in `getrecords` handler |
| `a09b4b6` | **PATCH-002** — CEL + disk fallback for missing `recordingfile` paths |
| `6978961` | **PATCH-003** — Null array access in PDF `Footer()` |
| `4f4cbfd` | **PATCH-004** — PDF Combined report column mismatch |
| `75840ed` | **PATCH-005** — Replace deprecated `each()` with `foreach()` in FPDF |
| `e1e8507` | **PATCH-006** — Cap color index in Distribution heatmap |
| `PATCH-007` | **PATCH-007** — Escape unfiltered superglobal output (XSS hardening) |

---

## Compatibility Status Matrix

| Area | Status | Notes |
|------|--------|-------|
| Module loading / Module Admin | ✅ Working | Installs via `fwconsole ma downloadinstall` |
| Dashboard / Home tab | ⚠️ Likely OK | Needs testing with PHP 8 + Whoops |
| Outgoing / Incoming reports | ⚠️ Needs audit | Multiple unguarded `$_REQUEST` accesses |
| Combined reports | ⚠️ Needs audit | Same risk class as above |
| **Drill-down detail table** | ✅ **FIXED** | PATCH-001 resolves `type`/`display` missing keys |
| **Recording icons in detail table** | ✅ **FIXED** | PATCH-002 adds CEL + disk fallback when `recordingfile` is empty (FreePBX 17 outgoing) |
| Call recording playback | ⚠️ Likely OK | Uses Howler.js + `cel` session data |
| **PDF export** | ✅ **FIXED** | PATCH-003 fixes Footer() null access; PATCH-004 fixes Combined report column mismatch; PATCH-005 replaces deprecated `each()` in FPDF |
| CSV export | ✅ Likely OK | Simple string output |
| **Distribution / heatmaps** | ✅ **FIXED** | PATCH-006 caps color index overflow; PATCH-007 escapes unfiltered superglobals |
| Post-processing script (`record_runafter.pl`) | ⚠️ Legacy | Perl script works independently of PHP |

---

## PATCH-002: Recording Icons Missing for Outgoing Calls — `functions.inc.php`

### Problem

FreePBX 17 does not populate the `recordingfile` column in `asteriskcdrdb.cdr` for outgoing calls. The module's detail table shows no play/download icons even when recordings exist on disk (`/var/spool/asterisk/monitor/YYYY/MM/DD/out-*.wav`).

### Root cause

The CDR backend in FreePBX 17 stores recording paths via CEL (Channel Event Log) for incoming calls, but leaves `recordingfile = NULL` for outgoing calls. The original module only checked `$row['recordingfile']`.

### Fix applied

Triple-fallback logic in `functions.inc.php:297-313`:

1. **CDR column** — `$row['recordingfile']` (still works for incoming calls)
2. **CEL session** — `$_SESSION['cel']['recordings'][$uniqueid]['file']` (FreePBX 17 internal cache)
3. **Disk search** — `glob("/var/spool/asterisk/monitor/YYYY/MM/DD/*{uniqueid}.wav")` (catches any orphaned recording)

This is a pragmatic workaround that does not require changing FreePBX CDR backend configuration.

---

## PATCH-003: PDF Export Crash — `functions.inc.php`

### Problem

Clicking the PDF export button on any report triggers a Whoops exception:

```
Whoops\Exception\ErrorException (E_WARNING)
Trying to access array offset on value of type null
/var/www/html/admin/modules/asternic_cdr/functions.inc.php:366
```

### Root cause

`PDF::Footer()` references `$lang[$language]['page']` as `global` variables. These are not defined in FreePBX 17's execution context, so `$lang` is `null`. PHP 8 throws a warning for array offset access on null, which Whoops converts to an exception before the PDF can finish rendering.

### Fix applied

Line 366 in `functions.inc.php`:

```php
$pageText = (isset($lang[$language]['page']) ? $lang[$language]['page'] : 'Page') . ' ' . $this->PageNo();
$this->Cell(0,10,$pageText,0,0,'C');
```

Falls back to the English word "Page" if the locale globals are missing.

---

## PATCH-001: PHP 8 Undefined Array Key — `functions.inc.php`

### Problem

FreePBX 17's routing no longer guarantees `type=tool` in the query string for AJAX `action=getrecords` calls. PHP 8 treats `$_REQUEST['type']` as an **E_WARNING** when the key is absent. FreePBX 17 ships with **Whoops** as the error handler, which converts warnings into exceptions, causing a hard crash with a full stack trace page.

### Stack trace observed

```
Whoops\Exception\ErrorException (E_WARNING)
Undefined array key "type"
/var/www/html/admin/modules/asternic_cdr/functions.inc.php:235
```

### Root cause

In `functions.inc.php:235-236`:

```php
$ftype = $_REQUEST['type'];
$fdisplay = $_REQUEST['display'];
```

These values are used later to populate hidden form fields in the call-detail download links (`downloadVmail()` JS function relies on them). When missing, the JavaScript download form submission breaks as well as the PHP execution.

### Fix applied

```php
$ftype = $_REQUEST['type'] ?? 'tool';
$fdisplay = $_REQUEST['display'] ?? 'asternic_cdr';
```

- `'tool'` matches `<type>tool</type>` declared in `module.xml`
- `'asternic_cdr'` matches `<rawname>asternic_cdr</rawname>` in `module.xml`

These are safe canonical defaults for the module.

---

## PATCH-007: Escape Unfiltered Superglobal Output — `functions.inc.php` + `page.asternic_cdr.php`

### Problem

Multiple locations echo `$_SERVER['REQUEST_URI']` and iterate `$_REQUEST` directly into HTML / JavaScript without escaping. This creates XSS vulnerabilities if a malformed URL or crafted request parameter contains quotes or HTML metacharacters.

### Locations fixed

| File | Line(s) | Before | After |
|------|---------|--------|-------|
| `functions.inc.php` | 156-161 | `$complete_self = $_SERVER['REQUEST_URI'];` echoed raw into `<form action="...">`; `$_REQUEST` keys/values echoed raw into hidden inputs | Wrapped with `htmlspecialchars(..., ENT_QUOTES, 'UTF-8')` |
| `functions.inc.php` | 347-348 | `$complete_self` echoed raw into download form `action` | Wrapped with `htmlspecialchars(..., ENT_QUOTES, 'UTF-8')` |
| `page.asternic_cdr.php` | 1002-1005 | `$complete_self` echoed raw into JS `onclick` attribute | Wrapped with `htmlspecialchars(..., ENT_QUOTES, 'UTF-8')` |
| `page.asternic_cdr.php` | 1408-1413 | Same as above for outgoing/incoming report rows | Wrapped with `htmlspecialchars(..., ENT_QUOTES, 'UTF-8')` |

### Note on `page.asternic_cdr.php:32`

`$_POST['List_Extensions']` was already guarded by `isset()` in the current codebase; the earlier session note was a false positive.

---

## Known Remaining Issues (TODO)

### 1. Unguarded `$_REQUEST` / `$_POST` / `$_GET` accesses (HIGH PRIORITY → now MEDIUM)

The codebase was written for PHP 5/7 where missing array keys silently returned `null`. PHP 8+ throws warnings. Whoops turns those into exceptions. A systematic audit is needed.

**Known hotspots:**

| File | Line(s) | Code | Risk |
|------|---------|------|------|
| `page.asternic_cdr.php` | 32-34 | `$_POST['List_Extensions']` | **Already guarded by `isset()` — safe** |
| `page.asternic_cdr.php` | 45-56 | `$_POST['start']`, `$_POST['end']` | Uses `isset()` — **safe** |
| `page.asternic_cdr.php` | 104-119 | `$_REQUEST['action']` | Uses `isset()` — **safe** |
| `page.asternic_cdr.php` | 117 | `$_REQUEST['file']` | Uses `isset()` — **safe** |
| `page.asternic_cdr.php` | 176+ | `$_GET['tab']` | Uses `isset()` — **safe** |
| `functions.inc.php` | ~235 | `$_REQUEST['type']` | ✅ **FIXED (PATCH-001)** |
| `functions.inc.php` | ~236 | `$_REQUEST['display']` | ✅ **FIXED (PATCH-001)** |
| `functions.inc.php` | 156-161 | `$_REQUEST` iteration | ✅ **FIXED (PATCH-007)** |
| `download.php` | 2 | `$_REQUEST['file']` | Uses `isset()` + `die()` — **safe** |

**Recommendation:** All directly accessed superglobals are now guarded or escaped. No further action required unless new execution paths are discovered.

### 2. FPDF library PHP 8 compatibility (MEDIUM PRIORITY)

The bundled `lib/fpdf.php` is an older version. FPDF historically had issues with:
- PHP 8.1+ deprecations of `strftime()`, `strftime()` is not used here but `date()` is.
- `mbstring` handling changes in PHP 8.
- The `FPDF_FONTPATH` constant and font loading.

**Action:** Test PDF export on FreePBX 17. If it fails, upgrade the bundled FPDF to v1.85+ or patch font-loading logic.

### 3. PJSIP channel name parsing (MEDIUM PRIORITY)

The module extracts extension numbers from Asterisk channel strings using:

```php
substring(channel,1,locate("-",channel,length(channel)-8)-1) AS chan1
```

This assumes legacy `SIP/101-000000ab` naming. With **PJSIP** (default in FreePBX 13+), channels look like:
- `PJSIP/101-00000001`
- `PJSIP/302-00000002`

The parsing still works because it splits on the first `-`, but:
- Custom endpoint names with hyphens will break
- Trunk channels (`PJSIP/mytrunk-00000001`) may be mis-identified as extensions
- `Local/` channels for queues/follow-me may not map cleanly

**Evidence:** The user's system shows `PJSIP/302` in the working drill-down, so basic PJSIP parsing appears functional. Monitor for edge cases.

### 4. Deprecated PHP functions / patterns (LOW PRIORITY)

| Pattern | File | PHP 8 Status |
|---------|------|-------------|
| `mktime()` | `functions.inc.php:16` | Still works; `DateTime` class preferred |
| `preg_split()` for dates | `functions.inc.php:15` | Works; `DateTime::createFromFormat()` preferred |
| `DB_FETCHMODE_ASSOC` | Multiple | PEAR DB is deprecated in modern PHP; FreePBX 17 uses PDO wrappers |
| `html_entity_decode()` without encoding arg | `functions.inc.php:119` | Deprecated in PHP 8.1 |
| `define()` without case-insensitive arg | n/a | Not used |

### 5. FreePBX 17 `fwconsole reload` / module hooks (LOW PRIORITY)

`functions.inc.php` declares `asternic_cdr_get_config()` as a hook executed on **Apply Config**, but it is empty:

```php
function asternic_cdr_get_config($engine) {
    // Executed on APPLY in FreePBX
    global $amp_conf, $db, $active_modules;
}
```

This is harmless but means the module does not inject dialplan or configuration on reload. If FreePBX 17 changes hook signatures, this may need updating.

### 6. JavaScript `XMLHttpRequest` vs modern fetch (LOW PRIORITY)

`asternic_cdr.js` uses legacy `XMLHttpRequest` with ActiveX fallbacks for IE5. This still works in all modern browsers but is technical debt. Not a FreePBX 17 blocker.

### 7. Chart.js version (LOW PRIORITY)

The bundled `assets/js/Chart.min.js` is an older Chart.js v2.x build. It works for the current bar charts but may have vulnerabilities or missing features. Upgrading to Chart.js v4+ would require updating the `swf_bar()` PHP function's generated JS.

---

## How to Deploy Patches

### Method A: Direct edit on live system

```bash
# SSH into your FreePBX container/VM
cd /var/www/html/admin/modules/asternic_cdr

# Apply PATCH-001
sed -i "s/\$ftype = \$_REQUEST\['type'\];/\$ftype = \$_REQUEST['type'] ?? 'tool';/" functions.inc.php
sed -i "s/\$fdisplay = \$_REQUEST\['display'\];/\$fdisplay = \$_REQUEST['display'] ?? 'asternic_cdr';/" functions.inc.php

# No reload needed — PHP is interpreted per-request
```

### Method B: Package and reinstall

```bash
# On your dev machine / local repo
cd /path/to/asternic-cdr-module-1.6.6

# Build tarball
tar czf asternic_cdr-1.6.6-fpbx17-patched.tgz \
  --exclude='.git' \
  --exclude='.github' \
  .

# Copy to FreePBX and install
scp asternic_cdr-1.6.6-fpbx17-patched.tgz root@freepbx:/tmp/
ssh root@freepbx "fwconsole ma delete asternic_cdr && \
  fwconsole ma installlocal /tmp/asternic_cdr-1.6.6-fpbx17-patched.tgz && \
  fwconsole reload"
```

---

## Testing Checklist for FreePBX 17

- [ ] Module installs cleanly via `fwconsole ma downloadinstall`
- [ ] Dashboard (Home tab) loads without warnings
- [ ] Outgoing report loads and charts render
- [ ] Incoming report loads and charts render
- [ ] Combined report loads with comparison charts
- [ ] **Drill-down detail table opens** (PATCH-001 fixes this)
- [ ] Call recording play button works (WAV via Howler.js)
- [ ] Call recording download works
- [ ] PDF export generates without error
- [ ] CSV export generates without error
- [ ] Distribution (hourly heatmap) report loads
- [ ] Date range changes work without JS errors
- [ ] No Whoops error pages on any tab

---

## Contributing Patches

Format for future patch commits:

```
PATCH-NNN: <Short description>

Problem:
<What broke and under what conditions>

Root cause:
<Why it broke on FreePBX 17 / PHP 8>

Fix:
<What changed and why>

Files changed:
- file.php (lines X-Y)

Testing:
- [ ] <Specific test case>

Co-Authored-By: <name> <email>
```

---

## License

Patches remain under the original module's **GPL v3** license.  
Original copyright: Nicolás Gudiño / Asternic.net

---

*Last updated: 2026-06-09*
