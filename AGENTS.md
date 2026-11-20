# AGENTS.md

Operating rules for AI agents working in this repository.

This repo is a **documentation base**, not an application. It has no build, tests, or runnable source in normal operation. The Markdown files are the product. Two on-request protocols ([F](#f-verify-code-examples-in-practice-on-request), [G](#g-audit-documentation-for-fabrications-on-request)) can create a temporary gitignored lab, but nothing from it is ever committed.

**Languages:** documents are written in **Russian**; code, API names, flags, file paths, commit messages, and this file are **English**.

---

## Rule 0 — map first, scan never

The table in [Project map](#project-map) is the index of this repository. Treat it as the source of truth for *where things live*.

The default workflow is **map → grep → slice**:

1. Find the topic in the map.
2. Locate the exact term with `rg -n '<term>' <file>`.
3. Read only the matching section (`read` with `offset`/`limit`), not the whole file.
4. Edit with a targeted `edit`, not a full rewrite.

Open a file end-to-end only when you are rewriting it, or when the map says it is short. Reading every document "to get context" burns tokens and adds nothing: most files are 500–1900 lines and repeat the same canon.

**Completion criterion:** you can name the one or two files that own the fact you are changing, and you have read no more than the sections you need.

---

## Project map

Paths are relative to the repository root. "Reach for it when" lists the branches that should trigger opening the file — if none apply, do not open it.

| Path | What it holds | Reach for it when |
| --- | --- | --- |
| `README.md` | Public entry point: purpose, file table, key conclusions, page conventions, AI disclosure | Onboarding; updating the file table; changing public conventions |
| `AGENTS.md` | This file: agent workflow, canon, verification, on-request code/audit protocols | Any task starts here |
| `00-glossary.md` | Terms: loaders, mappings, Mixin, injectors, data components, rendering, debugging | A term is ambiguous, or you need the canonical one-line definition |
| `01-toolchain/1-build-systems.md` | Gradle plugins per era, Java/Gradle matrix, minimal build scripts, NeoForm/NFRT | Build fails; adding a project template; a plugin version changed |
| `01-toolchain/2-ide-and-navigation.md` | IntelliJ, Minecraft Development, Ravel, MixinMCP, jdx, mapping browsers, navigation routes | Choosing an IDE tool; explaining how to find a class or method |
| `01-toolchain/3-decompilers-bytecode.md` | Vineflower/CFR/Recaf, `javap`, bytecode reading, instruction → `@At` table | A decompiler lies; descriptor or instruction semantics are unclear |
| `01-toolchain/4-debug-and-diagnostics.md` | `mixin.debug.*` flags, log reading, hotswap, spark/JFR, GameTest/HeadlessMc | Runtime debugging, profiling, test automation |
| `01-toolchain/5-content-and-pipeline.md` | Datagen (NeoForge/Fabric), ItemModel, Blockbench, scripting, NBT tools, publishing, CI | Assets, generated JSON, release automation |
| `01-toolchain/6-tool-matrix.md` | Cross-category tool table with dead/legacy markers | Picking a tool; checking whether a tool is maintained |
| `02-mixin-playbook.md` | **Core:** four decisions, 12-step target search, `@At` reference, `@Expression`, injector matrix, remap rules, end-to-end examples | Any "where do I inject / which injector" question |
| `03-mixin-cookbook.md` | 18 recipes in 1.20.1 / 1.21.x / 26.x variants | A concrete task: damage, tooltip, HUD, tick, packet, recipe, spawn, etc. |
| `04-mixin-debugging.md` | Failure triage, error catalogue, refmap/namespace, conflicts, dev vs prod | A mixin does not apply or crashes |
| `05-version-changes/00-index.md` | Timeline 1.7.10 → 26.4, six eras, five watersheds, "where did X move" table | Locating the era for a version or a rename |
| `05-version-changes/1-legacy-forge.md` | 1.7.10–1.12.2: MCP, coremods, `GameRegistry`, ASM | Porting from legacy Forge; Mixin history |
| `05-version-changes/2-flattening.md` | 1.13–1.16.5: flattening, registries, `DeferredRegister`, `MatrixStack` | Porting 1.13–1.16 mods |
| `05-version-changes/3-modern-rename.md` | 1.17–1.19.4: Mojmap, `Level`/`BlockEntity`, registries, chat | Porting 1.17–1.19 mods |
| `05-version-changes/4-data-components.md` | 1.20.x: Forge/NeoForge split, data components, datapack registries | Porting 1.20.x mods; item data questions |
| `05-version-changes/5-render-rewrite.md` | 1.21.x: ItemModel, RenderPipeline, GuiRenderState, feature submission | Porting 1.21.x; GUI/render mixins |
| `05-version-changes/6-unobfuscated.md` | 26.1–26.3: deobfuscation, Java 25, templates, Renderpearl/SDL3/OIT | Anything 26.x; mapping removal; graphics backend |
| `06-loaders-and-mappings.md` | Loader architecture, ModLauncher/Knot, mapping history, registries/events, Architectury/MultiLoader/Unimined, Sinytra, porting checklists, legacy stack | Choosing a loader; cross-loader port; mapping questions |
| `07-sources.md` | Primary sources by topic, verification method, refresh checklist | Adding a claim; checking a source; post-drop update |
| `LICENSE` | MIT | Legal only |

If a fact lives in one of these files, **link to it** instead of restating it. When a fact is unavoidably repeated (version numbers, Java requirements), it belongs to the [Canon](#canon) and must match everywhere.

`tests/` is deliberately absent from this map. It is a gitignored scratch lab created only under [protocol F](#f-verify-code-examples-in-practice-on-request); documents must never link to it.

---

## Canon

These facts are cross-cutting and have caused contradictions before. Every document must agree with this table and the bullets below. If a source disproves one, update the canon first, then `rg` for the old value and fix every copy.

| Minecraft | Java | Primary loader stack | Build plugin | Mappings |
| --- | --- | --- | --- | --- |
| 1.7.10–1.16.5 | 8 | Forge | ForgeGradle 1–5, RetroFuturaGradle | MCP |
| 1.17.x | 16 | Forge, Fabric | ForgeGradle 5, Loom | Mojmap + SRG |
| 1.18–1.20.4 | 17 | Forge, NeoForge (1.20.2+), Fabric, Quilt | ForgeGradle 6, ModDevGradle, Loom | Mojmap (+ Parchment) |
| 1.20.5–1.21.11 | 21 | NeoForge, Fabric | ModDevGradle, NeoGradle, Loom | Mojmap (+ Parchment) |
| 26.1–26.3 | 25 | NeoForge, Fabric | ModDevGradle 2.x, NeoGradle 7.x, Loom 1.14+ | none — game is deobfuscated |

- **1.21.11 is the last obfuscated release.** 26.1 is the first unobfuscated one: Yarn, intermediary, and refmaps are not used for 26.x; `officialMojangMappings()` fails on 26.x.
- **`ResourceLocation` was renamed to `Identifier`** in 1.21.11/26.1. Same type, same methods.
- **26.1 changed the item model:** `ItemStack` needs loaded registries, so immutable data uses `ItemStackTemplate`; NeoForge mirrors this with `FluidStackTemplate`/`ItemResource`.
- **26.1 renamed the GUI context:** `GuiGraphics` → `GuiGraphicsExtractor`, `Screen#render` → `extractRenderState`.
- **26.2 added experimental Vulkan**; **26.3 moved the render API to Renderpearl, switched GLFW → SDL3, and added OIT.**
- **Loom split plugin IDs in 1.14:** `net.fabricmc.fabric-loom` (unobfuscated) vs `net.fabricmc.fabric-loom-remap` (obfuscated). Loom needs Gradle 9.2+.
- **MixinExtras is the default for new code:** `@WrapOperation` over `@Redirect`, `@ModifyExpressionValue` over `@ModifyConstant`, `@ModifyReturnValue` over `@Inject(RETURN)`.
- **`@At("LOAD")` / `@At("STORE")` exist but work only with `@ModifyVariable`.**
- **`mixin.remap` is not a system property.** `remap` is an annotation attribute; the property is `mixin.env.remapRefMap`.

---

## Sources and uncertainty

Every factual sentence needs a primary source: official docs, javadoc, game/loader source, maven-metadata, or the Minecraft Wiki. Community posts may support a claim but never carry it alone.

When writing:

1. Prefer the source that *owns* the fact: Mojang for the game, Fabric/NeoForge for their APIs, the repository for a tool.
2. Record the version and date next to uncertain material: `проверено 2026-11-21 на NeoForge 26.3.0.113`.
3. Tag anything not fully confirmed:
   - `[draft]` — official docs still in draft;
   - `[community]` — community practice, not first-party;
   - `[unverified]` — not confirmed in a running game.
4. Name the namespace (Mojmap vs Yarn) when names could differ, and scope code to a version.
5. Treat code as illustrative unless its page says otherwise. Code blocks document an idea; they are not a build target of this repository.

Absence of a tag means the claim is backed by the cited source and the page's `Проверено на` block. Tags mark uncertainty only; there is no `[verified]` tag to add.

**Completion criterion:** every non-obvious claim has a link or a tag, and no API name was written from memory without a check.

---

## Writing rules

- Follow the page template in `README.md` → "Конвенции страницы": H1 → breadcrumb → TL;DR callout → contents → sections → "Проверено на" → "Источники". Content pages must end with both blocks; meta pages (`README.md`, `AGENTS.md`, `00-glossary.md`, `07-sources.md`) may omit them because they navigate or aggregate instead of asserting new facts.
- One concept, one home. Cross-link instead of duplicating. The glossary holds definitions; the playbook holds procedure; the cookbook holds examples.
- Tables for comparisons, callouts (`> [!NOTE]`, `> [!WARNING]`) for traps, Mermaid only when a flow is genuinely non-linear.
- Keep Russian prose natural and direct. Do not translate API names, flags, or file paths.
- Write the positive instruction. "Use a targeted `edit`" beats "don't rewrite files".
- No filler intros, no "in this section we will". Start with the fact.

---

## Workflows

Each workflow ends with a **completion criterion**. Do not stop before it is met.

### A. Edit an existing page

1. Run `rg -n` for the term; read only the matching section.
2. Check the Canon and the page's "Проверено на" block.
3. Make a targeted `edit`. Preserve the surrounding style and links.
4. If the change affects a fact repeated elsewhere, `rg` the old value and update every copy.
5. Run the [verification](#verification) commands.
6. Commit one logical change.

**Done when:** the diff is minimal, all copies agree, links resolve, and the page still follows the template.

### B. Add a new page

1. Confirm the topic is not already covered by an existing page; extend that page instead if it is.
2. Add the file under the right folder and give it the standard template.
3. Add a row to the `README.md` file table and to the [Project map](#project-map) here, with a real "reach for it when" trigger.
4. Add sources to `07-sources.md` if the page introduces new primary sources.
5. Cross-link from at least one related page, so the new page is reachable.
6. Verify and commit.

**Done when:** the page is reachable from README and the map, has sources, and no orphan copy of existing facts.

### C. New Minecraft drop (26.4+)

1. Read the official release notes and the NeoForge/Fabric primers for the drop.
2. Update the Canon table and bullets first.
3. Update `05-version-changes/00-index.md` (timeline, era table).
4. Update the era page (`6-unobfuscated.md` for 26.x) with a "what changed" section.
5. Update affected recipes in `03-mixin-cookbook.md` and procedures in `02-mixin-playbook.md`.
6. Bump versions in build docs and the tool matrix only where the source confirms them.
7. Run the [refresh checklist](07-sources.md#что-перепроверить-после-264) from `07-sources.md`.

**Done when:** no page claims the previous drop is current, and old values are gone (`rg` for them).

### D. Factual fix

1. Find the owning page via the map.
2. Check the primary source; if the source is ambiguous, keep the existing claim and add `[unverified]` rather than guessing.
3. Fix the claim and every copy.
4. If the error came from a wrong mental model, add one line to the page explaining the trap — do not rewrite the page.

**Done when:** the source is cited in the diff description and the old claim no longer appears.

### E. Delegate to a subagent

Give the agent:

- the exact file paths it may write;
- "do not run git; do not touch other files";
- the source rules and the template pointer;
- a completion criterion: every claim sourced or tagged.

After it returns: review every risky fact, fix fabrications, run verification, then commit. A subagent's output is a draft, not a finished page.

### F. Verify code examples in practice (on request)

**Trigger:** the user explicitly asks to verify, compile, or test the examples. The default workflow treats code as illustrative; this protocol temporarily replaces that assumption for the requested scope.

**Setup**

1. Create `tests/` in the repository root. It is gitignored: never stage it, never link documentation to it, never ship it.
2. Use **one Gradle project per (loader, Minecraft version) pair**: `tests/neoforge-26.3`, `tests/fabric-1.21.1`, `tests/forge-1.20.1`. Plugins, toolchains and mappings conflict across eras; separate builds keep each one honest.
3. Start from the official MDK/template for that version (see `01-toolchain/1-build-systems.md`), pin exact versions, and use the matching toolchain: 8 for legacy, 16 for 1.17, 17 for 1.18–1.20.4, 21 for 1.20.5–1.21.11, 25 for 26.x.
4. Copy the documented code **verbatim first**. Add only imports, package, and the minimal class shell. If it does not compile as written, that is a finding, not a reason to silently rewrite it.

**Inventory**

List every fenced block in scope: file, line, language, claimed version/loader, kind (mixin, plain Java, JSON, Gradle, shell, pseudo-code). `rg -n '^```' -g '*.md'` gives the candidates; record the table before touching any build.

Classify each block:

| Kind | Check |
| --- | --- |
| Java / mixin | Compiles against the pinned stack |
| JSON (`pack.mcmeta`, models, recipes, `mixins.json`) | Parses, correct schema, loads in game where relevant |
| Gradle / properties | Applies, task exists, build succeeds |
| Shell | Runs in a clean checkout of the documented layout |
| Pseudo-code | Cannot compile; mark `illustrative` in the doc |

**Compile**

- One source file per example, package `dev.docscheck.<doc>.<n>`; keep the documented names.
- A mixin compiles only if its target, descriptor and `@Shadow` members exist on the pinned version. Compilation alone does not prove an `@At` target — see the behavior step.
- `./gradlew build` per project. Record the exact task and outcome.

**Behavior**

Define the expected observation **before** running; a run without an expectation is not a test.

- Pure logic → JUnit.
- Game logic → GameTest (Fabric/NeoForge) or a small test mod with assertions.
- Mixins → launch with `-Dmixin.debug.export=true -Dmixin.debug.countInjections=true`; assert the apply log line and the changed behavior. For GUI, verify manually or via the loader's screenshot/test hooks.
- Destructive examples (world edits, drops, spawns) run on a disposable world only.

**Fix loop**

When a check contradicts the doc:

1. Fix the **documentation**, not the harness, unless the harness is demonstrably wrong.
2. Update the page's `Проверено на` and, if the status changed, the README verification table.
3. Keep harness changes out of git history; commit only doc corrections.
4. If an example cannot be exercised (needs a GPU, a player, a specific mod), mark it `[unverified]` with the reason instead of claiming success.

**Report**

Return a table: sample → version/loader → compile → behavior → doc action (none / fixed / tagged / blocked). Name the exact versions and commands used.

**Done when:** every block in scope has a status (compiled, behavior verified, blocked, not applicable), and every divergence is either fixed in the docs or tagged with its reason.

### G. Audit documentation for fabrications (on request)

**Trigger:** the user asks to re-check correctness or to look for invented content. Scope is the files the user names; with no scope, audit all docs, high-risk sections first.

**Priority order.** Spend effort where a model is most likely to invent:

1. Exact signatures, descriptors and parameter order.
2. Tool APIs: names of MCP tools, Gradle properties, flags, CLI commands.
3. Versions, dates, Java requirements.
4. Error strings and log lines.
5. Paths, package names, file names.
6. 26.x claims, then 1.21.x, then older eras.

**Steps**

1. Build a claim inventory per file: every testable statement becomes a row (claim, location, source, verdict).
2. Verify against the source that owns the fact: game/loader source, javadoc, official docs, maven-metadata, release notes. Record the URL or file path and the access date.
3. Assign one verdict:
   - `confirmed` — matches the owner source;
   - `wrong` — contradicts it;
   - `stale` — was true for an older version;
   - `unverifiable` — no primary source found;
   - `community` — only community sources support it.
4. Act on the verdict:
   - `wrong` → fix the claim and every copy (`rg` the old value);
   - `stale` → update or scope it to the version where it was true;
   - `unverifiable` → narrow the claim and add `[unverified]`; never upgrade it to fact;
   - `community` → keep the `[community]` tag and cite the source.
5. Cross-check the [Canon](#canon) and the `Проверено на` blocks. Contradictions between pages are findings too.
6. Run the mechanical checks under [Verification](#verification): links, CJK, whitespace, anchors.
7. Commit fixes atomically: one commit per corrected claim or tight group, message names the wrong fact.

**Anti-patterns**

- Asking another model, or reading a blog that repeats the same page, is not verification. Only the owner source counts.
- "It compiles" does not validate a mixin target. Use protocol F for behavior.
- "It is already in this repository" is not evidence.
- Adding a note on top of an obviously wrong name instead of fixing it.
- Re-auditing everything on every run: scope first, then go deep.

**Report**

Return a table: claim → verdict → source → action → commit. State explicitly what remains `[unverified]` and why.

**Done when:** every high-risk claim in scope has a verdict, every `wrong`/`stale` claim is fixed, every remaining uncertainty is tagged, and the mechanical checks pass.

---

## Verification

Run from the repository root.

Relative link check (must print `missing: 0`):

```bash
python3 - <<'EOF'
import os, re
missing = {}
for dirpath, _, files in os.walk('.'):
    if '.git' in dirpath:
        continue
    for f in files:
        if not f.endswith('.md'):
            continue
        p = os.path.join(dirpath, f)
        txt = open(p, encoding='utf-8').read()
        for m in re.finditer(r'\]\(([^)#]+\.md)(#[^)]*)?\)', txt):
            t = m.group(1)
            if t.startswith('http'):
                continue
            tp = os.path.normpath(os.path.join(dirpath, t))
            if not os.path.exists(tp):
                missing.setdefault(tp, []).append(p)
for k, v in sorted(missing.items()):
    print(k, '<-', ', '.join(sorted(set(v))))
print('missing:', len(missing))
EOF
```

Other checks:

```bash
git diff --check                                  # whitespace errors
rg -nP '[\x{4e00}-\x{9fff}]' -g '*.md'            # stray CJK characters
rg -n 'TODO:|TBD|FIXME' -g '*.md'                 # unfinished text
```

Also confirm by eye: template order, TL;DR present, `Проверено на` and `Источники` at the end, tables render (no broken pipes).

### Code examples

Code blocks are documentation, not a build target: no example in this repo is compiled or run by the documentation workflow itself. That does not lower the bar for accuracy — it moves the check earlier, to the source.

When adding or changing code:

1. **Pin the context.** State the Minecraft version and loader, because the same mixin differs between 1.20.1, 1.21.x, and 26.x.
2. **Verify signatures before writing, not after.** Check the descriptor against the game/loader source, javadoc, or `javap -s`. A plausible-looking parameter list is the most common fabrication.
3. **Mark uncertainty.** If a signature, method name, or parameter order is not confirmed, put `[unverified]` next to the example and say which part is uncertain.
4. **Keep it minimal.** Show the annotation, the handler, and the one line that matters. Omit imports and boilerplate unless they carry the point.
5. **Add a verification hint where it helps:** `javap -s`, IDE `Ctrl+Click`, `-Dmixin.debug.export=true` + `.mixin.out/class`.
6. **If a sample was actually compiled or tested**, say so explicitly in the page and name the environment. Do not generalize that result to other samples.

Update the README verification table when the status changes (for example, "examples compiled on 26.3" versus "not compiled"). The table describes the current state; this section describes the rule.

---

## Git

- Conventional Commits, English, imperative mood: `docs(mixins): add target-discovery playbook`.
- Scopes: `toolchain`, `mixins`, `versions`, `loaders`, `sources`, `glossary`, `meta`.
- One logical change per commit. Do not mix a version bump with a rewrite.
- Body explains **why**, not what; the diff already shows what.
- Stage explicit paths; never `git add .`.
- Never commit with broken relative links or stray CJK.
- Do not rewrite history on `main`; this repo is published.

**Done when:** `git log --oneline` reads as a changelog a human can follow.

---

## Definition of done

A task is complete only when all of these are true:

- [ ] The change lives in the file that owns the fact.
- [ ] Canon facts match; no stale copies remain.
- [ ] Every new claim has a source or an uncertainty tag.
- [ ] Relative links resolve; `missing: 0`.
- [ ] The page follows the template.
- [ ] The diff is minimal and readable.
- [ ] One atomic commit exists.

---

## Escalation

Stop and ask when:

- two primary sources disagree and neither is clearly newer;
- a change would alter the Canon (a rename, a new Java requirement, a loader split);
- the user asks for a claim you cannot source;
- a task implies deleting or rewriting a page someone else may be editing.

Guessing is the failure mode this repo is built to avoid. An explicit `[unverified]` is always better than a confident invented API.

> [!NOTE]
> The repository-level `AGENTS.md` you may see in the home directory mentions `code-review-graph` MCP tools. This repo contains no code graph; those tools add nothing here. Use the map, `rg`, and the primary sources.
