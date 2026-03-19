# iAPI Skills Eval Framework — Implementation Plan

## Goal

An eval framework that validates the Interactivity API skills produce correct code when given to any LLM model.

## Pipeline per test case

1. **Generate** — Call a model with the skill files as system context and the prompt as user message. No extra instructions — replicates a real user scenario exactly.
2. **Extract** — Lightweight LLM call to extract `block.json`, `render.php`, and `view.js` from the raw response. All three files must be present; if any is missing, the test case fails immediately.
3. **Judge (LLM-as-judge)** — Feed the extracted code + skill reference files + case-specific expected patterns to a judge model. The judge checks against best practices from the skill references (no separate best practices file). Uses a single global system prompt (`judge-prompt.md`).
4. **Scaffold** — Place the extracted files into a static WordPress plugin skeleton (`iapi-eval`). No dynamic slugs — the plugin template is copied as-is and generated files are dropped in.
5. **E2E** — Deploy the plugin into wp-env, run Playwright tests to verify it works.

## Key decisions

- **Language**: TypeScript + Node.js (single runtime for wp-env, Playwright, LLM SDKs)
- **WordPress env**: `@wordpress/env` (wp-env) — official, Docker-based, battle-tested with Playwright in Gutenberg
- **No build step**: Script modules work natively since WordPress 6.5+ (no wp-scripts needed)
- **Best practices**: Fed directly from existing skill reference files to the judge — no separate checklist file
- **Judge system prompt**: A single global file (`judge-prompt.md`) at the repo root, shared by all test cases
- **Code extraction**: Separate lightweight LLM call (can use a cheap model) to extract files from the raw model output — more reliable than regex parsing
- **Realistic generation**: No role preamble or output format instructions added to the generation prompt — only the skill files and the user prompt, exactly as a real user would experience
- **Static plugin template**: Fixed slug (`iapi-eval`), no placeholder replacement. Test cases run sequentially with cleanup between them.
- **E2E**: Playwright
- **Test case discovery**: File-based (scan `test-cases/` for directories with `prompt.md`)
- **Results**: JSON artifacts saved to disk, console table summary
- **First example**: Counter block

## Directory structure

```
test-iapi-skills/
├── package.json
├── tsconfig.json
├── playwright.config.ts
├── .env.example                        # API keys for LLM providers
├── .gitignore
├── Plan.md                             # This file
│
├── wp-interactivity-api/               # Existing skill files (already in repo)
│   ├── SKILL.md
│   └── references/
│       ├── directives-and-store.md
│       ├── server-side-rendering.md
│       ├── the-reactive-and-declarative-mindset.md
│       ├── undestanding-global-state-local-context-and-derived-state.md
│       ├── client-side-navigation.md
│       └── using-typescript.md
│
├── src/
│   ├── runner.ts                       # Main CLI entry point / orchestrator
│   ├── generate.ts                     # Step 1: call model with skills + prompt
│   ├── extract.ts                      # Step 2: LLM call to extract files from raw output
│   ├── judge.ts                        # Step 3: LLM-as-judge evaluation
│   ├── scaffold.ts                     # Step 4: copy plugin template + drop in generated files
│   ├── e2e.ts                          # Step 5: deploy plugin, run Playwright
│   ├── report.ts                       # Aggregate results, print summary
│   ├── models/
│   │   ├── index.ts                    # Model provider registry/factory
│   │   ├── anthropic.ts               # Anthropic adapter
│   │   ├── openai.ts                  # OpenAI adapter
│   │   └── types.ts                   # Common ModelProvider interface
│   └── utils/
│       ├── skill-loader.ts            # Read and assemble skill files into context
│       └── wp-env.ts                  # wp-env lifecycle helpers
│
├── judge-prompt.md                     # Global system prompt for the judge model
│
├── plugin-template/                    # Static WP plugin skeleton (copied as-is)
│   ├── iapi-eval.php                   # Plugin entry file (registers the block)
│   └── src/
│       └── blocks/
│           └── eval-block/             # Generated files are placed here
│
├── test-cases/
│   └── counter/
│       ├── prompt.md                   # The generation prompt
│       ├── expected-patterns.yaml      # Case-specific patterns for the judge
│       └── e2e.spec.ts                # Playwright test
│
├── .wp-env.json                        # wp-env configuration
│
└── results/                            # Git-ignored, generated at runtime
    └── {run-id}/
        ├── summary.json                # Aggregate pass/fail
        └── counter/
            ├── generated-code.md       # Raw LLM output
            ├── generated-files/        # Extracted files (render.php, view.js, etc.)
            ├── plugin/                 # Scaffolded WP plugin
            ├── judge-result.json       # Judge verdict + per-pattern reasoning
            └── e2e-result.json         # Playwright pass/fail + failure details
```

## Plugin template

The `plugin-template/` is completely static — copied as-is for every test case, no placeholder replacement.

**`iapi-eval.php`**:
```php
<?php
/**
 * Plugin Name: iAPI Eval
 */
function iapi_eval_init() {
    register_block_type( __DIR__ . '/src/blocks/eval-block' );
}
add_action( 'init', 'iapi_eval_init' );
```

WordPress reads the `block.json` inside `src/blocks/eval-block/` and handles everything from there. The model-generated `block.json` is expected to include:
- `"render": "file:./render.php"` — tells WordPress to use render.php for server-side rendering
- `"viewScriptModule": "file:./view.js"` — tells WordPress to load view.js as a script module
- `"supports": { "interactivity": true }` — enables server directive processing

No `render_callback` is needed in the PHP file. Getting these `block.json` fields right is itself part of what we're evaluating.

## Implementation steps

### Step 1: Project setup

- Initialize `package.json` with TypeScript, Playwright, wp-env, and LLM SDK dependencies
- Create `tsconfig.json`
- Create `.env.example` with `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `JUDGE_MODEL`, `MODEL`
- Create `.gitignore` (ignore `results/`, `.env`, `node_modules/`)
- Create `.wp-env.json` pointing to the plugin directory
- Create the static `plugin-template/` directory with `iapi-eval.php`

### Step 2: Model provider abstraction (`src/models/`)

- Define `ModelProvider` interface: `generate(systemPrompt, userPrompt, config) → string`
- Implement `AnthropicProvider` using `@anthropic-ai/sdk`
- Implement `OpenAIProvider` using `openai`
- Create factory in `index.ts` that selects provider based on model name

### Step 3: Generation step (`src/generate.ts`)

- `skill-loader.ts`: Read SKILL.md + all must-have reference files, concatenate them
- Send to the model under test exactly as the skill system would:
  - **System message**: The skill files (SKILL.md + must-have references) — nothing else
  - **User message**: The test case's `prompt.md` — nothing else
- No role preamble, no output format instructions — this replicates the real user experience
- Configurable LLM parameters (`temperature`, `maxTokens`, `timeout`) with sensible defaults TBD. Retry with exponential backoff on transient errors (429, 5xx).
- Save raw output to `results/{run-id}/{case}/generated-code.md`

### Step 4: Extraction step (`src/extract.ts`)

- Make a lightweight LLM call (can use a cheap/fast model) with the raw model output
- Configurable LLM parameters (`temperature`, `maxTokens`, `timeout`) with sensible defaults TBD. Retry with exponential backoff on transient errors (429, 5xx).
- Prompt: "Extract the block.json, render.php, and view.js files from this response. Return them as structured JSON."
- Expected response:
  ```json
  {
    "files": {
      "block.json": "...",
      "render.php": "...",
      "view.js": "..."
    }
  }
  ```
- **All three files must be present.** If any file is missing or the extraction fails, the test case is marked as a generation failure and all subsequent steps (judge, scaffold, e2e) are skipped.
- Save extracted files to `results/{run-id}/{case}/generated-files/`

### Step 5: Judge step (`src/judge.ts`)

- Load the global `judge-prompt.md` as the judge system prompt
- Assemble judge input: skill reference files (as best practices source) + case-specific `expected-patterns.yaml` + extracted code files
- Configurable LLM parameters (`temperature`, `maxTokens`, `timeout`) with sensible defaults TBD. Retry with exponential backoff on transient errors (429, 5xx).
- Call the judge model requesting structured JSON output:
  ```json
  {
    "pass": true,
    "patterns": [
      {
        "id": "uses-data-wp-text",
        "pass": true,
        "reasoning": "render.php uses data-wp-text=\"context.counter\" to display the count"
      }
    ],
    "best_practices": [
      {
        "rule": "async actions must use generators",
        "pass": true,
        "severity": "error",
        "reasoning": "No async actions in this code, so not applicable"
      }
    ],
    "summary": "The generated code correctly implements a counter block..."
  }
  ```
  - `pass`: Overall verdict
  - `patterns`: One entry per item from `expected-patterns.yaml` — did the model follow the expected pattern?
  - `best_practices`: Rules the judge identifies from the skill references — `error` severity means it would break things, `warning` is a style issue
  - `summary`: Free-text explanation for debugging failures
- Pass/fail logic: overall pass = all `error`-severity best practice checks pass AND all expected patterns pass
- Save result to `results/{run-id}/{case}/judge-result.json`

### Step 6: Plugin scaffolding (`src/scaffold.ts`)

- Place the extracted `block.json`, `render.php`, and `view.js` directly into `plugin-template/src/blocks/eval-block/` — this is the static path that `.wp-env.json` points to, so wp-env always sees the plugin without dynamic mounting
- Also copy the files to `results/{run-id}/{case}/generated-files/` for archival

### Step 7: E2E step (`src/e2e.ts`)

- `wp-env.ts`: Helpers to start/stop wp-env, activate plugins, create test pages via WP-CLI
- For each test case:
  1. Activate the plugin via WP-CLI (already mounted via the static `plugin-template/` path)
  2. Verify activation succeeded (check `wp plugin list`) — if it failed (e.g., PHP fatal in generated code), skip E2E and record the error
  3. Create a test page containing `<!-- wp:iapi-eval/eval-block /-->` via WP-CLI (`render.php` handles all server-side output)
  4. Run the case-specific Playwright spec
  5. Collect pass/fail results and failure details (capture screenshots and traces on failure via Playwright config)
  6. Clean up (deactivate plugin, delete test page)
- Save result to `results/{run-id}/{case}/e2e-result.json`

### Step 8: Runner / orchestrator (`src/runner.ts`)

- CLI entry point with flags: `--model`, `--case`, `--skip-e2e`, `--judge-model`
- Discover test cases by scanning `test-cases/*/prompt.md`
- For each test case: run generate → extract → judge → scaffold → e2e (sequentially)
- Aggregate results into `results/{run-id}/summary.json`

### Step 9: Reporting (`src/report.ts`)

- Print console table:
  ```
  Test Case    | Generate | Judge | E2E  | Overall
  counter      | OK       | PASS  | PASS | PASS
  ```
- On failure: print failure reasons and path to generated code
- Overall pass = judge pass AND e2e pass
- Exit code 0 if all pass, 1 if any fail

### Step 10: Counter test case

- Write `test-cases/counter/prompt.md`: "Build a counter block with +/- buttons using the Interactivity API, with independent instances via data-wp-context"
- Define the `expected-patterns.yaml` schema (fields like `id`, `description`, `file`, `severity`) and document it so future test case authors know what to write
- Write `test-cases/counter/expected-patterns.yaml`: Case-specific patterns (uses data-wp-text, uses data-wp-on--click, uses getContext(), etc.)
- Write `test-cases/counter/e2e.spec.ts`: Click +, assert count increments; click -, assert count decrements; multiple instances are independent
- Write `judge-prompt.md`: Global system prompt instructing the judge to evaluate code against the skill references and case-specific patterns

### Step 11: End-to-end testing

- Run the full pipeline manually against at least one model
- Verify: generation produces extractable code, extraction works, judge evaluates correctly, plugin scaffolds and activates, Playwright tests pass
- Fix any issues in the pipeline

### Step 12: Local development guide

- Write a guide covering: prerequisites (Node.js, Docker, API keys), setup instructions, how to run the pipeline, and how to add new test cases

## CLI usage (v1)

```bash
# Install
npm install

# Configure
cp .env.example .env  # Add API keys

# Start WordPress environment (requires Docker)
npx wp-env start

# Run all test cases
npx tsx src/runner.ts --model claude-sonnet-4-20250514

# Run a single test case
npx tsx src/runner.ts --model claude-sonnet-4-20250514 --case counter

# Skip e2e (faster iteration)
npx tsx src/runner.ts --model gpt-4o --skip-e2e

# Use a different judge model
npx tsx src/runner.ts --model gpt-4o --judge-model claude-sonnet-4-20250514
```

## Follow-ups (not in v1)

- **`--steps` flag**: Run only specific pipeline stages (e.g., `--steps generate`, `--steps judge`, `--steps e2e`). Useful for debugging individual stages without re-running the entire pipeline.
- **`reeval` command**: Re-run judge and/or e2e against code from a previous run without calling the model again. Saves API costs when iterating on judge prompt, e2e specs, or skill references.
- **`_template/` directory**: A copyable template directory under `test-cases/` to make adding new test cases even easier.
- **GitHub Actions CI workflow**: Automated eval runs on schedule or on changes to skill files, with results uploaded as artifacts.
- **Dynamic plugin slugs**: Needed if parallel test case execution is added later.
