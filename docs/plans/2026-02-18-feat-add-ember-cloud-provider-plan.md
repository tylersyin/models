---
title: Add Ember Cloud Provider
type: feat
date: 2026-02-18
---

# feat: Add Ember Cloud Provider

## Overview

Add Ember Cloud (embercloud.ai) as a new provider in the Portkey models repository. Ember Cloud offers GLM-family models (GLM-5, GLM-4.7, GLM-4.7 Flash, GLM-4.6, GLM-4.5, GLM-4.5 Air) via an OpenAI-compatible API with competitive pricing and cache read support.

## Problem Statement / Motivation

Ember Cloud is a provider of GLM-series models with an OpenAI-compatible API (`https://api.embercloud.ai/v1`). It currently offers 6 chat models with tool calling support, reasoning capabilities (on select models), and prompt caching. Adding it to Portkey's models database enables users to route requests through Portkey to Ember Cloud with accurate pricing tracking.

## Proposed Solution

Create two new JSON files following the existing Portkey provider conventions:

1. `pricing/ember-cloud.json` - Token pricing for all 6 models
2. `general/ember-cloud.json` - Model configurations, params, and capabilities

## Technical Approach

### Pricing Conversion

Ember Cloud publishes pricing in **USD per 1M tokens**. Portkey stores prices in **cents per token**.

Formula: `cents_per_token = (usd_per_million / 1,000,000) * 100`

| Model | Input ($/1M) | Output ($/1M) | Cache Read ($/1M) | Input (cents/token) | Output (cents/token) | Cache Read (cents/token) |
|-------|-------------|---------------|-------------------|--------------------|--------------------|------------------------|
| glm-5 | $1.00 | $3.20 | $0.20 | 0.0001 | 0.00032 | 0.00002 |
| glm-4.7 | $0.40 | $1.75 | $0.08 | 0.00004 | 0.000175 | 0.000008 |
| glm-4.7-flash | $0.06 | $0.40 | $0.01 | 0.000006 | 0.00004 | 0.000001 |
| glm-4.6 | $0.43 | $1.74 | $0.08 | 0.000043 | 0.000174 | 0.000008 |
| glm-4.5 | $0.60 | $2.20 | $0.11 | 0.00006 | 0.00022 | 0.000011 |
| glm-4.5-air | $0.13 | $0.85 | $0.025 | 0.000013 | 0.000085 | 0.0000025 |

### Model Capabilities (from Ember Cloud source code)

| Model | Context | Max Output | Tool Calling | Reasoning | Logprobs |
|-------|---------|------------|-------------|-----------|----------|
| glm-5 | 203K | 131K | Yes | Yes | No |
| glm-4.7 | 200K | 131K | Yes | Yes | Yes |
| glm-4.7-flash | 200K | 131K | Yes | No | Yes |
| glm-4.6 | 200K | 128K | Yes | Yes | Yes |
| glm-4.5 | 131K | 96K | Yes | No | Yes |
| glm-4.5-air | 131K | 96K | Yes | No | Yes |

All models support: text input/output, streaming, JSON mode, temperature, top_p, max_tokens, stop sequences, seed.

## Acceptance Criteria

- [x] Create `pricing/ember-cloud.json` with default entry and all 6 model entries
- [x] Create `general/ember-cloud.json` with default params and per-model configurations
- [x] All prices use cache_read_input_token field for cache read pricing
- [x] JSON validates cleanly (`jq empty pricing/ember-cloud.json && jq empty general/ember-cloud.json`)
- [x] Model names match Ember Cloud's API exactly (`glm-5`, `glm-4.7`, `glm-4.7-flash`, `glm-4.6`, `glm-4.5`, `glm-4.5-air`)
- [x] Tool calling support indicated via `"supported": ["tools"]` on all models
- [x] Reasoning-capable models noted appropriately

## MVP

### pricing/ember-cloud.json

```json
{
  "default": {
    "pricing_config": {
      "pay_as_you_go": {
        "request_token": { "price": 0 },
        "response_token": { "price": 0 }
      },
      "calculate": {
        "request": {
          "operation": "multiply",
          "operands": [
            { "value": "input_tokens" },
            { "value": "rates.request_token" }
          ]
        },
        "response": {
          "operation": "multiply",
          "operands": [
            { "value": "output_tokens" },
            { "value": "rates.response_token" }
          ]
        }
      },
      "currency": "USD"
    }
  },
  "glm-5": {
    "pricing_config": {
      "pay_as_you_go": {
        "request_token": { "price": 0.0001 },
        "response_token": { "price": 0.00032 },
        "cache_read_input_token": { "price": 0.00002 }
      }
    }
  },
  "glm-4.7": {
    "pricing_config": {
      "pay_as_you_go": {
        "request_token": { "price": 0.00004 },
        "response_token": { "price": 0.000175 },
        "cache_read_input_token": { "price": 0.000008 }
      }
    }
  },
  "glm-4.7-flash": {
    "pricing_config": {
      "pay_as_you_go": {
        "request_token": { "price": 0.000006 },
        "response_token": { "price": 0.00004 },
        "cache_read_input_token": { "price": 0.000001 }
      }
    }
  },
  "glm-4.6": {
    "pricing_config": {
      "pay_as_you_go": {
        "request_token": { "price": 0.000043 },
        "response_token": { "price": 0.000174 },
        "cache_read_input_token": { "price": 0.000008 }
      }
    }
  },
  "glm-4.5": {
    "pricing_config": {
      "pay_as_you_go": {
        "request_token": { "price": 0.00006 },
        "response_token": { "price": 0.00022 },
        "cache_read_input_token": { "price": 0.000011 }
      }
    }
  },
  "glm-4.5-air": {
    "pricing_config": {
      "pay_as_you_go": {
        "request_token": { "price": 0.000013 },
        "response_token": { "price": 0.000085 },
        "cache_read_input_token": { "price": 0.0000025 }
      }
    }
  }
}
```

### general/ember-cloud.json

```json
{
  "name": "ember-cloud",
  "description": "",
  "default": {
    "params": [
      { "key": "max_tokens", "defaultValue": 1024, "minValue": 1, "maxValue": 131000 },
      { "key": "temperature", "defaultValue": 1, "minValue": 0, "maxValue": 2 },
      { "key": "top_p", "defaultValue": 1, "minValue": 0, "maxValue": 1 },
      { "key": "stop", "defaultValue": null, "type": "array-of-strings", "skipValues": [null, []] },
      { "key": "stream", "defaultValue": true, "type": "boolean" },
      {
        "key": "tool_choice",
        "type": "non-view-manage-data",
        "defaultValue": null,
        "options": [
          { "value": "none", "name": "None" },
          { "value": "auto", "name": "Auto" },
          { "value": "required", "name": "Required" },
          { "value": "custom", "name": "Custom", "schema": { "type": "json" } }
        ],
        "skipValues": [null, []],
        "rule": {
          "default": {
            "condition": "tools",
            "then": "auto",
            "else": null
          }
        }
      }
    ],
    "messages": { "options": ["system", "user", "assistant"] },
    "type": { "primary": "chat", "supported": ["tools"] }
  },
  "glm-5": {
    "params": [{ "key": "max_tokens", "maxValue": 131000 }],
    "type": { "primary": "chat", "supported": ["tools"] }
  },
  "glm-4.7": {
    "params": [{ "key": "max_tokens", "maxValue": 131000 }],
    "type": { "primary": "chat", "supported": ["tools"] }
  },
  "glm-4.7-flash": {
    "params": [{ "key": "max_tokens", "maxValue": 131000 }],
    "type": { "primary": "chat", "supported": ["tools"] }
  },
  "glm-4.6": {
    "params": [{ "key": "max_tokens", "maxValue": 128000 }],
    "type": { "primary": "chat", "supported": ["tools"] }
  },
  "glm-4.5": {
    "params": [{ "key": "max_tokens", "maxValue": 96000 }],
    "type": { "primary": "chat", "supported": ["tools"] }
  },
  "glm-4.5-air": {
    "params": [{ "key": "max_tokens", "maxValue": 96000 }],
    "type": { "primary": "chat", "supported": ["tools"] }
  }
}
```

## Implementation Steps

1. Create `pricing/ember-cloud.json` with the pricing data above
2. Create `general/ember-cloud.json` with the model config data above
3. Validate both files with `jq empty`
4. Commit with message: `[ember-cloud] Add Ember Cloud provider with GLM model pricing and configs`

## Source Data

- Pricing sourced from Ember Cloud billing module (`packages/shared/src/billing/index.ts`)
- Model capabilities sourced from Ember Cloud models endpoint (`apps/api/src/routes/models.ts`)
- Pricing page confirmation (`apps/web/src/app/docs/pricing/page.tsx`)

## References

- Portkey CONTRIBUTING.md: price format is cents per token, not dollars
- Reference providers used for structure: `pricing/cerebras.json`, `general/cerebras.json`
- Ember Cloud API: `https://api.embercloud.ai/v1`
