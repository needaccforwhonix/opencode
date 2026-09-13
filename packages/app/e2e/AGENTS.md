# E2E Testing Guide

## Build/Lint/Test Commands

```bash
# Run all e2e tests
bun test:e2e

# Run specific test file
bun test:e2e -- app/home.spec.ts

# Run single test by title
bun test:e2e -- -g "home renders and shows core entrypoints"

# Run tests with UI mode (for debugging)
bun test:e2e:ui

# Run tests locally with full server setup
bun test:e2e:local

# View test report
bun test:e2e:report

# Typecheck
bun typecheck
```

## Test Structure

All tests live in `packages/app/e2e/`:

```
e2e/
├── fixtures.ts       # Test fixtures (test, expect, gotoSession, sdk)
├── actions.ts        # Reusable action helpers
├── selectors.ts      # DOM selectors
├── utils.ts          # Utilities (serverUrl, modKey, path helpers)
└── [feature]/
    └── *.spec.ts     # Test files
```

## Test Patterns

### Basic Test Structure

```typescript
import { test, expect } from "../fixtures"
import { promptSelector } from "../selectors"
import { withSession } from "../actions"

test("test description", async ({ page, sdk, gotoSession }) => {
  await gotoSession() // or gotoSession(sessionID)

  // Your test code
  await expect(page.locator(promptSelector)).toBeVisible()
})
```

### Using Fixtures

- `page` - Playwright page
- `sdk` - OpenCode SDK client for API calls
- `gotoSession(sessionID?)` - Navigate to session

### Helper Functions

**Actions** (`actions.ts`):

- `openPalette(page)` - Open command palette
- `openSettings(page)` - Open settings dialog
- `closeDialog(page, dialog)` - Close any dialog
- `openSidebar(page)` / `closeSidebar(page)` - Toggle sidebar
- `withSession(sdk, title, callback)` - Create temp session
- `clickListItem(container, filter)` - Click list item by key/text

**Selectors** (`selectors.ts`):

- `promptSelector` - Prompt input
- `terminalSelector` - Terminal panel
- `sessionItemSelector(id)` - Session in sidebar
- `listItemSelector` - Generic list items

**Utils** (`utils.ts`):

- `modKey` - Meta (Mac) or Control (Linux/Win)
- `serverUrl` - Backend server URL
- `sessionPath(dir, id?)` - Build session URL

## Code Style Guidelines

### Imports

Always import from `../fixtures`, not `@playwright/test`:

```typescript
// ✅ Good
import { test, expect } from "../fixtures"

// ❌ Bad
import { test, expect } from "@playwright/test"
```

### Naming Conventions

- Test files: `feature-name.spec.ts`
- Test names: lowercase, descriptive: `"sidebar can be toggled"`
- Variables: camelCase
- Constants: SCREAMING_SNAKE_CASE

### Error Handling

Tests should clean up after themselves:

```typescript
test("test with cleanup", async ({ page, sdk, gotoSession }) => {
  await withSession(sdk, "test session", async (session) => {
    await gotoSession(session.id)
    // Test code...
  }) // Auto-deletes session
})
```

### Timeouts

Default: 60s per test, 10s per assertion. Override when needed:

```typescript
test.setTimeout(120_000) // For long LLM operations
test("slow test", async () => {
  await expect.poll(() => check(), { timeout: 90_000 }).toBe(true)
})
```

### Selectors

Use `data-component`, `data-action`, or semantic roles:

```typescript
// ✅ Good
await page.locator('[data-component="prompt-input"]').click()
await page.getByRole("button", { name: "Open settings" }).click()

// ❌ Bad
await page.locator(".css-class-name").click()
await page.locator("#id-name").click()
```

### Keyboard Shortcuts

Use `modKey` for cross-platform compatibility:

```typescript
import { modKey } from "../utils"

await page.keyboard.press(`${modKey}+B`) // Toggle sidebar
await page.keyboard.press(`${modKey}+Comma`) // Open settings
```

## Writing New Tests

1. Choose appropriate folder or create new one
2. Import from `../fixtures`
3. Use helper functions from `../actions` and `../selectors`
4. Clean up any created resources
5. Use specific selectors (avoid CSS classes)
6. Test one feature per test file

## Local Development

For UI debugging, use:

```bash
bun test:e2e:ui
```

This opens Playwright's interactive UI for step-through debugging.



## 🛡️ Strict Embedding Separation, Zero-Fallback Law & 24/7 Dual-GPU Invariant
- **Reference**: `/home/m1st/.agents/rules/RULE_Strict_Embedding_Separation_And_Dual_Pipeline.md`

### 1. Das Absolute Fallback-Verbot (Zero-Fallback Law)
Unter keinen Umständen, zu keinem Zeitpunkt und aus keinem Grund darf ein Fallback zwischen verschiedenen Embedding-Modellen stattfinden.
* **Geltende Aktion:** Fällt ein Embedding-Modell aus oder ist überlastet, MUSS die Operation sofort hart fehlschlagen (`Fail-Fast`) oder die Payload transaktional in einer Queue (NATS/SQLite) verharren, bis das exakte Modell bereit ist.
* **Verboten:** Kein stiller oder dynamischer Modellwechsel (weder Jina -> Gemma noch umgekehrt).

### 2. Warum ein Embedding-Fallback mathematisch & informationstheoretisch unmöglich ist
* **Topologische Inkompatibilität heterogener Vektorräume (Non-Isomorphism):**
  Jedes Modell $f_\theta: \mathcal{X} \to \mathbb{R}^D$ projiziert Text in eine spezifische, gelernte Riemannsche Mannigfaltigkeit. Jina v5 ($D=256$) und EmbeddingGemma ($D=768$) spannen zwei völlig inkompatible geometrische Räume auf. Die Basisvektoren der semantischen Achsen sind ohne explizite Procrustes-Transformation nicht ausgerichtet.
* **Kollaps der Kosinus-Ähnlichkeit ($	ext{sim} \approx 0$):**
  Wird eine Suchanfrage mit Modell $B$ berechnet ($v_q = f_B(q)$), während der Dokumentenkorpus mit Modell $A$ indiziert wurde ($v_d = f_A(d)$), verhält sich das Skalarprodukt mathematisch wie das zweier rein zufälliger Vektoren auf einer hochdimensionalen Einheitssphäre:
  $$\mathbb{E}[\text{sim}(u, v)] = 0 \quad \text{mit Varianz} \quad \sigma^2 = \frac{1}{D}$$
  Der Nearest-Neighbor-Algorithmus (HNSW/k-NN) liefert stochastisches Rauschen. Das RAG-System erhält völlig falsche oder irrelevante Kontexte.
* **Irreversible Index-Vergiftung (Index Poisoning):**
  Wird auch nur ein einziger Vektor von Modell $B$ als "Fallback" in den Index von Modell $A$ geschrieben, verunreinigt er die Distanzgraphen und Clusterzentren dauerhaft.
* **Das Gesetz des Fail-Fast:**
  Ein Ausfall muss hart abbrechen (`HTTP 503 Service Unavailable / IngestionQueueBlocked`).

### 3. Duale 24/7 Erfassungspflicht (GPU-Only)
* **GPU-Only Mandat:** Es läuft absolut nichts auf der CPU — GPU ONLY (NVIDIA GB10 CUDA) für ausnahmslos jedes Embedding-Modell.
* **24/7 Parallelität:** Sowohl `jina-embeddings-v5-omni-nano-classification` (256D, ~4,1 GB VRAM) als auch `google/embeddinggemma-300m` (768D, ~1,2 GB VRAM) laufen dauerhaft 24/7 im VRAM (Summe ~5,3 GB VRAM).
* **Duale Erfassung:** Jeder zu indizierende Text/Chunk wird immer von beiden Modellen parallel eingebettet und getrennt persistiert.

### 4. Idioten- & Failsafe-Sicherung auf Datenbankebene
* **SQLite Schema CHECK-Constraints:**
  `model_signature TEXT NOT NULL CHECK(model_signature = '...')` und `dimension INTEGER NOT NULL CHECK(dimension = ...)` erzwingen atomare Abbrüche auf Engine-Ebene bei Modell-Mismatches.
* **Qdrant Collection Constraints:**
  Strikte Trennung in separate Collections (`dgx_text_embeddings_jina_256` vs `dgx_text_embeddings_gemma_768`) mit fixierter Vektordimension.


## ⚡ High-Quality Systems Programming Languages Priority (No-Python Policy)
- **Reference**: `/home/m1st/.agents/rules/RULE_High_Quality_Systems_Programming_Languages.md`
- **Rule**:
  1. **Bevorzugte Sprachen:** High Quality **Golang (Go), Rust, C++, Zig, PowerShell, C** sind IMMER und AUSNAHMSLOS die bevorzugten Programmiersprachen.
  2. **Kein Python:** Python ist für neue Daemons, Watcher, Automatisierungen, APIs, CLI-Tools und Dienste strikt untersagt (GIL-Bottlenecks, Dependency-Drift, Speicherineffizienz).
  3. **Natives Systems-Engineering:** Alle Hintergrunddienste, Caching-Ebenen und Task-Runner müssen als native, speichersichere und nebenläufige Binaries kompiliert werden.
