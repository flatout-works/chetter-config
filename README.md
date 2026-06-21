# Chetter Config

Git-backed runtime configuration for Chetter.

The MCP server reads `model-catalog.yaml` from this repository when `DEFINITIONS_REPO` points here. Do not store secret values in this repository; use environment variable names such as `api_key_env: ANTHROPIC_API_KEY`.
