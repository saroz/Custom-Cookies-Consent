# How to Build a Cookie Consent Banner in WordPress Without a Plugin (Complete Guide)

Most tutorials either point you to a plugin or show you half the code. This is the complete version — HTML, CSS, JS, and WordPress setup — built from a real production theme, with every GDPR mistake already fixed.

No plugin. No third-party consent service. Total control over the UI.

---

## What You'll Build

A fixed banner that appears in the bottom-left corner on first visit. The user can:
- Accept all cookies
- Reject all cookies
- Open a settings panel and toggle three categories individually:
  - **Analytical** — Google Analytics (GA4)
  - **Marketing** — Meta Pixel, Google Ads
  - **Functional** — Live chat, embedded maps

Each category loads its scripts independently. Nothing fires until the user gives explicit consent for that category.

---

## What You Need

- A custom WordPress theme (child theme works too)
- jQuery (loaded by WordPress by default)
- [js-cookie v2](https://github.com/js-cookie/js-cookie) — one small JS file, no npm needed
- Advanced Custom Fields (optional — only for an editable consent message in wp-admin)

---

## Step 1 — Add js-cookie to Your Theme

Download `js.cookie.js` from the [js-cookie releases page](https://github.com/js-cookie/js-cookie/releases) and place it in your theme's `js/` folder.

Then in `header.php`, load it **before** `wp_head()` and before any other scripts:

```php
<!doctype html>
<html <?php language_attributes(); ?>>
<head>
    <meta charset="<?php bloginfo( 'charset' ); ?>">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <!-- Load js-cookie before anything else -->
    <script src="<?php bloginfo('template_url'); ?>/js/js.cookie.js" data-cfasync="false"></script>

    <?php wp_head(); ?>
</head>
```

The `data-cfasync="false"` attribute prevents Cloudflare Rocket Loader from deferring it — important because the consent check runs immediately on page load.

> Do NOT add the Google Analytics `<script>` tag to your header. We will load it dynamically after consent. This is a key GDPR requirement.

---

## Step 2 — Add the Banner HTML to footer.php

Paste this block just before `<?php wp_footer(); ?>`:

```php
<?php
/**
 * Get the cookie consent message.
 * Priority: ACF options field → WordPress option → hardcoded default.
 * Works whether or not ACF is installed.
 */
if ( function_exists( 'get_field' ) ) {
    // ACF is available — pull from options page field
    $consent_message = get_field( 'projectx_cookie_consent_field', 'option', false );
} else {
    // No ACF — fall back to native WordPress option (saved via Settings → General)
    $consent_message = get_option( 'projectx_cookie_consent_text', '' );
}

// Final fallback — hardcoded default if both return empty
if ( empty( $consent_message ) ) {
    $consent_message = 'This website uses cookies to improve your experience.';
}
?>

<div id="projectx-ccs" class="projectx-cc">
    <!-- .container is a Bootstrap/theme utility class for max-width + padding.
         If your theme doesn't use Bootstrap, replace with a plain <div> and add
         padding via CSS, or remove it entirely. -->
    <div class="container">

        <span class="projectx-cc_message">
            <?php echo wp_kses_post( $consent_message ); ?>
            <?php /* Update /privacy-policy to match your actual privacy policy page slug */ ?>
            <a href="<?php echo esc_url( home_url( '/privacy-policy' ) ); ?>" target="_blank" rel="noopener">Read more.</a>
        </span>

        <div class="projectx-cc_settings-panel">

            <!-- Necessary — always on, no toggle -->
            <div class="projectx-cc_option">
                <div class="projectx-cc_option-header">
                    <small>Necessary</small>
                    <span class="projectx-cc_always-on">Always On</span>
                </div>
                <p>Required for the site to function. Cannot be disabled.</p>
            </div>

            <!-- Analytical — Google Analytics -->
            <label class="projectx-cc_option">
                <div class="projectx-cc_option-header">
                    <!-- No 'checked' attribute — GDPR requires active opt-in -->
                    <input type="checkbox" name="projectx-analytical">
                    <small>Analytical</small>
                </div>
                <p>Google Analytics — helps us understand how visitors use the site.</p>
            </label>

            <!-- Marketing — Meta Pixel, Google Ads -->
            <label class="projectx-cc_option">
                <div class="projectx-cc_option-header">
                    <input type="checkbox" name="projectx-marketing">
                    <small>Marketing</small>
                </div>
                <p>Meta Pixel, Google Ads — used to measure ad performance.</p>
            </label>

            <!-- Functional — live chat, maps -->
            <label class="projectx-cc_option">
                <div class="projectx-cc_option-header">
                    <input type="checkbox" name="projectx-functional">
                    <small>Functional</small>
                </div>
                <p>Live chat, embedded maps — enhanced features that use third-party scripts.</p>
            </label>

        </div>

        <div class="projectx-cc_compliance">
            <a role="button" tabindex="0" class="projectx-cc_btn projectx-cc_reject">Reject All</a>
            <a role="button" tabindex="0" class="projectx-cc_btn projectx-cc_settings">Cookies Setting</a>
            <a role="button" tabindex="0" class="projectx-cc_btn projectx-cc_allow">Accept All</a>
        </div>

    </div>
</div>
```

**Key points:**
- No `checked` on any checkbox — pre-ticked boxes are not valid GDPR consent
- Necessary cookies have no toggle — they are required for the site to work and don't need consent
- "Reject All" sits at the same level as "Accept All" — GDPR requires refusing to be as easy as accepting
- Each category is independent — a user can allow Analytical but block Marketing
- Update `/privacy-policy` to the actual slug of your privacy policy page — this page must exist
- `.container` is a Bootstrap/theme class — replace or remove it if your theme doesn't use Bootstrap

---

## Step 3 — The JavaScript

Create a file `js/cookie-consent.js` in your theme folder and paste the code below into it.

Then register and enqueue it in `functions.php`. The `array('jquery')` dependency ensures WordPress loads jQuery before your script:

```php
// In functions.php
function mytheme_enqueue_scripts() {
    wp_enqueue_script(
        'mytheme-cookie-consent',
        get_template_directory_uri() . '/js/cookie-consent.js',
        array( 'jquery' ),   // depends on jQuery — loads after it
        '1.0.0',
        true                 // load in footer
    );
}
add_action( 'wp_enqueue_scripts', 'mytheme_enqueue_scripts' );
```

> js-cookie is already loaded in `header.php` (Step 1), so it will be available when this script runs.

Now add the cookie consent logic to `js/cookie-consent.js`:

```js
jQuery(function ($) {

    function cookieConsent() {
        var cc           = $('.projectx-cc');
        var settingsPanel = $('.projectx-cc_settings-panel');

        // --- Restore each checkbox from its saved cookie ---
        if (Cookies.get('projectx-use-analytical') === 'allow') {
            $('input[name="projectx-analytical"]').prop('checked', true);
        }
        if (Cookies.get('projectx-use-marketing') === 'allow') {
            $('input[name="projectx-marketing"]').prop('checked', true);
        }
        if (Cookies.get('projectx-use-functional') === 'allow') {
            $('input[name="projectx-functional"]').prop('checked', true);
        }

        // --- Show or hide the banner ---
        if (Cookies.get('projectx-cookieconsent-status') !== undefined) {
            cc.addClass('projectx-cc_invisible');
        } else {
            cc.fadeIn(500);
        }

        // --- Toggle settings panel ---
        $('.projectx-cc_settings').on('click', function () {
            settingsPanel.toggleClass('open');
        });

        // --- Accept All ---
        // Reads each checkbox directly from the DOM — no localStorage needed
        $('.projectx-cc_allow').on('click', function () {
            var analytical = $('input[name="projectx-analytical"]').is(':checked') ? 'allow' : 'deny';
            var marketing  = $('input[name="projectx-marketing"]').is(':checked')  ? 'allow' : 'deny';
            var functional = $('input[name="projectx-functional"]').is(':checked') ? 'allow' : 'deny';

            Cookies.set('projectx-cookieconsent-status', 'allow', { expires: 365, path: '/' });
            Cookies.set('projectx-use-analytical', analytical, { expires: 365, path: '/' });
            Cookies.set('projectx-use-marketing',  marketing,  { expires: 365, path: '/' });
            Cookies.set('projectx-use-functional', functional, { expires: 365, path: '/' });

            if (analytical === 'allow') loadAnalytics();
            if (marketing  === 'allow') loadMarketing();
            if (functional === 'allow') loadFunctional();

            cc.addClass('projectx-cc_invisible');
        });

        // --- Reject All ---
        $('.projectx-cc_reject').on('click', function () {
            Cookies.set('projectx-cookieconsent-status', 'deny', { expires: 365, path: '/' });
            Cookies.set('projectx-use-analytical', 'deny', { expires: 365, path: '/' });
            Cookies.set('projectx-use-marketing',  'deny', { expires: 365, path: '/' });
            Cookies.set('projectx-use-functional', 'deny', { expires: 365, path: '/' });
            cc.addClass('projectx-cc_invisible');
        });
    }

    // --- Analytical: Google Analytics (GA4) ---
    function loadAnalytics() {
        var s = document.createElement('script');
        s.src = 'https://www.googletagmanager.com/gtag/js?id=G-XXXXXXXXXX'; // replace with your ID
        s.async = true;
        document.head.appendChild(s);
        s.onload = function () {
            window.dataLayer = window.dataLayer || [];
            function gtag(){ dataLayer.push(arguments); }
            gtag('js', new Date());
            gtag('config', 'G-XXXXXXXXXX'); // replace with your ID
        };
    }

    // --- Marketing: Meta Pixel ---
    function loadMarketing() {
        // Replace YOUR_PIXEL_ID with your Meta Pixel ID
        !function(f,b,e,v,n,t,s){
            if(f.fbq)return;n=f.fbq=function(){n.callMethod?
            n.callMethod.apply(n,arguments):n.queue.push(arguments)};
            if(!f._fbq)f._fbq=n;n.push=n;n.loaded=!0;n.version='2.0';
            n.queue=[];t=b.createElement(e);t.async=!0;
            t.src=v;s=b.getElementsByTagName(e)[0];
            s.parentNode.insertBefore(t,s)
        }(window,document,'script','https://connect.facebook.net/en_US/fbevents.js');
        fbq('init', 'YOUR_PIXEL_ID');
        fbq('track', 'PageView');
    }

    // --- Functional: live chat, maps, etc. ---
    // This function is a placeholder — fill it with your own third-party scripts.
    // Common examples: Tawk.to (live chat), Intercom, Google Maps JS API.
    // Whatever you add here will ONLY run after the user consents to Functional cookies.
    //
    // Example — Tawk.to live chat:
    // var s1 = document.createElement('script');
    // s1.src = 'https://embed.tawk.to/YOUR_PROPERTY_ID/default';
    // s1.async = true;
    // document.head.appendChild(s1);
    function loadFunctional() {
        // add your functional scripts here
    }

    // --- On every page load ---
    // Re-fire scripts for categories the user already consented to in a previous visit
    if (Cookies.get('projectx-use-analytical') === 'allow') loadAnalytics();
    if (Cookies.get('projectx-use-marketing')  === 'allow') loadMarketing();
    if (Cookies.get('projectx-use-functional') === 'allow') loadFunctional();

    cookieConsent();

});
```

**What this does:**
- Each category has its own cookie and its own load function — fully independent
- Reject All writes `deny` to all three cookies in one click
- On every page load, cookies are checked and scripts fire only for consented categories
- No `localStorage` — state comes from cookies and the DOM only

---

## Step 4 — The CSS

Add these styles to your theme's stylesheet. Customise the colours to match your brand.

```css
/* Banner wrapper — fixed bottom-left, hidden until JS fades it in */
.projectx-cc {
    background-color: #ffffff;
    box-shadow: rgba(12, 12, 12, 0.05) 0 0.25rem 1.25rem;
    padding: 1.875rem 1.25rem;
    position: fixed;
    width: 20rem;
    max-width: calc(100% - 2rem);
    z-index: 9999;
    left: 2rem;
    bottom: 2rem;
    display: none; /* JS controls visibility */
    user-select: none;
}

/* Hidden state after user interacts */
.projectx-cc_invisible {
    opacity: 0;
    z-index: -1;
    pointer-events: none;
}

/* Consent message text */
.projectx-cc_message {
    font-size: 0.875rem;
    color: #4D4D4F;
    margin-bottom: 1.25rem;
    display: block;
}
.projectx-cc_message a {
    color: #DF3022;
}

/* Settings panel — hidden until toggled */
.projectx-cc_settings-panel {
    background-color: #fefefe;
    border: 1px solid #eee;
    padding: 0.75rem;
    margin-bottom: 1.25rem;
    display: none;
}
.projectx-cc_settings-panel.open {
    display: block;
}

/* Each cookie category row */
.projectx-cc_option {
    padding: 0.625rem 0;
    border-bottom: 1px solid #f0f0f0;
    font-size: 0.8125rem;
    color: #4D4D4F;
}
.projectx-cc_option:last-child {
    border-bottom: none;
    padding-bottom: 0;
}
.projectx-cc_option p {
    margin: 0.25rem 0 0;
    font-size: 0.75rem;
    color: #888;
}

/* Row header — label text + checkbox or "Always On" badge */
.projectx-cc_option-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 0.5rem;
    cursor: pointer;
}
.projectx-cc_option-header small {
    font-weight: 600;
    flex: 1;
}
.projectx-cc_option-header input[type="checkbox"] {
    width: 1rem;
    height: 1rem;
    cursor: pointer;
}

/* "Always On" badge for Necessary cookies */
.projectx-cc_always-on {
    font-size: 0.6875rem;
    color: #888;
    font-style: italic;
}

/* Button row */
.projectx-cc_compliance {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 0.5rem;
}

/* Shared button style */
.projectx-cc_btn {
    font-size: 0.75rem;
    height: 2.25rem;
    line-height: 2.25rem;
    padding: 0 1rem;
    display: inline-block;
    border-radius: 0;
    cursor: pointer;
    text-decoration: none;
}
.projectx-cc_btn:focus,
.projectx-cc_btn:hover {
    text-decoration: none;
    outline: 2px solid transparent;
}

/* Reject button */
.projectx-cc_reject {
    color: rgba(12, 12, 12, 0.5);
    padding: 0;
}
.projectx-cc_reject:hover {
    color: rgba(12, 12, 12, 0.8);
}

/* Settings button */
.projectx-cc_settings {
    color: rgba(12, 12, 12, 0.5);
}
.projectx-cc_settings:hover {
    color: rgba(12, 12, 12, 0.8);
}

/* Accept button */
.projectx-cc_allow {
    background-color: #C9534F;
    color: #ffffff;
    text-transform: uppercase;
    font-weight: 600;
}
.projectx-cc_allow:hover {
    background-color: #DF3022;
    color: #ffffff;
}
```

---

## Step 5 — ACF Editable Message (Optional)

If you want the consent message to be editable from wp-admin without touching code, you have two options. **Option B (Local JSON) is recommended** — it's cleaner, requires no PHP registration, and the field group travels with the theme in version control.

---

### Option A — PHP Registration

Register the field directly in `functions.php`. Requires ACF Pro with an Options Page enabled.

```php
// In functions.php or a custom fields file
if ( function_exists('acf_add_local_field') ) {
    acf_add_local_field(array(
        'key'           => 'field_cookie_consent_msg',
        'label'         => 'Cookie Consent Message',
        'name'          => 'projectx_cookie_consent_field',
        'type'          => 'textarea',
        'parent'        => 'group_YOUR_OPTIONS_GROUP_KEY',
        'default_value' => 'This website uses cookies to improve your experience.',
        'rows'          => 3,
    ));
}
```

---

### Option B — ACF Local JSON (Recommended)

ACF can register field groups automatically from `.json` files — no PHP needed. This is ACF's own recommended approach for keeping fields in version control alongside your theme.

**How it works:**
1. Create an `acf-json/` folder in your theme root
2. Drop the JSON file below into it
3. ACF picks it up on the next admin page load — no code required

```
your-theme/
├── acf-json/
│   └── group_cookie_consent.json   ← place this here
├── footer.php
├── functions.php
└── ...
```

**`acf-json/group_cookie_consent.json`**

```json
[
  {
    "key": "group_cookie_consent",
    "title": "Cookie Consent",
    "fields": [
      {
        "key": "field_cookie_consent_msg",
        "label": "Cookie Consent Message",
        "name": "projectx_cookie_consent_field",
        "type": "textarea",
        "instructions": "Short message shown in the cookie consent banner.",
        "required": 0,
        "default_value": "This website uses cookies to improve your experience.",
        "rows": 3,
        "new_lines": "wpautop"
      }
    ],
    "location": [
      [
        {
          "param": "options_page",
          "operator": "==",
          "value": "acf-options"
        }
      ]
    ],
    "active": true
  }
]
```

> **Note:** The `location` value `"acf-options"` assumes you have registered a standard ACF options page. If your options page slug is different, update that value to match.

**Why Local JSON over PHP?**
- No PHP to maintain — field definition lives in one place
- Committed to git — every developer on the project gets the same field structure automatically
- ACF syncs it visually in the admin — you can see if the JSON is out of date with what's in the database
- Works with ACF Free and ACF Pro

---

### Option C — No ACF (WordPress Native)

No plugin needed at all. Register the field using the WordPress Settings API and it appears under **Settings → General** in wp-admin.

**`functions.php`**

```php
add_action( 'admin_init', function () {
    register_setting(
        'general',
        'projectx_cookie_consent_text',
        array(
            'type'              => 'string',
            'sanitize_callback' => 'sanitize_textarea_field',
            'default'           => 'This website uses cookies to improve your experience.',
        )
    );

    add_settings_field(
        'projectx_cookie_consent_text',
        'Cookie Consent Message',
        function () {
            $val = get_option( 'projectx_cookie_consent_text', '' );
            echo '<textarea name="projectx_cookie_consent_text" rows="3" class="large-text">'
                . esc_textarea( $val )
                . '</textarea>';
            echo '<p class="description">Shown in the cookie consent banner at the bottom of every page.</p>';
        },
        'general'
    );
});
```

That's it — no extra plugin, no custom admin page. The field lives at **Settings → General → Cookie Consent Message**.

---

### How the Fallback Chain Works

The `footer.php` code in Step 2 handles all three options automatically — you don't need to change it per project:

```
ACF installed?
  └─ Yes → get_field('projectx_cookie_consent_field', 'option')
  └─ No  → get_option('projectx_cookie_consent_text')
             └─ Empty → 'This website uses cookies to improve your experience.'
```

Whichever setup the project uses, the banner always has a message.

---

## Step 6 — Rename for Your Project

All prefixes are `projectx-`. Do a find-and-replace across your files before going live:

| Find | Replace |
|---|---|
| `projectx-cc` (CSS classes) | `mytheme-cc` |
| `projectx-cookieconsent-status` | `mytheme-cookieconsent-status` |
| `projectx-use-analytical` | `mytheme-use-analytical` |
| `projectx-use-marketing` | `mytheme-use-marketing` |
| `projectx-use-functional` | `mytheme-use-functional` |
| `projectx-analytical` (input name) | `mytheme-analytical` |
| `projectx-marketing` (input name) | `mytheme-marketing` |
| `projectx-functional` (input name) | `mytheme-functional` |
| `projectx_cookie_consent_field` (ACF) | `mytheme_cookie_consent_field` |

This avoids cookie name collisions if the same browser visits multiple sites built from this template.

---

## How the Cookies Work

| Cookie | Value | Meaning |
|---|---|---|
| `projectx-cookieconsent-status` | `allow` | User has interacted with the banner |
| `projectx-cookieconsent-status` | `deny` | User explicitly rejected all |
| `projectx-use-analytical` | `allow` | Load Google Analytics (GA4) |
| `projectx-use-analytical` | `deny` | Block Google Analytics |
| `projectx-use-marketing` | `allow` | Load Meta Pixel / Google Ads |
| `projectx-use-marketing` | `deny` | Block marketing scripts |
| `projectx-use-functional` | `allow` | Load live chat, maps, etc. |
| `projectx-use-functional` | `deny` | Block functional scripts |

All cookies expire after 365 days. If `projectx-cookieconsent-status` is absent, the banner shows again.

---

## Common Mistakes to Avoid

| Mistake | Why It Matters |
|---|---|
| Loading `<script async src="gtag...">` unconditionally | Contacts Google's servers before consent — GDPR violation |
| Pre-ticking the consent checkbox | Not valid consent under GDPR Article 7 |
| Using `localStorage` for consent state | Never expires, creates state mismatch when cookies are cleared |
| No Reject button | Refusing must be as easy as accepting under GDPR |
| Checking `$(this).hasClass('open')` on the button instead of the panel | Classic toggle bug — panel never closes |

---

## Dependencies

- [js-cookie v2](https://github.com/js-cookie/js-cookie) — included as a single JS file, no build step
- jQuery — already loaded by WordPress
- Advanced Custom Fields — optional, only for editable consent message

---

*This implementation is production-tested in a custom WordPress theme. Adapt the prefixes, colours, and message to your project before shipping.*
