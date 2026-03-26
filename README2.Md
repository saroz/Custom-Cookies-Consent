# Cookie Consent System

A lightweight, GDPR-aware cookie consent implementation built into this WordPress theme. No plugin required.

---

## How It Works

### Cookies & Storage Used

| Name | Type | Purpose | Expires |
|---|---|---|---|
| `projectx-cookieconsent-status` | Cookie | Whether the user has interacted with the banner (`allow`) | 365 days |
| `projectx-use-analytical` | Cookie | Whether analytical tracking is permitted (`allow` / `deny`) | 365 days |
| `projectx-use-analytical` | localStorage | Temporary analytical preference before the user clicks Accept | Session |

### User Flow

1. On first visit the consent banner fades in (bottom-left corner).
2. **Cookies Setting** button toggles a panel with an "Analytical" checkbox (checked by default).
3. User toggles the checkbox → stored in `localStorage` temporarily.
4. **ACCEPT** button:
   - Reads localStorage to decide analytical preference.
   - Writes both cookies (`projectx-cookieconsent-status` and `projectx-use-analytical`) for 365 days.
   - Hides the banner.
5. Google Analytics (`gtag`) only initialises if `projectx-use-analytical === 'allow'`.

---

## Files Involved

| File | Role |
|---|---|
| `js/js.cookie.js` | [js-cookie](https://github.com/js-cookie/js-cookie) v2 library |
| `header.php` | Loads js.cookie.js; conditionally boots gtag based on the cookie |
| `footer.php` | Renders the consent banner HTML; reads consent message from ACF |
| `js/projectx.js` | `cookieConsent()` function — all banner logic |
| `projectx.css` | All `.projectx-cc*` styles |
| `custom-admin/fields.php` | ACF option field for the consent message text |

---

## Using This in Another Project

### Step 1 — Copy the JS library

Copy `js/js.cookie.js` into your theme's JS folder.

### Step 2 — Enqueue js.cookie.js before everything else

In `header.php`, load the library **before** `wp_head()` and before any analytics script:

```php
<script src="<?php bloginfo('template_url'); ?>/js/js.cookie.js" data-cfasync="false"></script>
```

The `data-cfasync="false"` attribute prevents Cloudflare Rocket Loader from deferring it.

### Step 3 — Add the gtag / analytics guard in header.php

Replace `UA-XXXXXXXX-X` with your GA property ID:

```php
<script async src="https://www.googletagmanager.com/gtag/js?id=UA-XXXXXXXX-X"></script>
<script>
    var allow_gtag = Cookies.get('projectx-use-analytical');
    if (allow_gtag == 'allow' && allow_gtag !== undefined) {
        window.dataLayer = window.dataLayer || [];
        function gtag(){dataLayer.push(arguments);}
        gtag('js', new Date());
        gtag('config', 'UA-XXXXXXXX-X');
    }
</script>
```

> **Note:** The original theme has a typo on this line — `('js', new Date())` is missing `gtag`. Correct it to `gtag('js', new Date())` in your copy.

### Step 4 — Add the banner HTML in footer.php

Paste this before `<?php wp_footer(); ?>`. Replace the message text as needed:

```php
<?php $cookies_consent = get_field('projectx_cookie_consent_field', 'option', false); ?>

<div id="projectx-ccs" class="projectx-cc">
    <div class="container">
        <span class="projectx-cc_message">
            <?php echo $cookies_consent ?: 'This website uses cookies to improve your experience.'; ?>
            <a href="<?php echo esc_url( home_url( '/privacy-policy' ) ); ?>" target="_blank" rel="noopener">read more.</a>
        </span>
        <div class="projectx-cc_analytical">
            <label class="allow-analytical form-check-label">
                <input type="checkbox" name="projectx-analytical" checked>
                <small>Analytical</small>
            </label>
        </div>
        <div class="projectx-cc_compliance">
            <a aria-label="setting cookies" role="button" tabindex="0" class="projectx-cc_btn projectx-cc_deny">Cookies Setting</a>
            <a aria-label="allow cookies" role="button" tabindex="0" class="projectx-cc_btn projectx-cc_allow">ACCEPT</a>
        </div>
    </div>
</div>
```

If you don't use ACF, replace `get_field(...)` with a plain string or a `get_option()` call.

### Step 5 — Copy the JavaScript logic

Copy the `cookieConsent()` function from `js/projectx.js` (lines 127–189) into your theme's JS file. It requires **jQuery** and **js-cookie** to already be loaded.

```js
jQuery(function ($) {
    function cookieConsent() {
        var projectx_cc = $('.projectx-cc');
        var analytical_cc = $('.projectx-cc_analytical');

        // Restore checkbox state from previous visit
        var check_analytical = Cookies.get('projectx-use-analytical');
        if ((check_analytical == 'allow' || check_analytical == undefined) && localStorage.getItem('projectx-use-analytical') !== 'deny') {
            $('input[name="projectx-analytical"]').prop('checked', true).val('allow');
        } else {
            $('input[name="projectx-analytical"]').prop('checked', false).val('deny');
        }

        // Show banner only if user hasn't interacted yet
        var check_cc = Cookies.get('projectx-cookieconsent-status');
        if (check_cc !== '' && check_cc !== undefined) {
            projectx_cc.addClass('projectx-cc_invisible');
        } else {
            projectx_cc.fadeIn(500);
        }

        // Toggle analytical settings panel
        $('.projectx-cc_deny').on('click', function () {
            if (!$(this).hasClass('open')) {
                analytical_cc.toggleClass('open');
            } else {
                analytical_cc.removeClass('open');
            }
        });

        // Track checkbox change in localStorage (before Accept is clicked)
        $(document).on('change', 'input[name="projectx-analytical"]', function () {
            if (this.checked) {
                $(this).prop('checked', true).val('allow');
                localStorage.setItem('projectx-use-analytical', 'allow');
            } else {
                $(this).prop('checked', false).val('deny');
                localStorage.setItem('projectx-use-analytical', 'deny');
            }
        });

        // Accept — write cookies and hide banner
        $('.projectx-cc_allow').on('click', function () {
            Cookies.set('projectx-cookieconsent-status', 'allow', { expires: 365, path: '/' });

            if (localStorage.getItem('projectx-use-analytical') == 'allow' || localStorage.getItem('projectx-use-analytical') == undefined) {
                localStorage.setItem('projectx-use-analytical', 'allow');
                Cookies.set('projectx-use-analytical', 'allow', { expires: 365, path: '/' });
            } else {
                Cookies.set('projectx-use-analytical', 'deny', { expires: 365, path: '/' });
            }
            projectx_cc.addClass('projectx-cc_invisible');
        });
    }
    cookieConsent();
});
```

### Step 6 — Copy the CSS

Copy all `.projectx-cc*` rules from `projectx.css` (around line 12258) into your stylesheet. Key classes:

| Class | Purpose |
|---|---|
| `.projectx-cc` | Banner wrapper — fixed bottom-left, hidden by default (`display:none`) |
| `.projectx-cc_invisible` | Hides banner after user accepts (opacity 0, z-index -1) |
| `.projectx-cc_analytical` | Collapsible settings panel (hidden until toggled) |
| `.projectx-cc_analytical.open` | Expanded settings panel |
| `.projectx-cc_message` | Consent text area |
| `.projectx-cc_compliance` | Button row |
| `.projectx-cc_btn` | Shared button style |
| `.projectx-cc_deny` | "Cookies Setting" button |
| `.projectx-cc_allow` | "ACCEPT" button |

Customise the brand colour (`#DF3022` / `#C9534F`) to match your project.

### Step 7 — ACF option field (optional)

If the new project also uses ACF, register an options page field to make the consent message editable from the WordPress admin:

```php
// In fields.php or a custom ACF registration file
acf_add_local_field(array(
    'key'           => 'field_5f1e719656a14',   // keep same key if reusing export
    'label'         => 'Cookie Consent Field',
    'name'          => 'projectx_cookie_consent_field',
    'type'          => 'textarea',
    'parent'        => 'group_YOUR_OPTIONS_GROUP_KEY',
    'default_value' => 'This website uses cookies to improve your experience.',
));
```

If ACF is not used, hardcode or use `get_option('cookie_consent_text')` instead.

---

## Renaming for a New Project

All cookie names and CSS classes are prefixed with `projectx-`. To avoid conflicts when using this in multiple projects at once, do a find-and-replace:

| Find | Replace with |
|---|---|
| `projectx-cc` (CSS) | `yourtheme-cc` |
| `projectx-cookieconsent-status` | `yourtheme-cookieconsent-status` |
| `projectx-use-analytical` | `yourtheme-use-analytical` |
| `projectx-analytical` (input name) | `yourtheme-analytical` |

---

## Code Audit

Issues found in the current implementation, ordered by severity.

---

### 1. localStorage is unnecessary and creates state mismatch risk

**Severity: Medium**

`localStorage` is used to temporarily hold the checkbox value before the user clicks Accept. The problem:

- `localStorage` **never expires** — it is not "session" storage as described in the table above. It persists until explicitly cleared.
- If a user clears their browser cookies but not localStorage, their old preference silently carries over on the next visit. The banner re-appears (cookie is gone) but the checkbox is pre-set from localStorage, and when they click Accept the old value is used without them realising it.

**Fix:** Drop localStorage entirely. Read the checkbox value directly from the DOM at the moment Accept is clicked.

```js
// Replace the entire localStorage block with this inside the Accept handler:
$('.projectx-cc_allow').on('click', function () {
    var analyticalValue = $('input[name="projectx-analytical"]').is(':checked') ? 'allow' : 'deny';

    Cookies.set('projectx-cookieconsent-status', 'allow', { expires: 365, path: '/' });
    Cookies.set('projectx-use-analytical', analyticalValue, { expires: 365, path: '/' });

    projectx_cc.addClass('projectx-cc_invisible');
});
```

And remove the `$(document).on('change', ...)` localStorage block entirely.

---

### 2. Pre-checked analytics checkbox violates GDPR

**Severity: High**

The checkbox has `checked` as a default attribute:

```html
<input type="checkbox" name="projectx-analytical" checked>
```

Under GDPR Article 7 and Recital 32, **pre-ticked boxes do not constitute valid consent** for non-essential cookies. The user must actively opt in.

**Fix:** Remove the `checked` attribute. Let the user choose.

```html
<input type="checkbox" name="projectx-analytical">
```

---

### 3. Google Analytics script downloads before consent is checked

**Severity: High**

```html
<script async src="https://www.googletagmanager.com/gtag/js?id=UA-XXXXXXXX-X"></script>
```

The `async` tag fires a **network request to Google's servers** regardless of consent. The inline guard only blocks `gtag('config')`, but the GA script file is still fetched. Connecting to a third-party server without consent can be a GDPR violation.

**Fix:** Inject the script dynamically only after consent is confirmed:

```js
// Inside the Accept handler, after setting cookies, add:
if (analyticalValue === 'allow') {
    var s = document.createElement('script');
    s.src = 'https://www.googletagmanager.com/gtag/js?id=UA-XXXXXXXX-X';
    s.async = true;
    document.head.appendChild(s);
    window.dataLayer = window.dataLayer || [];
    function gtag(){ dataLayer.push(arguments); }
    gtag('js', new Date());
    gtag('config', 'UA-XXXXXXXX-X');
}
```

Remove the `<script async src="...gtag...">` tag from `header.php` entirely.

---

### 4. Redundant `undefined` check in gtag guard

**Severity: Low (dead code)**

```js
if (allow_gtag == 'allow' && allow_gtag !== undefined)
```

If `allow_gtag` equals `'allow'`, it is by definition not `undefined`. The second condition is dead code and also uses loose equality (`==`) instead of strict (`===`).

**Fix:**

```js
if (Cookies.get('projectx-use-analytical') === 'allow')
```

---

### 5. "Cookies Setting" toggle button never closes the panel

**Severity: Medium (UX bug)**

```js
$('.projectx-cc_deny').on('click', function () {
    if (!$(this).hasClass('open')) {   // checks the BUTTON
        analytical_cc.toggleClass('open');
    } else {
        analytical_cc.removeClass('open');  // never reached
    }
});
```

The `open` class is added to `analytical_cc` (the panel), but the condition checks `$(this)` (the button). The button never has the `open` class, so the `else` branch never runs — clicking "Cookies Setting" a second time does not close the panel.

**Fix:**

```js
$('.projectx-cc_deny').on('click', function () {
    analytical_cc.toggleClass('open');
});
```

---

### 6. No explicit "Reject All" option

**Severity: Medium (GDPR)**

GDPR requires that **refusing consent must be as easy as giving it**. Currently there is no Reject button — a user who wants to deny all must: open the settings panel, uncheck the box, then click Accept.

**Fix:** Add a Reject button alongside Accept:

```html
<a role="button" tabindex="0" class="projectx-cc_btn projectx-cc_reject">Reject All</a>
<a role="button" tabindex="0" class="projectx-cc_btn projectx-cc_allow">Accept All</a>
```

```js
$('.projectx-cc_reject').on('click', function () {
    Cookies.set('projectx-cookieconsent-status', 'deny', { expires: 365, path: '/' });
    Cookies.set('projectx-use-analytical', 'deny', { expires: 365, path: '/' });
    projectx_cc.addClass('projectx-cc_invisible');
});
```

---

### 7. Dead empty-string check

**Severity: Low (dead code)**

```js
if (check_cc !== '' && check_cc !== undefined)
```

`Cookies.get()` returns `undefined` for a missing cookie — never an empty string `''`. The `!== ''` check is always `true` and can be removed.

**Fix:**

```js
if (check_cc !== undefined)
```

---

### Audit Summary

| # | Issue | Severity | Type |
|---|---|---|---|
| 1 | localStorage creates state mismatch | Medium | Logic |
| 2 | Pre-checked consent checkbox | **High** | GDPR |
| 3 | GA script loads before consent | **High** | GDPR |
| 4 | Redundant undefined check | Low | Dead code |
| 5 | Toggle button never closes panel | Medium | UX bug |
| 6 | No Reject All button | Medium | GDPR |
| 7 | Dead empty-string check | Low | Dead code |

---

## Dependencies

- [js-cookie](https://github.com/js-cookie/js-cookie) v2 (included as `js/js.cookie.js`)
- jQuery (already loaded by WordPress)
- Advanced Custom Fields (optional — only needed for the editable consent message)
