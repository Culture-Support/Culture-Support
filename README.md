
<img width="1024" height="576" alt="image" src="https://github.com/user-attachments/assets/7e24de50-18b2-424d-8a0f-a01192ce7728" />


```

┌─────────────────────────┐
│     User / Task Prompt  │
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│   Claude Agent Router   │
└────────────┬────────────┘
      ┌──────┴──────┐
      ▼             ▼
┌───────────┐ ┌───────────┐
│ Claude MCP│ │ Ollama AI │
│ Tools/API │ │ Local LLM│
└─────┬─────┘ └─────┬─────┘
      └──────┬──────┘
             ▼
┌─────────────────────────┐
│   CodePen UI Preview    │
└─────────────────────────┘

```


----------------------------

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Observability & Telemetry                       │
│           (OpenTelemetry, Langfuse, Tracing, Token Accounting)         │
└────────────────────────────────────┬───────────────────────────────────┘
                                     │
┌──────────────┐     ┌───────────────▼──────────────┐     ┌──────────────┐
│ User Request ├────►│  Input Guardrails & Security ├────►│ Context & RAG│
│  & Metadata  │     │ (Sanitization, Injection, PI)│     │(Hybrid + Re) │
└──────────────┘     └──────────────────────────────┘     └──────┬───────┘
                                                                 │
┌──────────────┐     ┌──────────────────────────────┐     ┌──────▼───────┐
│ Client / App │◄────┤ Output Guardrails & Parsing  │◄────┤ Model Engine │
│   Response   │     │ (Pydantic Schema, Halluc.)   │     │ (API / vLLM) │
└──────────────┘     └──────────────────────────────┘     └──────────────┘

```

------------------------

``` toml

name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # -------------------------------------------------------------
  # STAGE 1: Parse and Emit TOML Configuration
  # -------------------------------------------------------------
  configure:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.parse.outputs.matrix }}
      package_manager: ${{ steps.parse.outputs.package_manager }}
      lint_script: ${{ steps.parse.outputs.lint_script }}
      typecheck_script: ${{ steps.parse.outputs.typecheck_script }}
      test_script: ${{ steps.parse.outputs.test_script }}
      build_script: ${{ steps.parse.outputs.build_script }}
      output_dir: ${{ steps.parse.outputs.output_dir }}
      deploy_enabled: ${{ steps.parse.outputs.deploy_enabled }}
      deploy_target: ${{ steps.parse.outputs.deploy_target }}
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Node.js for Parsing
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Parse pipeline.toml
        id: parse
        run: |
          npm install --no-save smol-toml
          node .github/scripts/parse-config.mjs

  # -------------------------------------------------------------
  # STAGE 2: Matrix Verification & Build
  # -------------------------------------------------------------
  verify:
    needs: configure
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix: ${{ fromJson(needs.configure.outputs.matrix) }}

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Setup pnpm (if required)
        if: needs.configure.outputs.package_manager == 'pnpm'
        uses: pnpm/action-setup@v3
        with:
          version: latest

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node_version }}
          cache: ${{ needs.configure.outputs.package_manager }}

      - name: Install Dependencies
        run: |
          if [ "${{ needs.configure.outputs.package_manager }}" = "pnpm" ]; then
            pnpm install --frozen-lockfile
          elif [ "${{ needs.configure.outputs.package_manager }}" = "yarn" ]; then
            yarn install --frozen-lockfile
          else
            npm ci
          fi

      - name: Run Lint & Static Analysis
        run: ${{ needs.configure.outputs.package_manager }} run ${{ needs.configure.outputs.lint_script }}

      - name: Run Type Check
        run: ${{ needs.configure.outputs.package_manager }} run ${{ needs.configure.outputs.typecheck_script }}

      - name: Execute Tests
        run: ${{ needs.configure.outputs.package_manager }} run ${{ needs.configure.outputs.test_script }}

      - name: Execute Build
        run: ${{ needs.configure.outputs.package_manager }} run ${{ needs.configure.outputs.build_script }}

      - name: Upload Build Artifacts
        if: matrix.node_version == '20' # Only upload artifact once from primary LTS
        uses: actions/upload-artifact@v4
        with:
          name: build-artifact
          path: ${{ needs.configure.outputs.output_dir }}
          retention-days: 3

  # -------------------------------------------------------------
  # STAGE 3: Conditional Deployment
  # -------------------------------------------------------------
  deploy:
    needs: [configure, verify]
    if: needs.configure.outputs.deploy_enabled == 'true' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Download Build Artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-artifact
          path: dist

      - name: Dispatch Deployment
        run: |
          echo "Deploying target: ${{ needs.configure.outputs.deploy_target }}"
          # Integrate chosen deployment utility (Wrangler, AWS CLI, Vercel, NPM publish, etc.)

```

``` javascript

// .github/scripts/parse-config.mjs
import fs from 'node:fs';
import { parse } from 'smol-toml';

const tomlContent = fs.readFileSync('pipeline.toml', 'utf-8');
const config = parse(tomlContent);

// Prepare matrix object for GitHub Actions
const matrixOutput = {
  node_version: config.runtime.versions,
  include: []
};

// Export to GITHUB_OUTPUT environment file
const outputFile = process.env.GITHUB_OUTPUT;

function appendOutput(key, value) {
  const formatted = typeof value === 'object' ? JSON.stringify(value) : value;
  fs.appendFileSync(outputFile, `${key}=${formatted}\n`);
}

appendOutput('matrix', matrixOutput);
appendOutput('package_manager', config.runtime.package_manager);
appendOutput('lint_script', config.pipeline.quality.lint_script);
appendOutput('typecheck_script', config.pipeline.quality.typecheck_script);
appendOutput('test_script', config.pipeline.test.unit_script);
appendOutput('build_script', config.pipeline.build.build_script);
appendOutput('output_dir', config.pipeline.build.output_dir);
appendOutput('deploy_enabled', config.deployment.enabled);
appendOutput('deploy_target', config.deployment.target);

console.log('Successfully generated pipeline parameters from pipeline.toml');

```

```
Jesus Raises Beliefs
```

```
Jesus sending words trough bible

```

-----------------


