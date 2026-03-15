# MCP Testing Framework
## Technical Guide: Architecture, Usage & Integration

> **Repository:** [github.com/L-Qun/mcp-testing-framework](https://github.com/L-Qun/mcp-testing-framework)
> **Version:** Latest &nbsp;|&nbsp; **Date:** March 2026 &nbsp;|&nbsp; **Prepared for:** Internal Company Use

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [What is MCP and Why Test It?](#2-what-is-mcp-and-why-test-it)
3. [Architecture](#3-architecture)
4. [Installation and Project Setup](#4-installation-and-project-setup)
5. [Configuration Reference](#5-configuration-reference)
6. [Running Tests](#6-running-tests)
7. [Writing Effective Test Cases](#7-writing-effective-test-cases)
8. [Custom Model Providers](#8-custom-model-providers)
9. [Comparison with Alternative Approaches](#9-comparison-with-alternative-approaches)
10. [Best Practices and Recommendations](#10-best-practices-and-recommendations)
11. [Troubleshooting](#11-troubleshooting)
12. [Quick Reference](#12-quick-reference)

---

## 1. Executive Summary

The **MCP Testing Framework** (`mcp-testing-framework`) is an open-source evaluation tool specifically designed to help development teams test and validate Model Context Protocol (MCP) servers. MCP is the standard protocol through which AI language models communicate with external tools, APIs, file systems, and databases.

A critical challenge in building MCP servers is that Large Language Models (LLMs) rely heavily on the names, descriptions, and parameter definitions you provide in your server when deciding when and how to call tools. Evaluating the effectiveness of these definitions has traditionally been a subjective, difficult-to-quantify process. This framework directly addresses that gap by providing a repeatable, data-driven testing methodology.

> **Key Problem Solved:** Teams building MCP servers often cannot objectively measure how well their tool definitions, descriptions, and parameter schemas will be understood by different AI models. Poor definitions lead to missed tool calls, incorrect parameter usage, and unreliable agent behaviour. The `mcp-testing-framework` provides quantitative metrics to surface and fix these issues before production.

### Feature Overview

| Feature | Description |
|---|---|
| **Multi-Model Testing** | Simultaneously evaluate your MCP server against OpenAI, Google Gemini, Anthropic Claude, and DeepSeek models. |
| **Batch Evaluation** | Run test suites multiple rounds and calculate statistical pass rates, removing noise from single-run evaluations. |
| **YAML-Driven Config** | Fully declarative test definitions — no code required to write test cases or server configurations. |
| **Pass Threshold Control** | Set a minimum acceptable pass rate (0.0–1.0) to gate CI/CD pipelines and catch regressions early. |
| **Custom Model Support** | Register your own model providers via code to extend testing to any LLM via a simple provider interface. |
| **Multi-Server Support** | Configure and test multiple MCP servers in a single test run, via both stdio and SSE transports. |
| **Automated Reporting** | Auto-generates structured test reports in the `mcp-report/` directory after each evaluation run. |

---

## 2. What is MCP and Why Test It?

### 2.1 Model Context Protocol (MCP) Overview

The Model Context Protocol (MCP) is an open standard that enables AI assistants and language models to communicate with external data sources and tools in a consistent, structured way. An MCP Server exposes a collection of tools, resources, and prompts that any MCP-compatible client (such as Claude, a custom agent, or an IDE) can discover and invoke.

Every tool registered on an MCP server has three critical elements that the AI model reads to decide whether and how to call it:

- **Name** — the identifier the model uses to reference the tool
- **Description** — a natural language explanation the model reads to understand when to use the tool
- **Input Schema** — a JSON Schema defining the expected parameters, types, and constraints

### 2.2 The Testing Challenge

Unlike traditional software APIs which are called deterministically, MCP tool calls are selected by an AI model. This means the quality of your tool definitions directly affects call accuracy. Common failure modes include:

- A model failing to recognise which tool to call because the description is ambiguous
- A model calling the wrong tool when multiple tools have similar names or purposes
- A model providing incorrect parameter values because the schema description is unclear
- Different AI models behaving inconsistently against the same server definition

> **Why This Matters:** A single poorly-worded tool description can cause an AI agent to silently fail, use the wrong API endpoint, or pass malformed parameters — all without throwing an error. Testing reveals these issues before they affect users in production.

### 2.3 How the Framework Solves This

The `mcp-testing-framework` takes a **black-box evaluation approach**: you describe what input (prompt) should trigger which tool call with what parameters, then the framework submits that prompt to real AI models against your actual MCP server and measures whether the model selects the correct tool and parameters. Running this multiple rounds and averaging the results produces a statistically reliable pass rate.

---

## 3. Architecture

### 3.1 High-Level Architecture

The framework is structured as a CLI-driven evaluation pipeline with four major layers:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  LAYER 1 — CLI / Test Runner                                            │
│  npx mcp-testing-framework evaluate                                     │
│  Reads YAML config | Orchestrates test rounds | Aggregates results      │
├─────────────────────────────────────────────────────────────────────────┤
│  LAYER 2 — Model Provider Registry                                      │
│  Built-in: OpenAI | Anthropic | Google Gemini | DeepSeek               │
│  + Custom Providers (plug-in interface)                                 │
├─────────────────────────────────────────────────────────────────────────┤
│  LAYER 3 — MCP Transport Layer                                          │
│  stdio (command/args)  |  SSE / HTTP Streaming (URL-based)             │
├─────────────────────────────────────────────────────────────────────────┤
│  LAYER 4 — Reporting Engine                                             │
│  Pass rate per test | Per-model breakdown | mcp-report/ HTML & JSON     │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Component Details

#### Layer 1 — CLI / Test Runner

The entry point is the `mcp-testing-framework` CLI, invokable via `npx` without a local install. When `evaluate` is called, the runner reads the YAML configuration file, resolves all MCP server connections (spawning stdio processes or opening SSE connections), and then iterates through each defined test case for each specified model over the configured number of rounds. Results are buffered in memory and handed to the reporting engine on completion.

#### Layer 2 — Model Provider Registry

The framework ships with built-in adapter implementations for the four major model families. Each adapter wraps the respective vendor SDK, constructs a tool-use prompt from the test case, submits it to the model alongside the MCP server's tool definitions (retrieved live), and parses the model's `tool_call` response to check whether the correct tool and parameters were selected. Custom providers can be registered by implementing a simple provider interface, making it straightforward to test against private or self-hosted models.

#### Layer 3 — MCP Transport Layer

The framework connects to your MCP server using the standard MCP transports. Two modes are supported:

- **stdio mode:** The framework spawns your server as a child process using the `command` and `args` you specify, communicates over stdin/stdout using JSON-RPC 2.0, and tears the process down after the run.
- **SSE/HTTP mode:** The framework connects to a running server via the URL you provide, suitable for remote or already-running server instances. Environment variables can be injected into the server process to pass API keys and configuration.

#### Layer 4 — Reporting Engine

After all rounds complete, the reporting engine calculates pass rates per test case and per model, compares them against the configured `passThreshold`, and writes structured reports to the `mcp-report/` directory. Reports include both machine-readable JSON and human-readable HTML summaries, making them easy to archive, diff across releases, and integrate into CI/CD dashboards.

### 3.3 Data Flow

The following sequence describes a single test case evaluation across one model and one round:

| Step | Action | Detail |
|---|---|---|
| **1** | Framework reads YAML config | Parses `mcp-testing-framework.yaml`, validating all server configs, model references, and test case definitions. |
| **2** | MCP server connection established | Spawns the server process (stdio) or opens an HTTP connection (SSE) and performs the MCP handshake to retrieve available tools. |
| **3** | Prompt submitted to AI model | The test case prompt is sent to the selected model along with the tool schemas retrieved from the server. |
| **4** | Response parsed & validated | The model's `tool_call` response is parsed. The framework checks the selected tool name and supplied parameters against expectations. |
| **5** | Round result recorded | Pass or fail is recorded. Steps 3–4 repeat for the configured number of `testRounds`. |
| **6** | Pass rate calculated | `pass rate = passed rounds / total rounds`. Compared to `passThreshold` to determine test outcome. |
| **7** | Report generated | Results written to `mcp-report/` with a console summary showing per-model pass rates. |

---

## 4. Installation and Project Setup

### 4.1 Prerequisites

Before using the framework, ensure the following are available:

- **Node.js 18 or later** (required to run the CLI via `npx`)
- **An API key** for at least one supported model provider (OpenAI, Anthropic, Google Gemini, or DeepSeek)
- **An MCP server** to test — accessible via stdio command or a running HTTP/SSE endpoint

### 4.2 Initialising a New Test Project

The fastest way to get started is to use the built-in project scaffolder:

```bash
# Create a new test project in the 'mcp-tests' directory
npx mcp-testing-framework@latest init ./mcp-tests --example getting-started

# Navigate into the project
cd mcp-tests
```

The `init` command generates the following project structure:

```
mcp-tests/
  mcp-testing-framework.yaml   # Main configuration file
  .env                          # API keys (add your keys here)
  .env.example                  # Template for API key variables
  mcp-report/                   # Auto-created on first run, stores reports
  custom-providers/             # Optional: place custom model providers here
```

### 4.3 Configuring API Keys

Create a `.env` file in the project root with the API keys for the models you intend to test. **Never commit this file to version control.**

```bash
# .env — API keys for model providers
OPENAI_API_KEY=sk-your-openai-key-here
ANTHROPIC_API_KEY=sk-ant-your-anthropic-key-here
GOOGLE_GENERATIVE_AI_API_KEY=your-gemini-key-here
DEEPSEEK_API_KEY=your-deepseek-key-here
```

> **Security Note:** The `.env` file is automatically excluded from git via `.gitignore` in generated projects. Always verify this before committing. If your team uses a secrets manager (AWS Secrets Manager, HashiCorp Vault, GitHub Actions secrets), inject keys as environment variables at CI runtime instead.

---

## 5. Configuration Reference

All test behaviour is controlled by a single YAML file named `mcp-testing-framework.yaml` in your project root.

### 5.1 Full Configuration Example

```yaml
# mcp-testing-framework.yaml

# Number of times each test case is run per model.
# Higher values give more statistically stable pass rates.
testRound: 10

# Minimum pass rate (0.0 to 1.0) required to mark a test as passing.
# 0.8 means the model must correctly call the tool in at least 8 out of 10 rounds.
passThreshold: 0.8

# List of model identifiers to evaluate against your MCP server.
# Format: <provider>:<model-id>
modelsToTest:
  - openai:gpt-4o
  - openai:gpt-4-turbo
  - anthropic:claude-3-5-sonnet-20241022
  - google:gemini-1.5-pro
  - deepseek:deepseek-chat

# MCP server definitions. Each entry represents one server under test.
mcpServers:
  # stdio mode: framework spawns the server as a child process
  my-local-server:
    command: node
    args:
      - dist/server.js
    env:
      SERVER_API_KEY: ${SERVER_API_KEY}   # inject from environment

  # SSE / HTTP Streaming mode: connect to an already-running server
  my-remote-server:
    url: http://localhost:3000/sse

# Test case definitions.
testCases:
  - name: 'Get current weather for a city'
    prompt: 'What is the weather like in Tokyo right now?'
    expectedServerName: my-local-server
    expectedToolName: get_weather
    expectedArguments:
      city: Tokyo

  - name: 'Search for a product by keyword'
    prompt: 'Find me laptops under $1000'
    expectedServerName: my-local-server
    expectedToolName: search_products
    expectedArguments:
      query: laptops
      maxPrice: 1000
```

### 5.2 Configuration Parameter Reference

| Parameter | Type / Default | Description |
|---|---|---|
| `testRound` | integer / `10` | Number of evaluation rounds per test case per model. Increase for higher statistical confidence. Values of 5–20 are typical. |
| `passThreshold` | float / `0.8` | Minimum fraction of rounds that must pass (0.0–1.0). Set to `1.0` to require perfect consistency; 0.7–0.8 is common for production gates. |
| `modelsToTest` | string[] | List of model identifiers in `<provider>:<model-id>` format. Supported providers: `openai`, `anthropic`, `google`, `deepseek`, plus custom providers. |
| `mcpServers` | map | Map of named server configurations. Each entry is either stdio mode (`command` + `args`) or SSE mode (`url`). The key is used in test cases as `expectedServerName`. |
| `command` | string | Executable to spawn for stdio servers. Examples: `node`, `python`, `uvx`, `npx`. |
| `args` | string[] | Arguments passed to `command`. Example: `['dist/index.js', '--port', '3000']`. |
| `env` | map | Environment variables injected into the spawned server process. Supports `${VAR}` substitution from the host environment. |
| `url` | string | HTTP/SSE endpoint URL for remote server mode. Example: `http://localhost:3000/sse`. |
| `testCases` | array | Array of test case objects. Each defines a prompt, the server to target, the expected tool, and the expected parameter values. |
| `name` | string | Human-readable test case label shown in reports. |
| `prompt` | string | The natural language prompt sent to the AI model. Write it as a user would naturally ask for the feature the tool provides. |
| `expectedServerName` | string | The key of the `mcpServers` entry that should handle this test case. |
| `expectedToolName` | string | The tool name (as registered on the server) that the model is expected to call. |
| `expectedArguments` | map | Key/value pairs representing the parameter names and values the model should pass when calling the tool. |

---

## 6. Running Tests

### 6.1 Execute an Evaluation

With your configuration and API keys in place, run the evaluation command from the project root:

```bash
# Run the full evaluation
npx mcp-testing-framework@latest evaluate

# Or, if you have installed the package locally:
npx mcp-testing-framework evaluate
```

The CLI will display a live progress indicator as it runs each round, then print a results summary:

```
Running evaluation...

Test Case: Get current weather for a city
  Model: openai:gpt-4o           [ ========== ]  Pass Rate: 10/10  (100%)  PASS
  Model: anthropic:claude-3-5-s  [ ========== ]  Pass Rate:  9/10   (90%)  PASS
  Model: google:gemini-1.5-pro   [ ======....  ]  Pass Rate:  6/10   (60%)  FAIL

Test Case: Search for a product by keyword
  Model: openai:gpt-4o           [ ========== ]  Pass Rate: 10/10  (100%)  PASS
  Model: anthropic:claude-3-5-s  [ ========== ]  Pass Rate:  8/10   (80%)  PASS
  Model: google:gemini-1.5-pro   [ =======.... ]  Pass Rate:  7/10   (70%)  FAIL

Report saved to: ./mcp-report/2026-03-15_evaluation.html
```

### 6.2 Understanding the Report

The `mcp-report/` directory contains two output files after each run:

- **HTML file** — a human-friendly dashboard with colour-coded pass/fail indicators, per-model breakdown charts, and a full log of which rounds passed or failed.
- **JSON file** — raw results data suitable for programmatic processing, trend analysis, or feeding into your own dashboards.

> **Interpreting Results:** A FAIL result does not always mean your server is broken — it may mean your tool description needs to be reworded. Examine which rounds failed and what parameters the model attempted to pass. If the model consistently picks the right tool but passes slightly wrong parameters, refine the parameter descriptions in your JSON Schema. If the model picks the wrong tool, improve the tool's description to better differentiate it from similar tools.

### 6.3 CI/CD Integration

The CLI exits with a **non-zero exit code** when any test case fails to meet the `passThreshold`, making it directly usable as a CI gate:

```yaml
# GitHub Actions example (.github/workflows/mcp-eval.yml)
name: MCP Server Evaluation

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  evaluate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Build MCP server
        run: npm ci && npm run build

      - name: Run MCP evaluation
        run: npx mcp-testing-framework@latest evaluate
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}

      - name: Upload report
        uses: actions/upload-artifact@v4
        with:
          name: mcp-report
          path: mcp-report/
```

---

## 7. Writing Effective Test Cases

### 7.1 Test Case Design Principles

| Principle | Guidance |
|---|---|
| **Write natural prompts** | Phrase prompts as a real user would type them. Avoid using the exact tool name or parameter names in the prompt — this creates artificially easy tests. |
| **One tool per test** | Each test case should have exactly one correct tool call. Avoid ambiguous prompts that could legitimately trigger multiple tools. |
| **Test edge cases** | Include prompts with unusual phrasing, synonyms, or different orderings to ensure the model generalises the tool definition rather than pattern-matching keywords. |
| **Parameterised expectations** | Be precise with `expectedArguments`. If your tool accepts a city name, test both capitalised and lowercase versions to understand model normalisation behaviour. |
| **Separate tools clearly** | Write test cases for every tool your server exposes. Tools that are never tested provide no signal about their definition quality. |
| **Use realistic prompts** | Base prompts on real usage examples collected from users or product requirements. Test cases derived from actual usage are far more valuable than synthetic examples. |

### 7.2 Example: Weather MCP Server

The following is a well-structured set of test cases for a hypothetical Weather MCP server with two tools: `get_current_weather` and `get_forecast`:

```yaml
testCases:
  # Test 1: Core tool selection — unambiguous prompt
  - name: 'Current weather — direct city query'
    prompt: "What's the weather in Paris right now?"
    expectedServerName: weather-server
    expectedToolName: get_current_weather
    expectedArguments:
      city: Paris

  # Test 2: Same tool, different phrasing
  - name: 'Current weather — indirect phrasing'
    prompt: 'Should I bring an umbrella in London today?'
    expectedServerName: weather-server
    expectedToolName: get_current_weather
    expectedArguments:
      city: London

  # Test 3: Forecast tool — distinguishing from current
  - name: 'Forecast — future tense prompt'
    prompt: 'Will it rain in New York this weekend?'
    expectedServerName: weather-server
    expectedToolName: get_forecast
    expectedArguments:
      city: New York
      days: 3

  # Test 4: Forecast tool with explicit duration
  - name: 'Forecast — explicit duration'
    prompt: 'Give me a 7-day forecast for Tokyo'
    expectedServerName: weather-server
    expectedToolName: get_forecast
    expectedArguments:
      city: Tokyo
      days: 7
```

### 7.3 Diagnosing Failures

When a test case fails, examine the report's failure log to identify the pattern:

- **Wrong tool selected:** Rewrite the tool description to more clearly differentiate it from the tool the model chose instead. Add concrete examples of when to use this tool vs. the alternatives.
- **Correct tool but wrong parameters:** Improve the JSON Schema descriptions for the affected parameters. Add examples and constraints. Specify units, formats, and acceptable values explicitly.
- **Inconsistent results (sometimes right, sometimes wrong):** The prompt is at the model's decision boundary. Make the tool description more distinctive or add more synonym coverage. Also consider raising `testRound` to get a more reliable statistical signal.

---

## 8. Custom Model Providers

The framework's built-in providers cover OpenAI, Anthropic, Google Gemini, and DeepSeek. For other models — including private endpoints, self-hosted open-source models, or any vendor not yet natively supported — you can register a **custom provider**.

### 8.1 Provider Interface

A custom provider is a JavaScript/TypeScript module that exports an object conforming to the framework's provider interface:

```javascript
// custom-providers/my-provider.js
module.exports = {
  // Provider identifier — used in modelsToTest as 'myprovider:my-model'
  name: 'myprovider',

  /**
   * Evaluates a single prompt against the model.
   * @param {string} prompt         - The test case prompt
   * @param {object[]} tools        - MCP tool schemas (JSON Schema format)
   * @param {string} modelId        - The model ID from modelsToTest config
   * @returns {{ toolName: string, arguments: object }}
   */
  async evaluate(prompt, tools, modelId) {
    // Call your model here
    const response = await myModelClient.chat({
      model: modelId,
      messages: [{ role: 'user', content: prompt }],
      tools: tools,  // Pass the MCP tools as function definitions
    });

    // Parse and return the tool call
    const toolCall = response.choices[0].message.tool_calls[0];
    return {
      toolName: toolCall.function.name,
      arguments: JSON.parse(toolCall.function.arguments),
    };
  },
};
```

### 8.2 Registering the Custom Provider

Place the provider file in the `custom-providers/` directory. The framework auto-discovers provider modules in this directory at startup. Then reference the provider in your YAML:

```yaml
modelsToTest:
  - openai:gpt-4o
  - myprovider:my-custom-model-v1   # Uses the custom provider
  - myprovider:my-custom-model-v2   # Same provider, different model variant
```

---

## 9. Comparison with Alternative Approaches

| Tool / Approach | Primary Use Case | Best For |
|---|---|---|
| **mcp-testing-framework** | AI model evaluation — how well do your tool definitions guide model tool selection? | Teams wanting quantitative pass rates across multiple AI models to validate MCP server tool quality. |
| **MCP Inspector** (official) | Interactive visual inspection — manually explore and invoke your server's tools in a browser UI. | Developers debugging a specific tool call or exploring a server's capabilities during development. |
| **mcp-jest** | Protocol compliance and contract testing — does the server respond correctly to MCP requests? | Ensuring MCP wire protocol correctness, response schemas, and connection behaviour. |
| **thoughtspot/mcp-testing-kit** | Lightweight unit-style testing — invoke tools directly in Vitest without a real transport. | Fast unit tests for server logic without requiring a running server process or real AI calls. |
| **@t3ta/mcp-test** | End-to-end integration testing with Vitest — manage a real server lifecycle in tests. | Integration test suites needing full server lifecycle management alongside regular test runners. |

> **Recommendation:** Use `mcp-testing-framework` for ongoing model-quality evaluation integrated into your CI/CD pipeline. Use MCP Inspector during active development for interactive debugging. Use `mcp-jest` or `mcp-testing-kit` for unit and integration tests of server implementation logic. These tools are complementary, not mutually exclusive.

---

## 10. Best Practices and Recommendations

### 10.1 MCP Server Design

- Give every tool a unique, unambiguous name. Avoid generic names like `process` or `handle` — use `get_order_status`, `cancel_shipment`, `search_products` instead.
- Write tool descriptions as user-intent statements, not technical specifications. The description is read by the AI model, not a human engineer.
- Use concrete examples in descriptions. Instead of *"retrieves weather data for a location"*, write *"use this to get the current temperature, humidity, and conditions for a specified city"*.
- Add `enum` constraints and format hints to your JSON Schema parameters. This gives the model a precise vocabulary to match against.
- Keep tools focused on a single action. A tool that does too many things based on parameters is harder for models to reason about correctly.

### 10.2 Test Strategy

- Aim for at least **5 test cases per tool**. Cover the primary use case, at least two alternative phrasings, and any edge cases specific to your domain.
- Use `testRound: 10` as a baseline. Increase to `20` for critical production servers where consistency is paramount.
- Set `passThreshold: 0.8` as a reasonable initial target. Raise it to `0.9` or `1.0` as your server matures.
- Re-run evaluations on **every change** to a tool's name, description, or input schema — even small wording changes can have outsized effects on model behaviour.
- Test against **all models your users may employ**, not just the one you developed against. A tool definition that works perfectly for GPT-4o may score poorly with Gemini.

### 10.3 CI/CD Integration

- Store evaluation results as CI artifacts for trend analysis. Compare pass rates over time to detect gradual regressions.
- Run evaluations on pull requests that modify any MCP server definition file. Gate merges on passing the threshold.
- Keep API costs predictable by bounding `testRound` and the number of models tested in CI. Save comprehensive multi-model sweeps for nightly or release builds.
- Use environment variable injection (GitHub Actions secrets, GitLab CI variables) for API keys — never hard-code them in config files.

---

## 11. Troubleshooting

| Problem | Solution |
|---|---|
| **Server fails to start (stdio mode)** | Verify that `command` and `args` are correct by running them manually in a terminal. Check that required environment variables are defined in the `env` block. Ensure the server binary has execute permissions. |
| **Authentication error from model API** | Double-check the API key in your `.env` file. Confirm the key is exported as an environment variable if not using a `.env` file. Ensure the key has the required model access permissions. |
| **All test cases fail with 0% pass rate** | This usually indicates the server is not connecting properly, not exposing any tools, or the model cannot make tool calls with the specified model ID. Run MCP Inspector against the server to verify tool availability. |
| **Inconsistent results across runs** | Increase `testRound` to 20 or higher to stabilise the statistical signal. If results remain inconsistent, the prompt is genuinely ambiguous — revise the tool description and test prompt. |
| **Custom provider not being detected** | Ensure the provider file is in `custom-providers/` and exports a `name` property that matches the prefix used in `modelsToTest`. Restart the CLI after adding new provider files. |
| **Reports not generated** | Verify the `mcp-report/` directory exists and the current user has write permissions. The directory is created automatically if absent, but permissions issues can prevent this. |
| **Very high API costs per run** | Reduce `testRound` or limit the number of models in `modelsToTest` for routine runs. Use a lower-cost model tier for development and reserve full sweeps for release validation. |

---

## 12. Quick Reference

### 12.1 CLI Commands

```bash
# Scaffold a new test project
npx mcp-testing-framework@latest init <dir> --example getting-started

# Run the evaluation against your configuration
npx mcp-testing-framework@latest evaluate

# Run evaluation with a custom config file path
npx mcp-testing-framework@latest evaluate --config ./path/to/config.yaml
```

### 12.2 Supported Model Identifiers

```
# OpenAI
openai:gpt-4o
openai:gpt-4-turbo
openai:gpt-3.5-turbo

# Anthropic
anthropic:claude-3-5-sonnet-20241022
anthropic:claude-3-opus-20240229

# Google Gemini
google:gemini-1.5-pro
google:gemini-1.5-flash

# DeepSeek
deepseek:deepseek-chat

# Custom (register your own provider)
<your-provider-name>:<model-id>
```

### 12.3 Useful Links

- **GitHub Repository:** [https://github.com/L-Qun/mcp-testing-framework](https://github.com/L-Qun/mcp-testing-framework)
- **Model Context Protocol Documentation:** [https://modelcontextprotocol.io](https://modelcontextprotocol.io)
- **MCP Inspector (visual debugging tool):** [https://github.com/modelcontextprotocol/inspector](https://github.com/modelcontextprotocol/inspector)

---

*This document was prepared for internal company use based on the `mcp-testing-framework` open-source project by L-Qun. Last updated: March 2026.*
