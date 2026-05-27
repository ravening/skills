# Report Template

Use this structure for the final report.

```md
# Spring Boot Application Audit Report

**Project:** <name>
**Build tool:** <maven-or-gradle>
**Spring Boot version:** <version>
**Java version:** <version>
**Generated:** <timestamp>

## Executive Summary

| Category | Status | Notes |
|---|---|---|
| Architecture |  |  |
| Dependencies |  |  |
| Security |  |  |
| Code Quality |  |  |
| Configuration |  |  |
| Tests |  |  |

## Project Overview

- Modules:
- Main package roots:
- Java source count:
- Test source count:
- Key integrations:

## API Surface

| Method | Path | Controller | Auth | Notes |
|---|---|---|---|---|

## Architecture Diagram

```mermaid
<component diagram>
```

## End-to-End Flow

```mermaid
<sequence diagram>
```

## Dependency Findings

| Dependency | Version | Risk | Notes | Recommendation |
|---|---|---|---|---|

## Security Findings

### P0
### P1
### P2
### P3

## Code Review Findings

| Finding | Location | Severity | Recommendation |
|---|---|---|---|

## Configuration Findings

| Setting | Current State | Risk | Recommendation |
|---|---|---|---|

## Test Assessment

| Layer | Status | Gaps |
|---|---|---|

## Recommended Improvements

### P0 — Fix Immediately
### P1 — Fix This Sprint
### P2 — Plan Soon
### P3 — Backlog
```
