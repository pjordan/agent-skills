# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2026-04-11

### Added
- Initial release as a Claude Code plugin
- **agent-wiki** skill: persistent knowledge base for coding agents
  - `init` operation to bootstrap wiki from codebase scan
  - `ingest` operation to file discoveries with cross-references
  - `query` operation to consult accumulated knowledge
  - `lint` operation for wiki health checks
  - Based on [Andrej Karpathy's llm-wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- Plugin manifest and marketplace configuration
