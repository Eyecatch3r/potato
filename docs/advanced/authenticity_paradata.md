# Authenticity Paradata

Authenticity paradata are measurements of *how* an annotator works on a page, not
*what* the annotator answers. Potato uses them to find signs that a person answered
an item somewhere else, or that a program answered it.

The feature works on every task type: Likert scales, radio buttons, spans, rankings,
media and free text. [Keystroke logging](keystroke_logging.md) covers free-text
fields only.

Survey-paradata tools such as ECSP (Embedded Client Side Paradata, University of
Göttingen) collect the same kind of data. So do the "authenticity checks" of some
crowdsourcing platforms. Potato builds the tracker into the annotation page and
stores the data with the annotation.

!!! warning "Evidence for review, not proof"
    Each signal has an ordinary explanation. People leave a page to answer the
    door. They copy a phrase to look up a word. They work on small screens. A task
    that tells annotators to read outside sources causes paste and away events by
    design. Read the evidence for each item before you act on a flag. Do not reject
    work because of a flag alone.

## Terms

| Term | Meaning |
|---|---|
| Item | One instance that the annotator labels. |
| View | One load of an item in the browser. Each view gets a random `view_id`. |
| Snapshot | The running totals for one view. The browser sends the full snapshot each time. |
| Away | The page is hidden (other tab, minimized window), or the window is visible but another window has the focus. |
| External paste | A paste of text that comes from outside the page. The text is not part of the item, not an earlier copy from this page, and not in a text field of this page. |
| Flag | A rule on the server that fired for an item. Each flag holds the numbers that caused it. |

## Quick start

```yaml
authenticity:
  enabled: true                # default false
  phases: [annotation]         # start only after consent (see below)

# Write authenticity.<fmt> beside the annotations on export.
export_include_authenticity: true
```

The [example project](https://github.com/davidjurgens/potato/tree/master/examples/advanced/authenticity)
contains a full configuration.

## Data collection and data protection

This section is for the researcher who writes the consent form, the privacy notice
or the data protection impact assessment (DPIA). It lists each data item, how the
browser gets it, what goes to the server, and where the server keeps it.

This section is not legal advice. Your data protection officer and your ethics board
decide the legal basis and the consent text.

### Summary

- The feature is **off by default**. You must set `authenticity.enabled: true`.
- The browser sends **counts, durations, maxima and true/false values**. It never
  sends text.
- The browser reads copied and pasted text only to compare it with the page. The
  comparison happens in browser memory. Then the browser discards the text.
- The server links each record to the annotator's **username** and to the **item**.
  The data is therefore personal data about an identifiable person.
- Potato does not reject work or exclude annotators because of a flag. A person
  must read the evidence and decide.

### What the browser measures

The script `potato/static/authenticity_tracker.js` listens to standard browser
events. The table shows each data item, the browser source, and the values that go
to the server.

| Data item | Browser source | Values sent to the server |
|---|---|---|
| Leaving the page | `visibilitychange` event | `hidden_events`, `hidden_ms` |
| Focus on another window | `blur` and `focus` events | `blur_events`, `blur_ms`, `max_away_ms` |
| Time in a page that the task opened (codebook link) | Click on a link that opens a new tab, then an absence that starts within 3 s | `linked_away_events`, `linked_away_ms` |
| Copies of item text | `copy` event and the current selection | `item_copy_events`, `item_copy_chars` |
| Selection or right-click on item text | `selectionchange` and `contextmenu` events | `item_select_events`, `item_context_menu_events` |
| Other copies and cuts | `copy` and `cut` events | `other_copy_events`, `cut_events` |
| Pastes | `paste` event and the pasted text | `paste_events`, `item_paste_events`, `external_paste_events`, `external_paste_words`, `max_external_paste_words` |
| Copy, then away or paste | Time between an item copy and the next absence or paste | `max_away_after_item_copy_ms`, `max_paste_words_after_item_copy`, `copy_away_pairs` |
| Window size | `window.outerWidth`, `screen.availWidth`, `resize` event, `devicePixelRatio` | `min_window_ratio`, `large_screen`, `resize_events`, `zoom_changes` |
| Mouse | `pointermove` and `pointerdown` events | `pointer_moves`, `pointer_distance_px`, `mouse_clicks`, `teleport_clicks` |
| Touch or pen input | `pointerdown` event type | `touch_input` |
| Automation | `navigator.webdriver`, and the `isTrusted` property of pointer and key events | `webdriver`, `untrusted_events` |
| Time on the item | Browser clock | `elapsed_ms` |

Each entry in `copy_away_pairs` holds three numbers for one absence after an item
copy:

1. The time from the copy to the start of the absence, in milliseconds.
2. The length of the absence, in milliseconds.
3. The number of characters in the copy.

The browser keeps at most 20 entries for each view.

### What the browser does not send

| Data | What happens to it |
|---|---|
| Copied text | The browser reads the selection, compares it with the item text, and keeps only the last copy in memory. The browser does not send it. |
| Pasted text | The browser reads the text that the annotator pastes into this page. It compares the text with the item, the last copy and the page's text fields. Then it sends only the word count. |
| The clipboard at other times | The tracker does not use the clipboard API. It sees clipboard text only in a `paste` event, when the annotator pastes into the page. |
| Other tabs, other windows, other applications | The browser API gives no names, addresses or contents. The tracker records only the start and the end of each absence. |
| Keystrokes | This tracker records no keys. A key event counts only when a script, not a person, generated it (`isTrusted = false`). |
| Mouse positions | The browser uses coordinates in memory to calculate distance. It sends a total distance and counts, not coordinates or paths. |
| Screen width in pixels | The browser sends a ratio (window width ÷ screen width) and one true/false value (screen width ≥ 1200 px). |
| IP address, user agent | This module does not store them. Your web server, reverse proxy or hosting platform can log them separately. Check those logs too. |

If `track_pointer: false`, the browser does not register the `pointermove` listener.
Then `pointer_moves`, `pointer_distance_px` and `teleport_clicks` stay at 0.

### Identifiers and metadata that the server adds

The server adds these values to each view:

| Value | Source | Notes |
|---|---|---|
| Username | The login session | Potato rejects a post without a session (HTTP 401). The username can be an email, a platform ID (for example a Prolific ID) or a name. This depends on your login setup. |
| Item ID | The post | During annotation, the server refuses an item that the annotator does not have (HTTP 403). |
| `view_id` | The post | A random string that the browser makes for each view. It does not identify the person or the device. |
| `first_seen`, `last_seen` | Server clock | Unix time, in seconds, of the first and the last post of the view. |
| `phase`, `page` | The annotator's current phase | For example `annotation` or `poststudy`. |
| `server_span_ms` | `last_seen − first_seen` | Calculated on the server. The browser cannot change it. |
| `summary`, `verdict` | Server rules | The merged totals and the flags. The server stores no verdict if `detection.enabled: false`. |

### How the data travels

1. The browser keeps one snapshot for each view.
2. The browser sends the snapshot to `POST /api/track_authenticity` on the Potato
   server. It uses the same origin and the session cookie. No third party gets
   the data.
3. The browser sends a snapshot at these times:
    - 1.5 s and 5.5 s after the page loads.
    - Every 10 s (`flush_interval_ms`) when a value changed.
    - Every 30 s on a quiet page that is visible.
    - When the page becomes hidden or closes (`sendBeacon`).
    - When the annotator moves to a different item.
4. The server deletes unknown fields, changes values to the expected types, and
   limits values to a fixed range. Thus no text can go into storage through this
   route.
5. The server replaces the earlier snapshot of the same view. A repeated post
   therefore does not count twice.

### Where Potato stores the data

| Location | Content | Who can read it |
|---|---|---|
| `<output_annotation_dir>/<username>/user_state.json`, key `instance_id_to_behavioral_data.<item>.authenticity` | All views (with timestamps, phase and page), the summary and the verdict | Persons with access to the server's file system |
| Admin dashboard, Behavioral tab, **Authenticity** panel | Counts and flags for each annotator, the `authenticity_risk` ranking and the evidence for each item | Potato administrators |
| `GET /admin/api/authenticity` and `/admin/api/behavioral_analytics` | The same data as JSON | Potato administrators |
| `authenticity.<fmt>` export | One row for each (user, annotated item), with the summary fields, `verdict_level` and `flags` | The person who runs the export, and everyone who gets the file |

Potato writes the export file only if `export_include_authenticity: true`. The
export does not contain `copy_away_pairs`, the view timestamps, or data from survey,
consent and training pages.

Survey, consent and training pages have no item. Their views go into one shared
bucket, `__phase_page__`. Each view in it carries its phase and page.

The server writes `user_state.json` at these times:

- At the last post of a view (item change, page hidden).
- At a post that changes the flags of the item.
- At the next annotation save.
- When the server stops (Ctrl-C or SIGTERM).
- At other times, at most once each minute.

A crash can lose up to one minute of paradata. It cannot lose annotations.

The server keeps at most 50 views and 200 `copy_away_pairs` entries for each item.

### Retention and erasure

This module has no retention setting and no automatic deletion. The paradata stays
in `user_state.json` as long as the annotations stay there.

To delete the paradata of one annotator and keep the annotations:

1. Stop the server.
2. Open `<output_annotation_dir>/<username>/user_state.json`.
3. Delete the `authenticity` key in each entry of `instance_id_to_behavioral_data`.
4. Delete the `authenticity.<fmt>` exports that contain the annotator.

Plan a retention period in your data management plan. Delete exports that you no
longer need.

### Recording only in some phases

By default the tracker runs on every page, including the consent page. If consent
is the legal basis, set `phases` so that the tracker starts after consent:

```yaml
authenticity:
  enabled: true
  phases: [annotation]           # phase types or names from phases.order
```

- A value is a phase type (`consent`, `prestudy`, `instructions`, `training`,
  `annotation`, `poststudy`) or a phase name from `phases.order`.
- `null` or no key means every page.
- An unknown value stops the server at startup with a list of the valid values.

On a page outside `phases`, the server does not load the script. The server also
ignores a post from such a page and answers HTTP 200. This catches a cached page or
an old tab.

One exception applies. After the annotation phase, a post for one of the annotator's
items still counts as annotation. This post is the final snapshot of the last item.
It arrives after "next" moves the annotator on.

If `phases` does not include `annotation`, the admin panel does not count annotated
items as `paradata_missing`.

!!! note "Other trackers"
    Potato has other trackers with the same `phases` key: `keystroke_logging`,
    `annotation_telemetry` and `interaction_tracking`. The general
    [interaction tracker](behavioral_tracking.md) is **on by default**. A study
    that must record nothing before consent sets `phases` on it too. See
    [The general interaction tracker](#the-general-interaction-tracker).

### Notice to annotators

With `disclose_to_annotators: true` (the default), Potato shows this notice on every
page:

> This task records when you leave this page and whether you copy the item or paste
> text from elsewhere — not what you copy or paste.

- The server renders the notice. It shows also when the browser blocks scripts.
- The notice shows also on pages outside `phases`. Thus the participant can read it
  before consent.
- Use `disclosure_text` to show the text that your ethics approval specifies.
- If you set `disclose_to_annotators: false`, the server logs a warning at startup.

The notice is a reminder. It is not consent. Get consent in your consent phase,
before the tracker starts.

### Points for your consent form and privacy notice

Tell participants these facts in plain words:

- The task records when they leave the page, and for how long.
- The task records when they copy item text and when they paste text, and the
  number of characters or words. It does not record the text.
- The task records the size of the browser window relative to the screen, and
  mouse movement totals (if `track_pointer` is on).
- The task records if the browser reports automation software.
- The study server stores the data with their username.
- A researcher reads flagged items. A flag alone does not cause rejection.
- How long you keep the data, and how they can ask for erasure.

Sample text (change it to fit your study):

> While you work on this task, the web page records some information about how you
> work. It records when you leave the page and for how long. It records when you
> copy text from a task item or paste text into the page. It records how many
> characters or words you copy or paste. It does not record the text that you copy or paste,
> or the keys that you press. It also records the size of your browser window
> compared with your screen and how far your mouse moves. It records whether your
> browser reports automation software. We use this information to check data quality. A
> researcher reviews every case. We do not reject your work only because of this
> information. We keep this information with your answers until [date]. You can ask
> us to delete it at [contact].

### Points for a DPIA

| Topic | Fact |
|---|---|
| Data categories | Behavioural events (timing, counts), device features (window ratio, touch input, automation), pseudonymous or direct identifiers (username) |
| Special categories (GDPR Art. 9) | The tracker does not collect them on purpose. Timing data can show patterns about a person. Treat it as sensitive behavioural data. |
| Profiling and automated decisions (GDPR Art. 22) | The server calculates flags and a risk ranking. Potato takes no action on them. A person decides. |
| Data minimization | No text, no coordinates, no keys, no absolute screen size. `track_pointer: false` stops the mouse data. `phases` limits the pages. |
| Recipients | Only the Potato server and its administrators. No third-party service gets the data. |
| Device access | The script reads browser properties (`navigator.webdriver`, screen and window width). Ask your data protection officer if the ePrivacy rules for access to terminal equipment apply. |
| Security | Same as for the annotations: file-system permissions on the output folder, and admin access to the dashboard. |

## Flags

The server merges all views of an item. Then it applies these rules. The thresholds
stay on the server. The page never gets them.

| Flag | Fires when (default) | Severity |
|---|---|---|
| `automation` | `navigator.webdriver` is true | suspect |
| `copy_then_paste` | The annotator copied the item, then pasted ≥ 10 words of external text | suspect |
| `copy_then_away` | The annotator copied ≥ 40 characters of the item, left the page within 60 s, and stayed away ≥ 10 s | suspect |
| `external_paste` | The annotator pasted ≥ 10 words of external text, with no item copy before it | review |
| `item_copied` | The annotator copied ≥ 40 characters of the item, and the two copy rules above did not fire | review |
| `away_from_task` | The annotator left the page ≥ 3 times, or was away ≥ 60 s and for ≥ 50 % of the time on the item | review |
| `split_screen` | The window was ≤ 60 % of the screen width, on a screen ≥ 1200 px, with no touch input | review |
| `pointer_teleport` | ≥ 3 mouse clicks at a new position with no movement before them | review |
| `synthetic_input` | ≥ 3 pointer or key events with `isTrusted = false` | review |

`copy_then_away` describes a closed-format item: copy the item, ask somewhere else,
come back, click. All three conditions must be true for the same absence.

Browser extensions for keyboard navigation or accessibility can generate untrusted
events. That is why `synthetic_input` has the severity `review`.

Time in a page that the task opened (the codebook link) does not count for
`away_from_task`.

If an item has at least one `suspect` flag, its `verdict_level` is `suspect`. If it
has only `review` flags, the level is `review`. On touch devices the verdict gets a
note, because the pointer rules may not apply.

The admin rollup adds **`paradata_missing`**. It counts the annotated items that have
no authenticity record. Blocked or changed scripts cause this. Items annotated
before you enabled the feature also cause it. Compare it with the date when you
enabled the feature.

### Why these rules

- "A Penny for Your Prompts" (arXiv:2607.00403, 2026) used *copy the item, then
  paste outside text* as the main sign of LLM use. In that study, 30.7 % of
  responses had a single copy or paste. A single event was thus not a reliable
  sign. That is why `item_copied` and `external_paste` have the severity `review`.
- Tab switching alone was a weak sign in the same study (F1 = .545). It has a low
  weight in the risk ranking.
- Mouse-click counts and completion time did not separate LLM users from other
  annotators in that study. Neither is a flag. Potato uses pointer data only for
  the teleport check. A physical mouse cannot change position without movement
  events.
- Potato's own scripts generate only `input`, `change` and `click` events. A
  script-generated `pointerdown`, `mousedown` or `keydown` event thus came from a
  different source.

### Risk ranking

`authenticity_risk` multiplies the share of a user's items with each flag by a
weight, and adds the results:

| Flag | Weight |
|---|---|
| `automation` | 0.30 |
| `copy_then_paste` | 0.25 |
| `copy_then_away` | 0.20 |
| `external_paste` | 0.08 |
| `pointer_teleport` | 0.07 |
| `synthetic_input` | 0.07 |
| `item_copied` | 0.04 |
| `away_from_task` | 0.03 |
| `split_screen` | 0.02 |
| `paradata_missing` | 0.01 |

The ranking tells you which annotator to examine first. It is not a probability.

## Configuration

```yaml
authenticity:
  enabled: true                # default false
  phases: null                 # null = every page
  min_away_ms: 1000            # shorter absences do not count
  flush_interval_ms: 10000
  track_pointer: true          # false removes the pointermove listener
  disclose_to_annotators: true # show a notice on every page
  disclosure_text: null        # null = the default notice
  detection:
    enabled: true              # false = store data, calculate no flags
    thresholds:                # any key from the list below
      away_from_task.min_events: 5
      copy_then_away.min_away_ms: 20000

export_include_authenticity: true
```

Threshold keys:

- `copy_then_paste.min_words`
- `copy_then_away.min_away_ms`, `copy_then_away.min_copy_chars`,
  `copy_then_away.max_gap_ms`
- `external_paste.min_words`
- `item_copied.min_chars`
- `away_from_task.min_events`, `away_from_task.min_ms`,
  `away_from_task.min_fraction`
- `split_screen.max_window_ratio`
- `pointer_teleport.min_clicks`
- `synthetic_input.min_events`

An unknown key stops the server at startup.

### The general interaction tracker

The older [behavioral tracker](behavioral_tracking.md) records clicks, focus,
scrolling and shortcut keys. It is on by default. You can switch it off or show a
notice:

```yaml
interaction_tracking:
  enabled: true                 # false: no click, focus, scroll or key stream
  disclose_to_annotators: false # true shows a notice
  disclosure_text: null
  phases: [annotation, poststudy]
```

With `enabled: false`, the page and the server stop the click, focus, scroll and
key stream. Potato still records these data:

- AI-assistant use and the trail of annotation changes. They show where the saved
  answer came from.
- Item loads, saves, navigation and an `activity` mark with no target or detail.
  With them, Potato still measures the time for each item.

Outside its `phases`, the tracker sends only the item loads, saves, navigation and
the `activity` mark. On consent and survey pages it sends nothing. Potato still saves
the answers on those pages, and the time of each change, because they are the
participant's responses and not paradata. If `phases` does not include
`annotation`, the admin suspicion score shows the low-interaction and no-scroll
rates as unknown.

## Limits

- The browser measures everything. An annotator with technical skill can stop or
  change the script. `paradata_missing` finds the simple form of this. The server
  measures `server_span_ms`, so the browser cannot make it longer.
- A chatbot on a second device (for example a phone) leaves no trace on the page.
  Use these signals together with [attention checks and gold
  items](../workflow/quality_control.md). Tell annotators clearly not to use AI. In
  published platform tests, a plain request decreased AI use by a large amount.
- With `text_as_image` on, the annotator cannot copy the item. The copy rules thus
  never fire. The tracker still records when the annotator leaves the page.
- The pointer rules do not apply on touch devices.

## Ethics

These data are behavioural measurements of identifiable people. Keep
`disclose_to_annotators` on, unless your ethics approval allows silent collection.
Treat the export like other personal data. The guidance in [Keystroke Logging
Ethics](keystroke_logging_ethics.md) applies here too.
