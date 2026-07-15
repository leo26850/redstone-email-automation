# RED-429 — Lead-Form Auto-Reply Automation (v2 build)

**Redstone Manufacturing · prepared by KAMG · updated 2026-07-15 · client draft due Fri 2026-07-17**

One Google Apps Script project owned by `manuel@redstonemanufacturing.com`. Polls Gmail every 5 minutes for the site's Mailgun lead notifications, auto-replies to enterprise-domain leads as Manuel (CC Eric + Alec), logs the email to HubSpot, and forwards lone replies to Alec.

> **v2 note:** the parser targets the **live redstone-web pipeline** — the site sends its own notification (`src/lib/leads.ts`, `buildEmailText()`): from `no-reply@redstonemanufacturing.com`, to `sales@redstonemanufacturing.com`, subject `New Website Lead - {Company} - {Process}`. **That function is the parser's spec — if it changes, update `parseNotification_`.** The raw notification is never quoted into replies (it contains the internal HubSpot contact URL); the quoted block is re-rendered from parsed fields.

## Requirement map (Eric's 8 points)

| # | Requirement | Implementation |
|---|------------|----------------|
| 1 | Enterprise-domain only | `isEnterpriseDomain_` + 25-domain free-mail blocklist |
| 2 | From Manuel's account | Script owned by manuel@; `GmailApp.sendEmail` is a genuine send |
| 3 | Exact wording + Alec's calendar link | `sendAutoReply_` + `ALEC_CALENDAR_LINK` Script Property |
| 4 | CC Eric and Alec | `CONFIG.cc` |
| 5 | Lone replies → Alec | `forwardLoneReplies_` poller, dedup-tracked |
| 6 | Log in HubSpot | `logToHubSpot_` via the contactId the notification carries |
| 7 | Form submission in body | `buildFormBlock_` re-renders parsed fields (sanitized) |
| 8 | Subject "{Process} with Redstone Manufacturing" | From the "Type of manufacturing process" line |

## Deploy

1. **Confirm sales@ delivers into Manuel's inbox** (group/routing) — hard requirement; replies to Manuel and notifications must land in the same mailbox the script runs on.
2. As manuel@: script.google.com → New project → paste `Code.gs` + `appsscript.json` (enable "Show appsscript.json" in Project Settings).
3. Script Properties: `HUBSPOT_TOKEN` (reuse the site's Vercel token; verify email-engagement write scope), `ALEC_CALENDAR_LINK`.
4. Run `test_parseLatest()` against a real notification — check parsed fields + rendered quote block.
5. Run `setup()` once (OAuth, labels, 5-min trigger).
6. Test: dummy enterprise + gmail + article-form submissions; screenshot; send draft to Eric.

**Open items:** (1) Alec's calendar link · (2) sales@ → Manuel delivery confirmation · (3) article-form auto-reply policy (default OFF via `replyToArticleFormLeads`) · (4) HubSpot token scope check.

## Code.gs (v2, complete)

```javascript
/**
 * Redstone Manufacturing — lead-form auto-reply automation (RED-429) — v2
 *
 * v2 (2026-07-15): reworked against the LIVE redstone-web lead pipeline.
 * The new site (Astro/Vercel) sends its own notification via Mailgun
 * (src/lib/leads.ts) — this script parses THAT format, not the old
 * Gravity Forms email:
 *
 *   From:    Redstone Leads <no-reply@redstonemanufacturing.com>
 *   To:      sales@redstonemanufacturing.com
 *   Subject: New Website Lead - {Company} - {Process}
 *   Text body:
 *     New website lead — quote request via redstonemanufacturing.com
 *     {First Last}
 *     {jobtitle · company}          (line absent on article-form leads)
 *     Email: {lead email}
 *     Phone: {phone}                (optional)
 *     Project details
 *       Current stage of production: …
 *       Type of manufacturing process: …
 *       Main reason for estimate: …
 *       Estimated annual spend: …
 *       Target first shipment: …
 *       Current country of production: …
 *     Describe your part or product:
 *     {freeform}
 *     Attachments (n): …            (optional)
 *     View in HubSpot: …/record/0-1/{contactId}
 *
 * DEPLOYMENT CONSTRAINT: this script must run on the mailbox where BOTH
 * the sales@ notifications AND the lead's replies to Manuel land — i.e.
 * manuel@redstonemanufacturing.com, with sales@ delivering to Manuel
 * (group membership or routing). The reply-forwarder cannot work from
 * any other account.
 *
 * Trigger: time-driven, every 5 minutes → main(). Setup: see README.md;
 * run setup() once after setting Script Properties.
 */

var CONFIG = {
  cc: 'eric@redstonemanufacturing.com, alec@redstonemanufacturing.com',
  forwardTo: 'alec@redstonemanufacturing.com',
  companyDomain: 'redstonemanufacturing.com',

  // Leave '' when the script runs directly on Manuel's account. If it runs
  // on sales@ instead, set to Manuel's address AND configure it as a
  // verified send-as alias — but note the reply-forwarder then breaks
  // (replies land in Manuel's mailbox, not here). Prefer running on Manuel.
  sendAs: '',

  // Article-page closing form (/api/lead) leads have no process field.
  // Policy pending Eric's decision — default OFF (no auto-reply, labeled).
  replyToArticleFormLeads: false,

  // Matches the Mailgun notification exactly (from + subject are fixed
  // constants in redstone-web src/lib/leads.ts).
  notificationQuery: 'from:(no-reply@redstonemanufacturing.com) subject:"New Website Lead" -label:lead-auto/processed -label:lead-auto/error newer_than:3d',

  labels: {
    processed: 'lead-auto/processed',
    skipped: 'lead-auto/skipped',           // free-mail domain
    skippedArticle: 'lead-auto/skipped-articleform',
    sent: 'lead-auto/sent',                 // threads where we sent the auto-reply
    error: 'lead-auto/error',
  },

  freemailDomains: [
    'gmail.com', 'googlemail.com', 'yahoo.com', 'ymail.com', 'hotmail.com',
    'outlook.com', 'live.com', 'msn.com', 'icloud.com', 'me.com', 'mac.com',
    'aol.com', 'proton.me', 'protonmail.com', 'gmx.com', 'gmx.net',
    'zoho.com', 'mail.com', 'yandex.com', 'yandex.ru', 'qq.com', '163.com',
    '126.com', 'comcast.net', 'verizon.net', 'att.net', 'sbcglobal.net',
  ],
};

// Labeled project-detail lines in the notification text body (exact labels
// from redstone-web src/lib/leads.ts buildEmailText).
var PROJECT_LABELS = [
  'Current stage of production',
  'Type of manufacturing process',
  'Main reason for estimate',
  'Estimated annual spend',
  'Target first shipment',
  'Current country of production',
];

// ---------------------------------------------------------------- main

function main() {
  processNewLeads_();
  forwardLoneReplies_();
}

/** One-time setup: creates labels and the 5-minute trigger. */
function setup() {
  Object.keys(CONFIG.labels).forEach(function (k) {
    getOrCreateLabel_(CONFIG.labels[k]);
  });
  ScriptApp.getProjectTriggers().forEach(function (t) {
    if (t.getHandlerFunction() === 'main') ScriptApp.deleteTrigger(t);
  });
  ScriptApp.newTrigger('main').timeBased().everyMinutes(5).create();
  Logger.log('Setup complete: labels created, 5-minute trigger installed.');
}

// ------------------------------------------------- 1. new-lead pipeline

function processNewLeads_() {
  var threads = GmailApp.search(CONFIG.notificationQuery, 0, 20);
  threads.forEach(function (thread) {
    try {
      var msg = thread.getMessages()[0];
      var lead = parseNotification_(msg.getPlainBody());

      if (!lead.email) {
        throw new Error('Could not parse lead email from notification ' + msg.getId());
      }

      if (!lead.process && !CONFIG.replyToArticleFormLeads) {
        thread.addLabel(getOrCreateLabel_(CONFIG.labels.skippedArticle));
        thread.addLabel(getOrCreateLabel_(CONFIG.labels.processed));
        Logger.log('Skipped (article-form lead, no process): ' + lead.email);
        return;
      }

      if (!isEnterpriseDomain_(lead.email)) {
        thread.addLabel(getOrCreateLabel_(CONFIG.labels.skipped));
        thread.addLabel(getOrCreateLabel_(CONFIG.labels.processed));
        Logger.log('Skipped (free-mail): ' + lead.email);
        return;
      }

      var sent = sendAutoReply_(lead, msg.getDate());
      logToHubSpot_(lead, sent.subject, sent.textBody);

      thread.addLabel(getOrCreateLabel_(CONFIG.labels.processed));
      Logger.log('Auto-reply sent to ' + lead.email + ' — "' + sent.subject + '"');
    } catch (e) {
      thread.addLabel(getOrCreateLabel_(CONFIG.labels.error));
      Logger.log('ERROR on thread ' + thread.getId() + ': ' + e.message);
    }
  });
}

function sendAutoReply_(lead, formDate) {
  var subject = (lead.process || 'Manufacturing') + ' with Redstone Manufacturing';
  var firstName = firstName_(lead.name);
  var calendarLink = scriptProp_('ALEC_CALENDAR_LINK') || '';

  // Rebuilt from parsed fields — NEVER quote the raw notification: it
  // contains the internal HubSpot contact URL and ops footer.
  var formBlock = buildFormBlock_(lead);
  var dateStr = Utilities.formatDate(formDate, Session.getScriptTimeZone(), "EEE, MMM d, yyyy 'at' h:mm a");

  var textBody =
    'Hi ' + firstName + ',\n\n' +
    "Thanks for reaching out. I'd like to schedule a time to chat to learn more.\n\n" +
    'When would be a good time to connect with you over a video call?\n\n' +
    (calendarLink ? 'You can also book a time directly here: ' + calendarLink + '\n\n' : '') +
    'Manuel Ayala | Account Executive\n' +
    'Redstone Manufacturing\n' +
    'Direct: 330-632-3292\n' +
    'redstonemanufacturing.com\n\n' +
    'On ' + dateStr + ':\n\n' +
    quoteBlock_(formBlock);

  var htmlBody =
    '<p>Hi ' + escapeHtml_(firstName) + ',</p>' +
    "<p>Thanks for reaching out. I'd like to schedule a time to chat to learn more.</p>" +
    '<p>When would be a good time to connect with you over a video call?</p>' +
    (calendarLink
      ? '<p>You can also book a time directly here: <a href="' + calendarLink + '">' + calendarLink + '</a></p>'
      : '') +
    '<p><b>Manuel Ayala</b> | Account Executive<br>' +
    'Redstone Manufacturing<br>' +
    'Direct: 330-632-3292<br>' +
    '<a href="https://redstonemanufacturing.com">redstonemanufacturing.com</a></p>' +
    '<p>On ' + escapeHtml_(dateStr) + ':</p>' +
    '<blockquote style="margin:0 0 0 .8ex;border-left:1px solid #ccc;padding-left:1ex;color:#555;">' +
    escapeHtml_(formBlock).replace(/\n/g, '<br>') +
    '</blockquote>';

  var options = { cc: CONFIG.cc, htmlBody: htmlBody, name: 'Manuel Ayala' };
  if (CONFIG.sendAs) options.from = CONFIG.sendAs; // requires verified alias
  GmailApp.sendEmail(lead.email, subject, textBody, options);

  // Label the sent thread so the reply-forwarder can watch it.
  Utilities.sleep(2000);
  var sentThreads = GmailApp.search('in:sent to:' + lead.email + ' subject:"' + subject + '"', 0, 1);
  if (sentThreads.length) sentThreads[0].addLabel(getOrCreateLabel_(CONFIG.labels.sent));

  return { subject: subject, textBody: textBody };
}

/** The lead's submission, re-rendered like Manuel's manual quote block. */
function buildFormBlock_(lead) {
  var lines = ['Tell us about your project'];
  PROJECT_LABELS.forEach(function (label) {
    if (lead.fields[label]) lines.push(label + ': ' + lead.fields[label]);
  });
  if (lead.description) {
    lines.push('Describe your part or product:');
    lines.push(lead.description);
  }
  lines.push('');
  lines.push('Contact Information');
  if (lead.name) lines.push('Name: ' + lead.name);
  if (lead.company) lines.push('Company: ' + lead.company);
  lines.push('Work Email: ' + lead.email);
  if (lead.phone) lines.push('Phone Number: ' + lead.phone);
  return lines.join('\n');
}

// --------------------------------------------- 2. reply-forward watcher

/**
 * For every auto-reply thread, forward to Alec any message from the lead
 * where Alec is not on To/Cc (i.e. the lead replied only to Manuel).
 */
function forwardLoneReplies_() {
  var threads = GmailApp.search('label:' + CONFIG.labels.sent + ' newer_than:60d', 0, 50);
  var store = PropertiesService.getScriptProperties();
  var forwarded = JSON.parse(store.getProperty('FORWARDED_IDS') || '[]');

  threads.forEach(function (thread) {
    thread.getMessages().forEach(function (msg) {
      var id = msg.getId();
      if (forwarded.indexOf(id) !== -1) return;

      var from = (msg.getFrom() || '').toLowerCase();
      if (from.indexOf('@' + CONFIG.companyDomain) !== -1) return; // our own mail

      var recipients = ((msg.getTo() || '') + ',' + (msg.getCc() || '')).toLowerCase();
      if (recipients.indexOf(CONFIG.forwardTo.toLowerCase()) !== -1) return; // Alec already on it

      msg.forward(CONFIG.forwardTo, {
        subject: 'Fwd: ' + msg.getSubject(),
      });
      forwarded.push(id);
      Logger.log('Forwarded lone reply ' + id + ' to ' + CONFIG.forwardTo);
    });
  });

  if (forwarded.length > 2000) forwarded = forwarded.slice(-1000);
  store.setProperty('FORWARDED_IDS', JSON.stringify(forwarded));
}

// ------------------------------------------------------ 3. HubSpot log

/**
 * Logs the sent email as an Email engagement on the HubSpot contact.
 * The notification includes the contact id (the site already upserted the
 * contact), so normally no search is needed. Requires Script Property
 * HUBSPOT_TOKEN — the same private-app token redstone-web uses on Vercel.
 */
function logToHubSpot_(lead, subject, textBody) {
  var token = scriptProp_('HUBSPOT_TOKEN');
  if (!token) {
    Logger.log('HUBSPOT_TOKEN not set — skipping HubSpot logging');
    return;
  }

  var contactId = lead.contactId || findContactByEmail_(lead.email, token);
  if (!contactId) {
    Logger.log('No HubSpot contact found for ' + lead.email + ' — skipping log');
    return;
  }

  var payload = {
    properties: {
      hs_timestamp: new Date().toISOString(),
      hs_email_direction: 'EMAIL',
      hs_email_status: 'SENT',
      hs_email_subject: subject,
      hs_email_text: textBody,
    },
    associations: [{
      to: { id: contactId },
      types: [{ associationCategory: 'HUBSPOT_DEFINED', associationTypeId: 198 }], // email → contact
    }],
  };

  hubspot_('https://api.hubapi.com/crm/v3/objects/emails', 'post', token, payload);
}

function findContactByEmail_(email, token) {
  var search = hubspot_('https://api.hubapi.com/crm/v3/objects/contacts/search', 'post', token, {
    filterGroups: [{ filters: [{ propertyName: 'email', operator: 'EQ', value: email }] }],
    properties: ['email'],
    limit: 1,
  });
  return (search.results && search.results.length) ? search.results[0].id : null;
}

function hubspot_(url, method, token, payload) {
  var resp = UrlFetchApp.fetch(url, {
    method: method,
    contentType: 'application/json',
    headers: { Authorization: 'Bearer ' + token },
    payload: JSON.stringify(payload),
    muteHttpExceptions: true,
  });
  var code = resp.getResponseCode();
  if (code >= 300) throw new Error('HubSpot ' + code + ' on ' + url + ': ' + resp.getContentText().slice(0, 300));
  return JSON.parse(resp.getContentText() || '{}');
}

// -------------------------------------------------------- parsing utils

/**
 * Parses the Mailgun notification's plain-text body (format is a fixed
 * template in redstone-web src/lib/leads.ts buildEmailText — treat that
 * file as the spec; if it changes, update this parser).
 */
function parseNotification_(body) {
  var t = body.replace(/\r/g, '');

  var email = (t.match(/^Email:\s*([^\s@]+@[^\s@]+\.[^\s@]+)/mi) || [])[1] || null;
  var phone = ((t.match(/^Phone:\s*(.+)$/mi) || [])[1] || '').trim();
  var contactId = (t.match(/View in HubSpot:\s*\S*\/(\d+)\s*$/mi) || [])[1] || null;

  // Name = first non-empty line after the header; the optional next line
  // is "jobtitle · company" (absent on article-form leads).
  var name = '', company = '';
  var lines = t.split('\n');
  for (var i = 0; i < lines.length; i++) {
    if (/^New website lead/i.test(lines[i].trim())) {
      var rest = [];
      for (var j = i + 1; j < lines.length && rest.length < 2; j++) {
        var l = lines[j].trim();
        if (!l) continue;
        if (/^(Email|Phone|Project details):?/i.test(l)) break;
        rest.push(l);
      }
      name = rest[0] || '';
      if (rest[1]) {
        var parts = rest[1].split('·').map(function (p) { return p.trim(); });
        company = parts[parts.length - 1];
      }
      break;
    }
  }

  var fields = {};
  PROJECT_LABELS.forEach(function (label) {
    var m = t.match(new RegExp('^\\s*' + escapeRe_(label) + ':\\s*(.+)$', 'mi'));
    fields[label] = m ? m[1].trim() : '';
  });

  var desc = '';
  var dm = t.match(/Describe your part or product:\s*\n([\s\S]*?)(?=\n\s*Attachments \(|\n\s*View in HubSpot:|$)/i);
  if (dm) desc = dm[1].trim();

  return {
    email: email ? email.toLowerCase() : null,
    phone: phone,
    name: name,
    company: company,
    contactId: contactId,
    process: fields['Type of manufacturing process'] || '',
    fields: fields,
    description: desc,
  };
}

function isEnterpriseDomain_(email) {
  var domain = email.split('@')[1];
  return CONFIG.freemailDomains.indexOf(domain) === -1;
}

function firstName_(name) {
  var first = (name || '').trim().split(/\s+/)[0] || 'there';
  return first.charAt(0).toUpperCase() + first.slice(1);
}

function quoteBlock_(text) {
  return text.split('\n').map(function (l) { return '> ' + l; }).join('\n');
}

function getOrCreateLabel_(name) {
  return GmailApp.getUserLabelByName(name) || GmailApp.createLabel(name);
}

function scriptProp_(key) {
  return PropertiesService.getScriptProperties().getProperty(key);
}

function escapeRe_(s) {
  return s.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}

function escapeHtml_(s) {
  return String(s).replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;').replace(/"/g, '&quot;');
}

// ------------------------------------------------------------- testing

/** Dry run: parse the newest matching notification and log what WOULD be sent. */
function test_parseLatest() {
  var q = CONFIG.notificationQuery.replace(/-label:\S+\s*/g, '');
  var threads = GmailApp.search(q, 0, 1);
  if (!threads.length) { Logger.log('No matching notification email found.'); return; }
  var msg = threads[0].getMessages()[0];
  var lead = parseNotification_(msg.getPlainBody());
  Logger.log('Parsed: ' + JSON.stringify({
    email: lead.email, name: lead.name, company: lead.company,
    phone: lead.phone, contactId: lead.contactId, process: lead.process,
  }, null, 2));
  Logger.log('Fields: ' + JSON.stringify(lead.fields, null, 2));
  Logger.log('Description: ' + lead.description);
  Logger.log('Enterprise: ' + (lead.email ? isEnterpriseDomain_(lead.email) : 'n/a'));
  Logger.log('Article-form lead (would skip): ' + (!lead.process && !CONFIG.replyToArticleFormLeads));
  Logger.log('Subject would be: ' + (lead.process || 'Manufacturing') + ' with Redstone Manufacturing');
  Logger.log('--- form block that would be quoted ---\n' + buildFormBlock_(lead));
}
```

## appsscript.json

```json
{
  "timeZone": "America/Denver",
  "exceptionLogging": "STACKDRIVER",
  "runtimeVersion": "V8",
  "oauthScopes": [
    "https://mail.google.com/",
    "https://www.googleapis.com/auth/script.external_request",
    "https://www.googleapis.com/auth/script.scriptapp"
  ]
}
```
