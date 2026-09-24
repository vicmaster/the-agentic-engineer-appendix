# leads-crm / The Toggle That Showed the Database

## Date / Version Context

- **Date:** Commit `73cae83` on 2026-04-29 fixed boolean toggle persistence in the Settings form and added the first controller spec that saves a boolean setting as `false`. Merge commit `3b28c03` brought in the Codex-branch fix. The toggle UI had landed in the 2026-03-20 design overhaul (`a761614`), so the bug was in production roughly six weeks.
- **Project:** leads-crm — Rails 8 / PostgreSQL / Hotwire / RSpec. Sales-qualification CRM. The Settings page renders every setting, grouped by category, inside one `form_with url: update_all_settings_path, method: :patch` form: booleans as On/Off toggle groups, numbers, arrays, and strings as inputs, and one *Save All Changes* submit button at the bottom of the page.
- **Surface for this story:** `app/views/settings/index.html.erb` (the view and its inline CSS, where the toggle's highlight was computed) and `spec/controllers/settings_controller_spec.rb` (where no spec saved a boolean).
- **Glossary, used in this writeup:** *Toggle group* = the page's On/Off control: two `<label class="toggle-btn">` elements, each wrapping a radio button hidden with `display: none`, one with value `"true"` and one with `"false"`, so clicking a label checks its radio. *Render-time state* = a CSS class computed on the server from the stored value when the page is rendered; it doesn't change until the page is rendered again. *Autosave* = submitting the form when a control changes rather than waiting for a Save button — what Codex's fix added, and what the branch name `codex/fix-settings-toggle-autosave` refers to.

## What Was Being Attempted

Give the Settings page a cleaner On/Off control for boolean settings.

The 2026-03-20 design overhaul restyled the app, and the Settings page's booleans became segmented On/Off toggle groups: two button-styled labels side by side, the active one highlighted. Under each label sat a hidden radio button, so clicking the label checked the radio and the form would carry `"true"` or `"false"` for that setting.

The rest of the page stayed a classic form. Every setting — booleans, integers, floats, arrays, strings — lived inside one `form_with` block, and the only thing that sent it to the server was the *Save All Changes* button at the bottom.

By every check that existed, it worked. Each toggle rendered with the right side highlighted for the stored value. Submitting the form with *Save All Changes* persisted whatever the radios said. The controller spec for `update_all` was green.

What the page didn't do: show the user their click.

## What Went Wrong

The highlight followed the database, not the control.

Here's the pre-fix toggle, as it shipped:

```erb
<label class="toggle-btn <%= setting[:value] == true ? 'on' : '' %>">
  <%= f.radio_button "settings[#{setting[:key]}]", "true", checked: setting[:value] == true, style: "display: none;" %>
  On
</label>
<label class="toggle-btn <%= setting[:value] == false ? 'off-active' : '' %>">
  <%= f.radio_button "settings[#{setting[:key]}]", "false", checked: setting[:value] == false, style: "display: none;" %>
  Off
</label>
```

And the CSS that lit it:

```css
.toggle-btn.on { background: var(--accent-green); border-color: var(--accent-green); color: #FFF; }
.toggle-btn.off-active { background: var(--bg-surface); border-color: var(--border); color: var(--text-primary); }
```

The `on` and `off-active` classes are computed once, on the server, from `setting[:value]` — the saved value. No JavaScript touched them afterward. Clicking a label did change which hidden radio was checked, but the highlight was baked into the HTML and stayed where the database had put it.

From the user's perspective, the sequence is:
1. A setting is saved as *On*. The On button is green.
2. The user clicks *Off*. The hidden `"false"` radio becomes checked. The On button stays green.
3. Nothing is submitted. The change exists only in the browser, in a radio the user can't see.
4. The user clicks again (still nothing visible), decides the toggle is broken, or leaves the page. The change is gone.
5. If they happened to scroll down and click *Save All Changes*, it saved. The server path was fine.

The same held in the other direction for a setting saved as Off. Nothing errored. Every piece did what it was written to do. The bug was the gap between the pieces: **the visible state was wired to the stored value instead of to the control, and saving depended on a separate button the toggle gave no hint about.**

The test suite didn't catch it for a structural reason: it didn't look at that layer. Before the fix, the `update_all` specs exercised one integer setting (`scoring.cold_score_max`) — a valid save and an invalid one. No spec saved a boolean at all, in either direction. The spec Codex added — patch `'false'` to `update_all`, assert the setting reads back `false` — pins down the server half. But the fix didn't touch the controller, so that path was already correct; the new spec guards it going forward rather than reproducing the bug. The bug lived in the rendered page, between a click and a save, and no spec drove the page that way.

Everyone was working from the same model: *"the page shows the setting's value."* That was true on every page load. It was false for exactly the window that mattered — after the click, before a save that never came.

## How It Was Discovered

By another agent, on a different vendor, looking at the same repo.

The discovery channel is the one the companion story `leads-crm-codex-pr1.md` makes load-bearing: cross-vendor agent disagreement as a review primitive. Codex, pointed at leads-crm independently, opened PR #1 from branch `codex/fix-settings-toggle-autosave` (2026-04-29). Codex's fix addressed both halves: it tied the highlight to the checked radio instead of the stored value, made each radio submit the form on change, and added a spec for saving a boolean as `false`.

The non-discovery channel — and the part this entry's lens focuses on — is *why the same-vendor review chain didn't catch it*. Three layers of review had already passed: the agent's own first-write pass (Claude wrote it and considered it done), the operator's review (Victor didn't flag it), and the test suite (green, because nothing tested booleans, and nothing tested what the page did after a click).

All three layers shared the bug's mental model. The agent's mental model: *"render each setting from its stored value"* — the natural move when you're generating a page from a settings hash, and exactly the move that baked the highlight in. The operator's mental model: *"the toggles show the right state"* — true every time he loaded the page. The test's mental model: *"send params, assert the record"* — correct, and aimed at a layer where nothing was wrong. None of these mental models include *"what does the control show between the click and the save?"*

A reviewer who shares the bug's mental model can't catch it by reading more carefully. The bug isn't *visible* under that mental model. The reviewer needs to read with a *different* mental model — one anchored in the user's click rather than the stored value, or in the broader principle *what does the page show after the user acts, before anything is saved?* Codex's training surface (different from Claude's) happened to include the cross-vendor blind spot. The bug surfaced.

This is the *test-discipline* lesson on top of that *cross-vendor* lesson. The cross-vendor catch is one discovery channel; the structural discipline that would have caught it *without* the cross-vendor catch is **test the change path, at the layer where the change happens**. A check that renders the page and compares it to the stored value passes by construction, because the page *is* the stored value. The check that catches this one changes the value — click Off, confirm the highlight moves, reload, confirm it stuck. The off transition is the natural one to write first, and it's the one Codex's spec picked; the rule underneath it is broader: any transition away from what's stored.

## What Fixed It

Eighteen lines of view (+12/−6) and nine of spec. The view change does two things.

The highlight now follows the control. The CSS keys off whichever label contains the checked radio, and the labels carry static classes instead of server-computed ones:

```css
.toggle-btn:has(input:checked).on-option { background: var(--accent-green); border-color: var(--accent-green); color: #FFF; }
.toggle-btn:has(input:checked).off-option { background: var(--bg-surface); border-color: var(--border); color: var(--text-primary); }
```

And a click is a save. Each radio submits the form when it changes:

```erb
<label class="toggle-btn on-option">
  <%= f.radio_button "settings[#{setting[:key]}]", "true",
      checked: setting[:value] == true,
      onchange: "this.form.requestSubmit();",
      style: "display: none;" %>
  On
</label>
<label class="toggle-btn off-option">
  <%= f.radio_button "settings[#{setting[:key]}]", "false",
      checked: setting[:value] == false,
      onchange: "this.form.requestSubmit();",
      style: "display: none;" %>
  Off
</label>
```

The spec change adds:

```ruby
it "updates boolean settings to false" do
  SettingsManager.set('scoring.enable_auto_promotion', true)

  patch :update_all, params: { settings: { 'scoring.enable_auto_promotion' => 'false' } }

  expect(response).to redirect_to(settings_path(tab: 'configuration'))
  expect(SettingsManager.get('scoring.enable_auto_promotion')).to be false
end
```

The spec is load-bearing for the server path on the project's *next* boolean setting — it's the first time the suite pins down that a boolean can be saved as `false`. What it can't do is see the view. The view change fixed these toggles; a test that clicks a toggle and watches it is what would catch the next instance of this exact shape, and that test didn't ship in `73cae83`.

What's owed but not yet shipped: a project-level checklist for any new form control — *"does its visible state follow the control or the stored value? What saves it, and would the user know? Is there a test that changes it and reloads?"* Currently lives in operator habit and Codex's stricter review. Worth writing down in the `.specify/test-discipline.md` template (which doesn't exist yet — see also the companion stories `forge-async-adapter-smoke-test.md`'s deploy-checklist note and `leads-crm-directory-linking-backfill.md`'s backfill-checklist note; the checklist surface is accreting across multiple stories).

## The Durable Lesson

Simple config UIs lie when they show the database instead of the control. **A control whose visible state is computed from the stored value at render time looks correct on every page load and wrong the moment the user acts; when saving also depends on a separate button, the user's change quietly evaporates. Tests anchored on the stored value — render the page and compare to the record, send params and compare to the record — pass by construction.**

The mental-model flip the lesson rests on: tests should assert the *transition*, not the *snapshot*. The snapshot (*the page matches the record*) is what every page load shows. The transition (*the user acts → the control reflects it → the value persists*) is what the user actually does. The transition is where this bug lived, and it's the one a server-side suite can't see.

For agent-built forms specifically, the cost compounds. Keeping a control's visible state in sync with its value is basic, not obscure — but the agent's natural completion criterion is *the page renders the settings correctly*, and rendering from stored state is the most direct way to meet it. The criterion has no signal for *what the page shows after a click that hasn't been saved.* The agent extends mental models present in the spec; interaction state isn't in most specs unless the operator names it.

> **Heuristic.** For any control whose appearance is styled rather than native — segmented buttons, custom switches, hidden inputs under styled labels — check two things: that the visible state is driven by the control's own state (`:checked`, a bound value) rather than by a class computed from the stored value, and that something saves the change without the user having to know about a button elsewhere on the page. Then write one test that drives it like a user: change it, confirm the display follows, reload, confirm it stuck. Pair that with a server-side spec per direction — Codex's `'false'` spec is the model — so the persistence half is pinned too.

The shape generalizes beyond Rails and settings pages:

- **Server-rendered state the user can change.** Any badge, highlight, or selected state computed at render time from stored data goes stale the moment the user interacts, unless something client-side takes over.
- **Long forms with one Save button.** Every control above the button holds unsaved state with no indicator, and navigating away drops it. The fix is either save-on-change or a visible *unsaved changes* signal.
- **Optimistic UI, in reverse.** Optimistic UI shows the new state before the server confirms, and its risk is showing *saved* when the save failed. This bug is the mirror: showing *stored* when the user has already changed it. Both are a gap between visible state and persisted state.
- **Config consoles that show the file, not the running process.** An admin panel that reads config from disk shows a change as live before the service picks it up — visible state wired to the wrong source.

In every case, the pattern is identical: **the visible state is wired to something other than the thing the user is acting on, and the test checks that other thing.** The fix is structural — bind the display to the control, make the save path automatic or obvious, and test the interaction, not just the stored value.

The pairing worth naming: this is the *UI-state* version of the *synthetic-validation-drifts-from-reality* pattern. See the companion stories `forge-channel-id-regex.md` and `canvas-mcp-design-md-colors-parser.md` — all three stories share *the check and the code share a mental model, so the check can't catch the code's mistake*. The first is regex-vs-Slack-IDs; the second is community-DESIGN.md-parsing; this one is rendered-state-vs-control-state. Three layers of the same lesson — *the synthetic check shares the bug's blind spot* — applied at three different boundary layers (external system, community input, browser UI state vs. server-stored state). See also the companion story `leads-crm-codex-pr1.md` on the same incident from a different angle: it names the *cross-vendor agent disagreement* as the discovery channel; this story names *testing the change path* as the structural prevention. The two together cover what the incident teaches.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *controls whose visible state can drift from their actual state*. It doesn't apply to:

- **Native, visible form controls.** A plain checkbox or radio button the user sees directly shows its own checked state; the browser keeps display and value in sync. The drift here came from hiding the radios and styling the labels from the stored value.
- **Controls bound to client-side state.** If a front-end framework or controller derives the visible state from the control's current value, the render-time snapshot stops being the source of truth after load.
- **Save-button forms that say so.** A form that marks pending changes and warns before navigation has made the save step visible; the user isn't led to believe a click already saved.
- **Read-only displays.** A field that shows a value but doesn't accept changes has no transition to test.
- **Suites that already drive the UI.** If system tests already click through each control and reload, the lesson is already practiced. The lesson is for the more common case where the suite tests the server and trusts the page.

The signal: *after the user acts and before anything is saved, what is the visible state wired to?* If the answer is the stored value, the page will lie.

## What This Story Is *Not* Evidence For

- **Not evidence that hidden radios under styled labels are a bad pattern.** The fix kept them. It changed what the styling keys off, from the stored value to `:checked`.
- **Not evidence that a single Save button is wrong.** Save-button forms are fine when the controls show pending changes and the button is where the user expects it. The failure was the combination: a toggle that looked instant, on a form that wasn't.
- **Not evidence that autosave is always the answer.** It was the right fix here because a toggle reads as instant. For fields users edit in batches, a visible save step can be the better design.
- **Not evidence that Claude (the agent that wrote the original code) was uniquely careless.** The same bug would happen in any agent's output (or any human developer's output) without the cross-vendor catch. The lesson is about *test discipline that catches this class of bug structurally*, not about which agent produced it.
- **Not evidence that the cross-vendor review is the only discovery channel.** It was *one* discovery channel that worked here. The structural fix (test the change path) makes the discovery channel-independent — with the caveat that the spec that actually shipped guards the server path; the UI-level test that would catch the next instance of this exact shape is still owed.
