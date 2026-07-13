---
name: assessments
description: Author interactive assessment items in Graffiticode — multiple-choice quizzes, flashcards, spreadsheet problems, area-model math, magic squares, map-based questions, grade-level subject assessments (e.g. ELA reading/evidence items), and more. Use whenever the user wants to build a quiz, test, homework problem, study deck, or rubric-scored practice item across mixed question types. For requests that specifically target Learnosity (by name, or by referring to Learnosity's Item Bank, Items API, or LMS integration), prefer the `learnosity` skill instead — it is the narrower, Learnosity-focused sibling.
---

# Assessments

The `assessments` skill is the assessment authoring surface of Graffiticode. Each assessment type is backed by a different Graffiticode language, and the full set is discovered at runtime — the catalog is dynamic. Your job is to route the user's request to the right language and produce a rendered item, not to write code yourself.

## Prerequisite

The Graffiticode MCP connector must be installed and connected. If `list_languages` is unavailable, tell the user to connect it before proceeding.

## Workflow

Every authoring request follows the same four steps. Do not skip steps 1–2; the catalog changes over time and hardcoding language IDs is wrong.

**1. Discover the assessments language set.**

Call `list_languages(domain: "assessments")`. This returns the current domain members with their `id`, `name`, `description`, and `domains`. Read the descriptions — this is the source of truth.

**2. Pick the best match by shape of request.**

Match the user's intent against the returned `description`s. If more than one language could fit, call `get_language_info(language)` on the top candidate to see `supported_item_types` and `example_prompts` before deciding. For deeper reference, read the `user_guide_resource` URI via `ReadResource`.

Rough shape-to-language mapping (verify against actual `description`s; do not rely on memorized IDs).

**Specificity wins.** If the request names a subject, grade, or standard, route to the dialect that specializes in it — matched by `description` — *even though that subject's items use multiple-choice / short-text / cloze question types*. Do NOT send a subject-specific request to the general Learnosity language just because of the question type; the question type is not the discriminator, the subject/grade/standard is.

- **Subject-, grade-, or standards-specific assessments** (e.g. "Grade 5 ELA", a reading/evidence target, an SBAC/state-standard reading item) → the specialized dialect whose `description` names that subject + grade. Match by description, so a subject specialist added later wins here automatically.
- **Spreadsheet / tabular / formula-based problems (SUM, AVERAGE, IF, parameterized values)** → the spreadsheet language.
- **Flashcards, vocabulary pairs, match games, memory games** → the flashcard language.
- **Area-model multiplication with visual grids** → the area-model language.
- **Magic-square puzzles with grid number placement** → the magic-square language.
- **Interactive map / location-based questions** → the map-question language.
- **Concept webs / relationship diagrams** (central anchor, radial links, drag-and-drop concepts) → the concept-web language.

**There is no general fallback — and that is deliberate.** If no language in the returned set fits, do **not** quietly pick the closest one. A vendor-specific language (e.g. a Learnosity language, which emits Learnosity-shaped JSON for a Learnosity Item Bank / Items API / LMS) is **never** the answer to a generic request, however well its question types match. Instead:

1. Tell the user plainly what Graffiticode *does* have for their request — name the closest specialists and what each produces.
2. Ask how they want to proceed — e.g. fit the request to a specialist, or, **if and only if they actually use Learnosity**, author it as a Learnosity item.

Never infer Learnosity (or any vendor) from the question type alone. "Multiple-choice," "cloze," and "short text" are shapes every assessment platform has; they say nothing about the target platform. Only the user naming the platform does.

**3. Create the item.**

Call `create_item(language, description)` with a natural-language description. The `description` is a prompt to a language-specific AI, not Graffiticode source — write it as you would explain the item to a colleague.

A good description is specific about:
- **Subject and scope** — topic, grade band, difficulty
- **Quantity and structure** — number of items, layout, sections
- **Assessment rules** — scoring, rubric, answer key expectations, hints
- **Theme / styling** — color, tone, any accessibility needs

Bad: "Make a quiz about fractions."
Good: "Create a 5-item multiple-choice quiz on adding fractions with unlike denominators. Grade 5 level. Each item has four choices with one correct answer and three plausible distractors that reflect common computational errors. Include an answer key and a one-sentence explanation per item."

**4. Iterate with `update_item`.**

`update_item(item_id, modification)` preserves conversation history, so incremental edits compose naturally: "make the distractors harder," "add a hint on question 3," "switch to a dark theme," "change the topic from fractions to decimals." Prefer iteration over recreation — history is lost on a fresh create.

## Composite requests (content + a host/format)

Some requests are really **two parts of one whole**: a piece of *content* plus a *host or output
format* it should live in — e.g. "an ELA Grade 5 item **for Learnosity**", "a spreadsheet
question **in a Learnosity item**", "turn this passage into a Learnosity EBSR". Do **not** try to
one-shot these with a single `create_item` in the host language — the host will then author the
inner content itself, generically, instead of the right specialist authoring it.

Treat every such request uniformly as a **round-trip**: author the inner part, carry it across
with `get_spec`, then create the host item from that spec.

1. **Author the inner content in its own specialist.** Describe just the content (e.g. "a Grade 5
   ELA, Claim 1 Target 11 reasoning-and-evidence item about <topic>") and `create_item` it — the
   server routes it to the right specialist dialect. `get_item` to confirm it's ready.
2. **`get_spec(inner_item_id)`** — returns a complete, platform-neutral English description of the
   authored content (passage, stems, options, answer keys, rationales — everything).
3. **Create the host item from the spec.** `create_item(host_language, <that spec> + your intent
   framing)` — e.g. "Create a Learnosity EBSR from the following content: <spec>".

You never decide *how* the two parts combine — whether the host **embeds** the inner item as a
live widget or **re-authors** it natively is the generator's call. Your job is only to ask for the
two parts of the whole. Never paste an item's `src`/`data` or its id across languages — `get_spec`
is the only correct bridge.

## Output

Items render as interactive widgets inline in claude.ai. The tool response carries the widget metadata automatically. **The widget is the rendering. Your reply is a one-line summary, nothing more.**

Prefer the response's own summary fields for that one-sentence confirmation:

- **On first creation** (`create_item`): echo `description` ("what the code does") — e.g., *"Made a 5-item MCQ on photosynthesis with four distractors each."*
- **On edits** (`update_item`): echo `change_summary` ("what changed this turn") — e.g., *"Switched to dark theme and hardened the distractors on Q3."*

Don't re-parse `data` to describe what changed; the backend already wrote the summary for you. If a field is `null` (rare — typically only when the code generator failed), fall back to a brief summary drawn from the user's own request.

**Do not preview or simulate the item in chat.** No sample layouts, no mock multiple-choice blocks, no ASCII/Markdown renderings of the stem and options, no printed answer keys, no "here's what it looks like" sections. The widget renders the item — your one-liner is in addition to, not a substitute for, the widget. If the user asks "what does it look like?" or "show me the questions," point them at the widget; do not reproduce the content in prose or formatted text.

## Saving the item (free plan)

Each `create_item` / `update_item` response carries a **`view_url`** (the item's page on `app.graffiticode.org`); surface it so the user can open or share the rendered item. When the call was made **without credentials (free plan)**, the response also includes a **`claim_url`** and a **`claim_message`**, and the `view_url` carries the claim token — so when the user opens it, the render-host footer offers a one-click **"Claim it in Graffiticode →"** link for that item (the primary way to save it). Surface the `view_url` and, in chat, the `claim_message` (the same `/claim` destination by a manual route). Free-plan items are session-scoped and expire after 48 hours unless claimed. Only surface the URLs the server returned; if `claim_url` is absent the call was authenticated and the item already persists.

## Guardrails

- **Never write Graffiticode DSL directly.** The backend generates code from natural-language descriptions. If you catch yourself composing Graffiticode source, stop and use `create_item`/`update_item` instead.
- **Never hardcode language IDs in your reasoning.** Call `list_languages(domain: "assessments")` every session; memorized IDs go stale.
- **Do not invent languages.** If no returned language matches, say so — don't guess an ID.
- **Prefer domain-scoped discovery.** When the user is clearly in an assessment context, scope `list_languages` by `domain: "assessments"` rather than searching the whole catalog — it's faster and reduces wrong-language picks.
- **Never pick a Learnosity language unless the user named Learnosity.** A `learnosity`-domain language is off the table unless the user named Learnosity, an Item Bank, the Items API, or a Learnosity-integrated LMS. Question type (MCQ, cloze, short-text, ordering, choice-matrix) is **never** the discriminator — every platform has those. When the user *has* named Learnosity, the `learnosity` skill (if installed) is the better fit; it is tighter and scoped to that domain.
- **No silent fallback.** If nothing in the `assessments` set matches, say so and ask (see the routing section) — never settle for the nearest-looking language.
- **Respect the conversation.** On follow-up edits, call `update_item` on the existing `item_id`; don't start over unless the user explicitly asks for a new item.
