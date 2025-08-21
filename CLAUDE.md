# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Installation
- Install dependencies: `yarn`

### Development
- Start development server: `yarn start` (opens browser at http://localhost:3000)
- Build for production: `yarn build`
- Serve production build: `yarn serve`
- Clear cache: `yarn clear`

### Deployment
- Deploy to GitHub Pages: `yarn deploy`
- Deploy via SSH: `USE_SSH=true yarn deploy`
- Deploy with custom user: `GIT_USER=<username> yarn deploy`

### Content Management
- Generate translation files: `yarn write-translations`
- Auto-generate heading IDs: `yarn write-heading-ids`

## Architecture Overview

### Project Structure
This is a Docusaurus documentation site for the Forest City Labs Framework, a PSR-compliant PHP framework targeted towards GraphQL APIs.

**Key Directories:**
- `docs/`: Markdown documentation files organized by topic
- `src/`: React components and custom CSS
- `static/`: Static assets (images, favicons, etc.)
- `build/`: Generated static site (created by build command)

### Documentation Organization
- `docs/Standard Install/`: Getting started guides
- `docs/Components/`: Framework component documentation
- `docs/Utilities/`: Utility functions and helpers
- `docs/GraphQL/`: GraphQL-specific documentation with mapping and reference guides

### Configuration Files
- `docusaurus.config.js`: Main Docusaurus configuration
- `sidebars.js`: Documentation sidebar structure
- `babel.config.js`: Babel transpilation settings
- `package.json`: Node.js dependencies and scripts

### Site Configuration
- Base URL: `/` (configured for GitHub Pages)
- Organization: `forestcitylabs`
- Default locale: `en-ca`
- Supports PHP syntax highlighting in code blocks
- GitHub integration with links to source repositories

### Content Guidelines
- Documentation is written in Markdown/MDX format
- Code examples should use proper PHP syntax highlighting
- Images and assets stored in `/static/img/` directory
- Internal links use relative paths to other documentation pages

### Development Notes
- Uses React 18+ and Docusaurus 3.5+
- Requires Node.js 18.0 or higher
- Hot reload enabled during development
- Production builds generate optimized static files