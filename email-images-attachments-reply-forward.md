# Mail Client QA — Images, Attachments, Reply & Forward

Test script for the three areas of the mail client where behaviour depends on
HTML we did not write, on files we did not create, and on how a *different*
mail client renders what we send.

**Scope:** image sources in a received mail · inline (CID) attachment
resolution · attachment handling on receive, create, reply and forward ·
reply and forward body construction.

**Not covered here:** theme switching and dark-theme adaptation of sender
HTML, list/pagination
performance, folder counts, and regression of the shared components the
redesign also changed. Those need their own suites — see
[Gaps](#gaps-not-yet-covered).

**Status:** drafted 2026-09-22 against `wholix-react` branch
`feat/email-client-redesign` (head `eaa1dd90`). Several scenarios are expected
to fail today — see the [known-defect register](#known-defect-register) before
raising anything.

---

## How to run

### Environments

Every push to a branch in `wholix-react` auto-deploys to the dev frontend host.
Run the suite against the **branch deployment**, not production.

Run the whole suite once per mailbox type:

| Axis | Values |
|---|---|
| Mailbox type | IMAP (cPanel or custom) · OAuth Gmail · OAuth Microsoft |

That is the only axis. Provider differences are what this suite exists to
catch; nothing else here changes the outcome.

- **Desktop only.** The mail client is not a phone surface and no scenario
  needs a narrow viewport.
- **Light theme only.** Do not switch theme during this suite. Whether an
  image resolves, whether a forward carries its attachments, and what Outlook
  renders are all theme-independent. Everything to do with the theme — the
  switch itself and how sender HTML is re-coloured for dark mode — belongs to
  a separate suite; see [Gaps](#gaps-not-yet-covered).

A scenario that only applies to one mailbox type says so.

### External recipients

Several scenarios check what the *recipient* sees. Keep three external
addresses ready and check the received mail in the provider's own web UI, not
in Wholix:

- a Gmail address
- an Outlook.com or M365 address
- an address you can open in Outlook **desktop**

Outlook desktop matters on its own: it renders differently from Outlook Web and
is where inline-image problems usually surface first.

### Recording results

| Field | Value |
|---|---|
| Scenario | `IMG-04` |
| Mailbox | IMAP |
| Result | Pass / Fail / **Known** (matches a defect below) / Blocked |
| Observed | What actually happened, in one sentence |
| Evidence | Screenshot, or the network-tab entry |

Record **Known** rather than Fail when the behaviour matches the register —
but still write down what you saw. The register records what we expect to be
broken; QA confirms it is broken in the way we think it is, and nothing worse.

---

## Fixtures

Prepare these once. Scenarios reference them by id. Send each to a Wholix
mailbox from an external client so the mail arrives through the real path.

| Id | Fixture |
|---|---|
| **FX-01** | Newsletter with a remote **HTTPS** image hosted on a domain that is *not* on our allowlist (any brand newsletter will do) |
| **FX-02** | Mail with a plain **`http://`** image |
| **FX-03** | Mail with a **`data:` base64** image in the body |
| **FX-04** | Mail with **CID inline images** — send from Outlook desktop with a screenshot pasted into the body |
| **FX-05** | Mail with **both** inline images and real file attachments |
| **FX-06** | Thread of **8+ messages**, several carrying inline images |
| **FX-07** | Mail with **one** attachment |
| **FX-08** | Mail with **10+** attachments of mixed type (pdf, xlsx, docx, png, zip) |
| **FX-09** | Mail with an attachment whose filename contains **spaces, accents and a comma** |
| **FX-10** | Mail with an attachment **over 20 MB** |
| **FX-11** | Mail containing an **SVG** image, and a second one with an SVG referencing an external resource |
| **FX-12** | Mail whose body contains `<script>`, an `<img onerror=…>`, an `<svg onload=…>`, a `<form>` with an external action, and an `<iframe>` |
| **FX-13** | A signature configured with a **remote logo** on a non-allowlisted host, and a second signature with a **pasted (base64)** logo of ~4 MB |

---

## Known-defect register

Confirmed by code review on 2026-09-22. A scenario tagged with one of these is
expected to fail; record **Known**.

| Id | Defect | Severity | Origin | Code |
|---|---|---|---|---|
| **D-1** | Remote images load unconditionally — every tracking pixel fires on open, and the sender learns the reader's IP. No proxy, no "show images" gate | Major | Pre-existing | `Email/HtmlPreview.tsx:122-129` |
| **D-2** | **Reply and forward quotes lose the original's inline images.** The quote is built from a body where every `cid:` was already replaced by a 1×1 transparent placeholder | Blocker | Pre-existing | `Email/ViewMail.tsx:230-235, 319, 348` |
| **D-3** | Images inserted in the composer are sent as `data:` URIs. Gmail and Outlook strip these, so the recipient sees nothing. The backend supports CID (`attachments_metadata[]`); the frontend never calls it | Blocker | Regressed by `2de92def`, Jan 2026 | `MailComposer.tsx:608-622, 711-735` · `helpers/mail.ts:419` |
| **D-4** | The composer's image allowlist (~15 hosts) is far stricter than the reader's (any HTTPS). An image visible while reading is deleted on reply, each deletion raising a blocking dialog | Major | Pre-existing | `utils/constants/common.ts:178-216` |
| **D-5** | **Forward silently drops files attached in the composer** — `ForwardEmailRequest` has no field for them | Blocker | Pre-existing | `Mail.tsx:2561-2609` · `UnifiedEmailTypes.ts:270-277` |
| **D-6** | **Forward silently drops an edited subject** — same request, no subject field | Major | Pre-existing | as D-5 |
| **D-7** | Attachment preview is unreachable from the reader. `canPreview` defaults false and `ViewMail` never passes it, so every chip downloads | Major | Pre-existing | `Email/AttachmentSection.tsx:19-57` |
| **D-8** | Inline images are listed as attachment chips alongside real files, and the list now defaults to open | Minor | Pre-existing; made visible by the redesign | `Email/AttachmentSection.tsx:27` |
| **D-9** | Plain `http://` images have their `src` stripped, showing as broken | Minor — product decision | Pre-existing | `utils/helpers/common.ts:5-24` |
| **D-10** | Dropped images land at the end of the body, below the signature and the quote, not at the caret. Images over 10 MB are dropped with no message | Minor | New | `MailComposer.tsx:711-735` |
| **D-11** | A signature too large for `localStorage` reports success and silently fails to persist | Major | New | `utils/helpers/signature.ts:32-56` |
| **D-12** | The quoted original shown **inside the composer** is rendered in the main document and never passes through DOMPurify — only Jodit's much thinner default cleanup (strips `<script>` and `onerror`, neutralises `javascript:` links; other event-handler attributes are not addressed) | Needs live confirmation | Pre-existing | `helpers/mail.ts:36-56` — DOMPurify appears only in `HtmlPreview.tsx` |

---

## IMG — Image sources in a received mail

The body renders inside a sandboxed iframe (`srcDoc`, no `allow-same-origin`)
with DOMPurify applied in the parent. These scenarios check which sources
survive that path.

### IMG-01 · Remote HTTPS image loads

- **Preconditions:** FX-01
- **Steps:** Open the mail with the network tab recording.
- **Expected:** Image displays. The request goes directly to the sender's host
  and carries no `Referer` header.
- **Note:** The direct request is **D-1**. Confirm the absence of `Referer`;
  that part is working as intended.

### IMG-02 · Plain `http://` image is stripped

- **Preconditions:** FX-02
- **Steps:** Open the mail.
- **Expected:** Broken-image placeholder; no `src` on the element.
- **Known:** **D-9.** Record how bad it looks — how much of the mail is lost
  decides whether we proxy or allow `http://`.

### IMG-03 · Inline base64 image renders

- **Preconditions:** FX-03
- **Steps:** Open the mail.
- **Expected:** Displays immediately, with no network request for the image.

### IMG-04 · SVG renders without executing

- **Preconditions:** FX-11
- **Steps:** Open both mails.
- **Expected:** The SVG displays. Nothing executes. For the second mail, check
  the network tab: note whether the external reference is fetched — a silent
  fetch here is a tracking bypass of the kind Roundcube was caught by.

### IMG-05 · Links open safely

- **Preconditions:** any mail with a link
- **Steps:** Click a link in the body.
- **Expected:** Opens in a new tab. The element carries
  `rel="noopener noreferrer"` in the frame's DOM.

### IMG-06 · Wheel scrolling is handed back to the page

- **Preconditions:** any mail longer than the reading pane
- **Steps:** Scroll the wheel over the body, then over the header area above it.
- **Expected:** Both scroll the reading pane at the same rate. The frame never
  scrolls independently and there is no nested scrollbar.

### IMG-07 · Quoted history expands and the frame regrows

- **Preconditions:** a mail with a long quoted history
- **Steps:** Click the `•••` toggle, then click it again.
- **Expected:** The quote expands and the frame grows to fit it; collapsing
  returns to the original height. No clipped content at either size.

---

## CID — Inline attachment resolution

Inline images are not in the body when it first renders. The body is shown with
placeholders, the real bytes are downloaded per attachment, converted to base64
data URLs, and pushed into the live frame. These scenarios test that handover.

### CID-01 · Inline images resolve after the body paints

- **Preconditions:** FX-04
- **Steps:** Open the mail and watch.
- **Expected:** Placeholders appear first, then the real images swap in and the
  body reflows to its new height. No permanently blank boxes. The reflow does
  not throw away the reader's scroll position.

### CID-02 · Navigating away and back

- **Preconditions:** FX-04
- **Steps:** Open the mail, click to another mail before the images finish,
  then come back.
- **Expected:** Images are present on return. Check the network tab: the same
  attachments should not be downloaded twice.

### CID-03 · Whole thread resolves on open

- **Preconditions:** FX-06
- **Steps:** Open the thread with the network tab recording. Time how long
  until the expanded message is fully painted.
- **Expected:** The expanded message paints promptly.
- **Watch for:** whether **collapsed** messages are also downloading their
  inline images, and whether the downloads are sequential. Both are true in
  the current code and are the reason a long thread feels slow. Record the
  request count and the elapsed time — this is the measurement that justifies
  the fix.

### CID-04 · Large inline image

- **Preconditions:** a mail carrying a single inline image of 5 MB or more
- **Steps:** Open it. Watch memory in the browser task manager.
- **Expected:** It renders. Note the memory growth — each image is held as a
  base64 string in app state *and* cloned into the frame, so the footprint is
  roughly 2.7× the file size.

---

## ATT — Attachment handling

### ATT-01 · Inline images are listed as attachments

- **Preconditions:** FX-05
- **Steps:** Open the mail. Count the chips below the body.
- **Expected:** Only the real file attachments are listed.
- **Known:** **D-8.** Inline parts appear as chips too, and the section now
  opens by default. Record the count difference.

### ATT-02 · Clicking a chip

- **Preconditions:** FX-08
- **Steps:** Click a PDF chip, then an image chip, then a `.docx` chip.
- **Expected:** Preview for the PDF and the image; download for the `.docx`.
- **Known:** **D-7.** Everything downloads.

### ATT-03 · Download All appears only when it means something

- **Preconditions:** FX-07 and FX-08
- **Steps:** Open each.
- **Expected:** "Download All" is hidden for the single-attachment mail and
  shown for the multi-attachment one. Download All produces every file.

### ATT-04 · Reported size against actual

- **Preconditions:** FX-05
- **Steps:** Note the "at least X" figure, then download everything and compare.
- **Expected:** The figure is at or below the real total.
- **Watch for:** inline parts may report their base64-encoded size, which would
  overstate the total. Record the discrepancy.

### ATT-05 · Filenames with awkward characters

- **Preconditions:** FX-09
- **Steps:** Download the attachment; then attach the same file to a new mail
  and send it to an external address.
- **Expected:** The filename survives intact in both directions.

### ATT-06 · Adding and removing attachments in the composer

- **Preconditions:** none
- **Steps:** Compose a new mail. Attach 10+ files in **two separate batches**.
  Remove three individually, one of them from the first batch.
- **Expected:** Exactly the chosen three disappear. The chip strip scrolls
  rather than pushing the send button off screen.

### ATT-07 · Large attachment

- **Preconditions:** a file over 20 MB
- **Steps:** Attach it to a new mail and send.
- **Expected:** Either it sends, or a clear error arrives **before** the upload.
- **Watch for:** there is no client-side size check and no upload progress.
  Record how long the UI sits with no feedback, and what the failure looks like
  if the server rejects it.

### ATT-08 · Composer resets between uses

- **Preconditions:** none
- **Steps:** Attach a file, close the composer without sending, reopen it.
- **Expected:** The attachment list is empty along with the rest of the form.

### ATT-09 · Receiving what we sent

- **Preconditions:** none
- **Steps:** Send a mail with three attachments to a Gmail address and to an
  Outlook address. Open each in the provider's own UI.
- **Expected:** All three arrive, with correct names, sizes and types.

---

## RPY — Reply and forward

The three scenarios flagged below are the reason this suite exists. Each is a
silent data loss: the user sees the right thing on screen and the recipient
gets something different.

### RPY-01 · Reply quote keeps the original's inline images

> One of the three headline scenarios.

- **Preconditions:** FX-04
- **Steps:** Open the mail, confirm the inline images are visible in the
  reading pane, then press **Reply**. Look at the quoted section inside the
  composer.
- **Expected:** The same images appear in the quote.
- **Known:** **D-2.** They are invisible 1×1 placeholders. The images only ever
  existed inside the reading frame; the quote is built from the pre-resolution
  body.

### RPY-02 · What the recipient receives

> Continuation of RPY-01 — the part that actually matters.

- **Preconditions:** RPY-01 completed
- **Steps:** Send the reply to a Gmail address, an Outlook Web address and an
  Outlook **desktop** address. Open all three in the provider's own client.
  Also open the copy in our own Sent folder.
- **Expected:** The quoted history looks the same as the original did.
- **Known:** **D-2.** Record precisely what each client shows — a gap, a broken
  icon, or nothing at all. Attach screenshots from all three.

### RPY-03 · Forward carries files added in the composer

> One of the three headline scenarios.

- **Preconditions:** FX-07
- **Steps:** Open the mail, press **Forward**, attach one extra file in the
  composer, send to an external address, open it there.
- **Expected:** Both the original attachment and the newly added file arrive.
- **Known:** **D-5.** The added file is dropped with no warning. Confirm the
  original attachment *does* arrive — that part is handled by the backend and
  needs verifying separately on IMAP and on OAuth.

### RPY-04 · Forward keeps an edited subject

> One of the three headline scenarios.

- **Preconditions:** any mail
- **Steps:** Press **Forward**. Change the `Fwd:` subject to something
  distinctive. Send and open at the recipient.
- **Expected:** The edited subject arrives.
- **Known:** **D-6.** The original `Fwd:` subject arrives instead.

### RPY-05 · Forward carries the original's own attachments

- **Preconditions:** FX-08, on an IMAP mailbox **and** on an OAuth mailbox
- **Steps:** Forward the mail from each mailbox type. Open at the recipient.
- **Expected:** All original attachments arrive in both cases.
- **Note:** Forward is server-side — the client sends only the message id and
  the typed text. A failure here is a backend gap, not a UI one. Raise it
  against the API, and say which mailbox type it failed on.

### RPY-06 · Reply does not re-attach the original's files

- **Preconditions:** FX-08
- **Steps:** Reply to the mail and send.
- **Expected:** The reply carries no attachments. This is correct behaviour —
  the scenario exists so a future change does not break it.

### RPY-07 · Composer image allowlist versus the reader's

- **Preconditions:** FX-01
- **Steps:** Open the mail — confirm the remote logo displays. Press **Reply**
  and look at the quote. Then click the image inside the quote and try to
  resize or move it.
- **Expected:** The image survives both reading and replying, and survives
  being edited.
- **Known:** **D-4.** Quoted content is exempt from the allowlist on insertion,
  so the image should survive the initial load — but the exemption does not
  cover attribute changes, so editing it may delete it and raise a blocking
  "Image Blocked" dialog. Record which of the two happens.

### RPY-08 · Pasting content with blocked images

- **Preconditions:** a web page with five images on a non-allowlisted host
- **Steps:** Copy a block containing all five and paste it into the composer.
- **Expected:** Either accepted, or refused once with a non-blocking message.
- **Known:** **D-4.** Count the dialogs. Confirm Escape dismisses each one and
  that the composer is still usable afterwards.

### RPY-09 · Signature images

- **Preconditions:** FX-13
- **Steps:** With the remote-logo signature active, open the composer. Then
  switch to the base64 signature, save it, reload the page, and open the
  composer again.
- **Expected:** Both signatures render their logo, and both survive a reload.
- **Known:** **D-4** for the first (deleted with a dialog every time the
  composer opens — the signature sits outside the quote, so it is not exempt)
  and **D-11** for the second (reports success, does not persist).

### RPY-10 · Reply All recipients

- **Preconditions:** a mail addressed to several To and Cc recipients,
  including one of your own mailbox addresses
- **Steps:** Press **Reply All**.
- **Expected:** The original sender is in To; everyone else in Cc; your own
  address is excluded; no duplicates.

### RPY-11 · Caret and draft survival

- **Preconditions:** any mail
- **Steps:** Press **Reply**, type a line, collapse the inline composer, open a
  different mail, come back, press **Reply** again. Then click into the body.
- **Expected:** The half-written draft is still there, and the caret lands
  above the signature and above the quote.

### RPY-12 · Switching the sending mailbox mid-draft

- **Preconditions:** two or more connected mailboxes with different signatures
- **Steps:** Start a reply, type a line, then change the **From** mailbox.
- **Expected:** The typed text survives, the signature swaps to the new
  mailbox's, and the quoted section is untouched.

### RPY-13 · Dropping an image into a reply

- **Preconditions:** any mail with an active signature
- **Steps:** Press **Reply**, place the caret mid-paragraph, drag an image file
  onto the body. Then try again with an image over 10 MB.
- **Expected:** The first lands at the caret. The second is refused with a
  clear message about the limit.
- **Known:** **D-10.** The first lands at the very end, below the signature and
  below the quote. The second does nothing at all.

### RPY-14 · Composing an image and checking the recipient

- **Preconditions:** none
- **Steps:** Compose a new mail. Insert an image three ways — the toolbar
  button, a clipboard paste, and a file drop. Send to Gmail, Outlook Web and
  Outlook desktop. Open in each.
- **Expected:** All three images display for all three recipients.
- **Known:** **D-3.** They are sent as `data:` URIs, which Gmail and Outlook
  strip. Record exactly what each client shows for each insertion method —
  they may differ, and that tells us whether a partial fix is possible.

---

## SEC — Sanitisation of sender HTML

### SEC-01 · Active content in the reading pane

- **Preconditions:** FX-12
- **Steps:** Open the mail. Hover over every element. Check the frame's DOM.
- **Expected:** Nothing executes. `<script>`, `<iframe>` and `<form>` are absent
  from the DOM; event-handler attributes are gone.

### SEC-02 · Sender CSS cannot reach the app

- **Preconditions:** a mail whose body contains a `<style>` block setting
  `body { background: red }` plus a rule targeting one of the app's own class
  names
- **Steps:** Open the mail.
- **Expected:** Only the mail body is affected. The surrounding app is
  untouched. This is what the iframe buys us and it should hold.

### SEC-03 · Active content in the **composer's** quoted section

- **Preconditions:** FX-12
- **Steps:** Open the mail, press **Reply**, and interact with the quoted
  section in the composer — hover every element, click into it, tab through it.
  Watch the console.
- **Expected:** Nothing executes.
- **Known:** **D-12, needs live confirmation.** The quote is rendered in the
  main document, not in the iframe, and never passes through DOMPurify. Jodit's
  defaults strip `<script>` and `onerror` but do not address other event
  handlers. If anything fires here it is an XSS in the app's own origin —
  **stop and escalate immediately rather than continuing the suite.**

### SEC-04 · CSP hardening dry run

- **Preconditions:** a local dev build; this one is for a developer, not QA
- **Steps:** Add `script-src 'self'` to the CSP meta tag in `index.html` and
  open any mail.
- **Expected:** Confirm the failure mode: the frame's inline script is what
  reports body height, forwards the scroll wheel, adapts dark mode and swaps
  inline images in. All four should stop, leaving blank bodies at the 300 px
  default with no error.
- **Why:** a `srcdoc` iframe inherits the parent's CSP. This is what will happen
  the day someone hardens the policy, and it needs a mitigation designed before
  then, not after.

---

## Gaps — not yet covered

These need their own suites and are deliberately out of scope here:

- **Theme switching, and dark-theme adaptation of sender HTML.** The frame is
  rebuilt when the theme changes, and a heuristic then walks every element
  neutralising near-white backgrounds and near-black text. Both the rebuild —
  including a toggle while inline images are still resolving — and the
  re-colouring need validating against real branded newsletters.
- **List and pagination performance**, mailbox switching, and the request
  de-duplication cache.
- **Folder counts.** The redesign branch increments counts by hand; the
  server-authoritative mechanism on `main` is absent on the branch, so drift
  is expected until the two are reconciled.
- **Regression of shared components.** The redesign also changed
  `MenuPopover` (used in 24 files), `DraggablePanel`, `CommonPagination`,
  `RichTextEditor` and a globally imported stylesheet — so Table, Email
  Campaign, System Account Admin and the Add modal all need a pass.
