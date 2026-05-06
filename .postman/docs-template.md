# [API Name]

**Owner**: [Team Name] ([email] | [#slack-channel])
**Status**: Stable | Development | Deprecated
**Last Updated**: [Date]

## Purpose

Core API services for the hotel room booking platform. Used by front-end booking flows, internal platform services, and CI/CD pipelines to validate availability, pricing, reservations, and cross-sell recommendations across all property environments.

## Quick Start

1. Select the appropriate environment (dev → nonprod-preprod)
2. Authenticate using OAuth2 client credentials via the Resolve Secrets request or your environment's pre-wired token
3. Run a health check to verify connectivity
4. Browse the endpoints below for your first real request

## Authentication

**Type**: OAuth 2.0 Client Credentials

Credentials are resolved from AWS Secrets Manager (`api-credentials-{env}`) at runtime. In CI, the `CI=true` environment variable bypasses the secrets fetch and uses the mock server directly.

## Endpoints Overview

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /health | Service health check |
| GET | /[resources] | List all resources |
| POST | /[resources] | Create a resource |
| GET | /[resources]/{id} | Get a specific resource |

## Lifecycle

**Stable** — Production-ready, follows semantic versioning.

Breaking changes announced via workspace updates with 30 days notice.

## Support

Questions? Contact [Team] at [email] or [#slack-channel].
