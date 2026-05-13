# Changelog

All notable changes to this plugin are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Planned

- `pipeline` skill — status of your open/merged PRs across forks
- `log` skill — generate a portfolio entry from merged contributions

## [0.1.0] — 2026-05-13

### Added

- `find-issues` skill — ranked shortlist of ripe issues across watched repos
- `contribute-upstream` skill — reactive flow for bugs hit in third-party deps: pre-flight gate, repro bridge, fix, PR with confirmation gate, local-patch handoff
- `profile` skill — shared preferences for watched repos, languages, default GitHub account, budget, and "ripe" heuristic
- `.claude-plugin/plugin.json` + `.claude-plugin/marketplace.json` so the repo doubles as its own marketplace
