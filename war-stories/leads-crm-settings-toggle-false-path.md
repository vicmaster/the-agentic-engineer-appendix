# leads-crm / The Invisible False Case

## Date / Version Context

- **Date:** Commit `73cae83` on 2026-04-29 fixed boolean persistence in the Settings form and added the controller-spec coverage for the false case. Merge commit `3b28c03` brought in the Codex-branch fix. The bug had shipped earlier in the 2026-04-22 to 2026-04-29 arc when the agent-built Settings page first landed.
- **Project:** leads-crm — Rails 8 / Hotwire / Stimulus / RSpec. Sales-qualification CRM. The Settings page is a single-organization configuration surface with multiple boolean toggles (autosave, notification preferences, feature flags) plus other field types.
- **Surface for this story:** `app/views/settings/index.html.erb` (the form view that had the missing hidden fields) and `spec/controllers/settings_controller_spec.rb` (the test that didn't assert the false case).
- **Glossary, used in this writeup:** *`form.check_box`* = Rails' form helper that renders an HTML checkbox; by default, also renders a paired hidden field with the same `name` and value `"0"` so that the unchecked case submits *"0"* (false) rather than nothing. When a Rails view omits `form.check_box` and uses raw HTML, the hidden field has to be added manually. *Autosave UI* = a form pattern where individual fields submit on change rather than waiting for a Submit button — common in Settings pages because there's no natural moment to "save." *Invisible false case* = the test scenario where a value transitions from true to false, which is invisible to tests that only assert on the true → true (or default → true) paths.

## What Was Being Attempted

Build an autosave Settings form where toggling a switch persists immediately.

The product instinct was clean: Settings forms shouldn't have a save button. Users toggle a switch, the page makes an AJAX request, the server persists, the user sees a small confirmation. Modern, frictionless, the kind of UI that doesn't make users hunt for a Save button after each change.

The agent built the form to spec. The view rendered ~6 boolean toggles plus other field types. The Stimulus controller listened for change events on the toggles, fired the autosave request, animated the confirmation. The controller spec covered the autosave path — *user changes a value, the controller receives the update, the record's attribute is updated.*

By every internal correctness criterion, the form worked. The test was green. The autosave fired on click. The confirmation animation played. A user toggling a switch *off and then back on* would see the new state persist.

What the form didn't preserve: a switch that was *currently on* being toggled *off* and *staying off* after a reload.

## What Went Wrong

The unchecked case submitted nothing, not false.

The mechanic is a well-known Rails / HTML gotcha. An HTML checkbox element only submits its value when it's *checked*. An unchecked checkbox submits *nothing* — its name doesn't appear in the form's parameter list at all. This is HTML form encoding semantics, not a Rails choice; the same is true for any framework on top of HTML forms.

Rails' `form.check_box` helper handles this by also rendering a paired hidden input with the same `name` and value `"0"`. The pair encodes the false case explicitly:

```html
<!-- form.check_box renders both: -->
<input type="hidden" name="settings[autosave]" value="0">
<input type="checkbox" name="settings[autosave]" value="1">
```

When the user checks the box, the form submits both `settings[autosave]=0` (from the hidden field) and `settings[autosave]=1` (from the checkbox); HTML form semantics resolve the duplicate to the last value (`"1"`), and the parameter parser sees `true`. When the user unchecks the box, the form submits only `settings[autosave]=0` (from the hidden field), and the parameter parser sees `false`. Both states map to a real parameter; ActiveRecord can update either way.

The agent's view used a raw HTML checkbox without the paired hidden field. The view probably looked like this:

```erb
<input type="checkbox" name="settings[autosave]" value="1" data-action="change->settings#save">
```

This *renders* fine. It *autosaves* fine when checked. But the unchecked-and-save case submits a parameter list with no `settings[autosave]` entry at all. The Settings controller's `update` action passes the params to `update_attributes`; ActiveRecord, seeing no `autosave` key in the hash, doesn't change the `autosave` column. The record's value stays whatever it was.

From the user's perspective, the sequence is:
1. Toggle is currently *on*.
2. User clicks to turn it *off*. Toggle visually flips to off.
3. Autosave fires. The confirmation animation plays.
4. User reloads the page.
5. Toggle is back *on*.

The toggle *appeared* to save. The confirmation played. There was no error. The Stimulus controller did exactly what it was designed to do. The Rails controller did exactly what it was designed to do. The bug was the missing hidden field, and its symptom was *the unchecked state silently failing to persist*.

The test didn't catch it for a related reason. The controller spec asserted:

```ruby
patch :update, params: { settings: { autosave: "1" } }
expect(Setting.first.autosave).to eq(true)
```

The test sends a `settings[autosave]=1` parameter and asserts the resulting state is true. That assertion is correct in isolation. What's missing is the partnered assertion:

```ruby
patch :update, params: { settings: { autosave: "0" } }
expect(Setting.first.autosave).to eq(false)

# AND, critically:
patch :update, params: { settings: {} }  # the actual unchecked-case shape
expect(Setting.first.autosave).to eq(false)
```

The last spec is the load-bearing one. It would have failed against the original code (because `update_attributes({})` doesn't change anything, leaving the value at `true`). It would have passed against the hidden-field-paired code (because the hidden field guarantees `settings[autosave]=0` is in the params even when the box is unchecked, so the test never sees the empty-params case).

The test didn't include that scenario because the test author (agent or human) was *modeling the UI's behavior, not HTML form encoding's behavior*. The test imagined the UI's submission shape from the perspective of *"the user is toggling, so a value is being sent"*; HTML form encoding's reality is *"only checked boxes submit values."* The test data shape and the code's mental model both came from the UI-first perspective. Neither could catch what HTML form encoding does in the unchecked case.

## How It Was Discovered

By another agent, on a different vendor, looking at the same repo.

The discovery channel is the one the companion story `leads-crm-codex-pr1.md` makes load-bearing: cross-vendor agent disagreement as a review primitive. Codex, pointed at leads-crm independently, opened PR #1 from branch `codex/fix-settings-toggle-autosave` (2026-04-29). Codex's review noticed the missing hidden field, named the Rails `check_box`-paired-hidden-field semantics, proposed the fix, and shipped the test for the false case.

The non-discovery channel — and the part this entry's lens focuses on — is *why the same-vendor review chain didn't catch it*. Three layers of review had already passed: the agent's own first-write pass (Claude wrote it and considered it done), the operator's review (Victor, scanning the diff, didn't flag it), and the test suite (green, because the false-case spec wasn't written).

All three layers shared the bug's mental model. The agent's mental model: *"the user toggles a switch, the form submits the new value, the controller updates the record."* The operator's mental model: *"this is a standard autosave Settings form."* The test's mental model: *"send a value, assert the record reflects it."* None of these mental models include *"HTML checkboxes submit nothing when unchecked."*

A reviewer who shares the bug's mental model can't catch it by reading more carefully. The bug isn't *visible* under that mental model. The reviewer needs to read with a *different* mental model — one anchored in HTML form encoding semantics, or in Rails-specific gotchas, or in the broader principle *what does the parameter look like when the user does nothing?* Codex's training surface (different from Claude's) happened to include the cross-vendor blind spot. The bug surfaced.

This is the *test-discipline* lesson on top of that *cross-vendor* lesson. The cross-vendor catch is one discovery channel; the structural discipline that would have caught it *without* the cross-vendor catch is **test every state transition, including the implicit-default one**. The false case has to be explicitly asserted, because the false case is the case where the user *does nothing* — and *do nothing* is the case the test author is most likely to forget.

## What Fixed It

Three lines of fix. One ERB tag rendered the paired hidden field; one ERB tag added the `form.check_box` helper that does it correctly by default; one spec asserted the false case.

The structural change:

```erb
<!-- Before: raw HTML checkbox, no hidden field -->
<input type="checkbox" name="settings[autosave]" value="1" data-action="change->settings#save">

<!-- After: Rails form helper, hidden field paired automatically -->
<%= form.check_box :autosave, data: { action: "change->settings#save" } %>
```

The view change is one ERB tag swap. The spec change adds:

```ruby
it "persists when the toggle is turned off" do
  Setting.first.update!(autosave: true)
  patch :update, params: { settings: {} }
  expect(Setting.first.reload.autosave).to eq(false)
end
```

The spec is load-bearing for the project's *next* boolean toggle. The view change fixed this specific toggle; the spec discipline catches the entire class of future toggles. Both ship in `73cae83`.

What's owed but not yet shipped: a project-level *test-the-false-case* checklist for any new form field — *"have you asserted both the true → false and the false → true transitions, and the no-change case?"* Currently lives in operator habit and Codex's stricter review. Worth writing down in the `.specify/test-discipline.md` template (which doesn't exist yet — see also the companion stories `forge-async-adapter-smoke-test.md`'s deploy-checklist note and `leads-crm-directory-linking-backfill.md`'s backfill-checklist note; the checklist surface is accreting across multiple stories).

## The Durable Lesson

The false path is where "simple" config UIs lie. **HTML form encoding submits nothing for unchecked boxes; the test author and the code author both work in mental models where *"the user toggles → a value is submitted,"* and neither catches the case where *no toggle = no value submitted*.**

The mental-model flip the lesson rests on: tests should assert *every possible state transition*, not just the named-explicit ones. The named-explicit transition (*true → true*, *false → true*) is *what the user does*; the implicit-default transition (*true → false* via unchecking, or *false → false* via not-changing) is *what the user doesn't do*. The implicit transitions are where the false-path bugs live, and they're the cases test authors are most likely to forget because they have no narrative anchor — *"the user toggled it"* is a story; *"the user left it alone"* is the absence of a story.

For agent-built forms specifically, the cost compounds. The agent's natural completion criterion is *the user-visible flow works* — toggle changes state, autosave fires, confirmation animates. The criterion has no signal for *the parameter shape that arrives at the server when the user takes the default action.* The agent extends mental models present in the spec; HTML form encoding semantics aren't in most specs unless the operator names them.

> **Heuristic.** For any form field with a default state — checkboxes, radio buttons (when one is pre-selected), select dropdowns (with a first-option default), file inputs (the no-file case) — write a test that asserts the *value-doesn't-change* scenario, with the parameter list that the browser actually sends when the user takes the default action. For checkboxes specifically, that means asserting `params: { resource: {} }` produces the expected false-state record, not just `params: { resource: { field: "1" } }` produces the true-state record. The default case is the implicit case; the implicit case is where the bug hides.

The shape generalizes beyond Rails / HTML forms:

- **JSON APIs with partial updates.** A `PATCH /resource` endpoint that updates only the keys present in the body has the same shape: clients that *omit* a key get *no change*, not *the default value*. Tests have to assert the absent-key case explicitly.
- **GraphQL mutations.** Same pattern at a different layer — fields not included in the mutation aren't changed; tests have to cover the omitted-field case.
- **Environment-variable config.** A config system where unset variables default to some value (often empty string or null) hides bugs where the unset case isn't tested. Tests have to cover *what happens with no value*.
- **Feature flags with default-off semantics.** A flag system where missing flags evaluate to false silently — code paths that rely on *the flag being set to false* may never test the *flag missing entirely* case.
- **CSS / styling defaults.** A component whose style depends on a class being present has a *no-class* case that's easy to forget to test.

In every case, the pattern is identical: **the case where the user / caller / environment *does nothing* is the case where the system's default behavior is most load-bearing and least tested.** The fix is structural — assert the do-nothing case explicitly for every field, key, flag, or class that affects behavior.

The pairing worth naming: this is the *form-encoding* version of the *synthetic-validation-drifts-from-reality* pattern. See the companion stories `forge-channel-id-regex.md` and `canvas-mcp-design-md-colors-parser.md` — all three stories share *the test data and the code share a mental model, so the test can't catch the code's mistake*. The first is regex-vs-Slack-IDs; the second is community-DESIGN.md-parsing; this one is form-encoding semantics. Three layers of the same lesson — *the synthetic check shares the bug's blind spot* — applied at three different boundary layers (external system, community input, browser-to-server form encoding). See also the companion story `leads-crm-codex-pr1.md` on the same incident from a different angle: it names the *cross-vendor agent disagreement* as the discovery channel; this story names the *test-discipline-on-the-false-path* as the structural prevention. The two together cover what the incident teaches.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *fields with a default state that the user can decline to change*. It doesn't apply to:

- **Required fields with no default.** A required text field that the user must fill in doesn't have an implicit-default case to test. The validation catches the empty-submission path.
- **Fields where the API explicitly distinguishes *unset* from *false*.** Some JSON APIs treat `{ foo: null }`, `{ foo: false }`, and `{}` as three different states. If your code handles all three correctly, the lesson is already practiced.
- **Idempotent toggles on a controlled UI surface.** A toggle where the UI guarantees the parameter is *always* submitted (e.g., via a Stimulus controller that explicitly serializes the unchecked state as `false`) doesn't need the hidden-field pair. The lesson is for cases where the form's submission shape is *not* controlled by the application layer.
- **Read-only fields.** A field that displays but doesn't accept user modification doesn't have a state-transition test surface. The lesson is for editable fields.
- **Cases where the test suite already covers state-machine transitions exhaustively.** Some teams write tests that explicitly enumerate every transition; for them, the lesson is already practiced. The lesson is for the more common case where tests cover *the user's typical actions* rather than *every possible state transition*.

The signal: *can a user (or caller) cause the field to be unchanged without an explicit action?* If yes, write the test for that case. If no (every change requires an explicit action that submits a value), the implicit-default case doesn't exist.

## What This Story Is *Not* Evidence For

- **Not evidence that Rails `check_box` is poorly designed.** It does exactly the right thing — pairs a hidden field with the checkbox to handle HTML's unchecked-submits-nothing semantics. The bug was using raw HTML *without* the helper.
- **Not evidence that HTML form encoding is poorly designed.** Same point. The unchecked-submits-nothing rule is the right rule for *"the user didn't interact with this field"* — it just collides with the *"the user explicitly turned this off"* case that web apps want to distinguish.
- **Not evidence that autosave UIs are problematic.** They're fine. The bug is at the encoding layer, not the autosave layer. A non-autosave form with a Save button would have the same bug.
- **Not evidence that Claude (the agent that wrote the original code) was uniquely careless.** The same bug would happen in any agent's output (or any human developer's output) without the cross-vendor catch. The lesson is about *test discipline that catches this class of bug structurally*, not about which agent produced it.
- **Not evidence that the cross-vendor review is the only discovery channel.** It was *one* discovery channel that worked here. The structural fix (test the false case) makes the discovery channel-independent — the next instance of this bug shape would be caught by the spec, not by waiting for a second-vendor review.
