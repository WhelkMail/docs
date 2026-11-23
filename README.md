# WhelkMail Documentation

Documentation website for [WhelkMail](https://github.com/WhelkMail/WhelkMail) — a self-hosted webmail client for IMAP/SMTP accounts, deployable on Cloudflare Workers, Pages or Docker. This repository contains the source code for [WhelkMail's docs site](https://docs.whelkmail.eu).

Built with [docmd](https://github.com/docmd-io/docmd).

## Getting Started

This project requires **Node.js 18+** and npm.

```bash
# Install dependencies
npm install

# Start the development server
npm run dev

# Build the static site
npm run build

# Serve the built site locally
npm run preview
```

## Project Structure

```
├── docs/                       # Documentation content (Markdown)
│   ├── index.md                # Introduction / landing page
│   ├── getting-started.md      # Local setup and first account
│   ├── deployment.md           # Workers, Pages and Docker deployment
│   ├── configuration.md        # Bindings, secrets and variables
│   ├── oauth.md                # Google and Microsoft OAuth setup
│   ├── architecture.md         # Architecture and project layout
│   ├── mail-flows.md           # Request flows for mail operations
│   ├── data-model.md           # D1 tables and synchronization
│   ├── security.md             # Security model
│   ├── relay-accounts.md       # Send-only domains and storage
│   ├── labels-filters.md       # Labels, contacts, signatures, filters
│   ├── auto-reply.md           # Shared or per-account vacation responder
│   ├── pwa-push.md             # PWA, Web Push and release check
│   ├── forwarding-cleanup.md   # Forwarding and provider cleanup
│   ├── development.md          # Local workflow, migrations, tests
│   ├── upgrading.md            # Upgrading a deployment
│   ├── troubleshooting.md      # Common symptoms and fixes
│   ├── limitations.md          # Known limitations
│   ├── license.md              # MIT License
│   └── navigation.json         # Sidebar structure
├── assets/
│   ├── css/custom.css          # Global styles & branding
│   ├── js/logo-link.js         # Logo link behaviour
│   └── images/                 # Logo, favicon and screenshots
├── docmd.config.json           # Site configuration
└── package.json
```

## License

MIT — see [LICENSE](https://github.com/WhelkMail/WhelkMail/blob/main/LICENSE).
