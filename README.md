# FLTR Documentation

Official documentation for FLTR - Context as a Service platform.

## Overview

This documentation is built with [Mintlify](https://mintlify.com), a modern documentation platform optimized for developer experience.

## Local Development

Install the Mintlify CLI to preview documentation changes locally:

```bash
npm i -g mint
```

Run the documentation server at the root of this directory:

```bash
cd docs
mint dev
```

View your local preview at `http://localhost:3000`.

## Documentation Structure

```
docs/
├── docs.json              # Navigation configuration
├── introduction.mdx       # Landing page
├── quickstart/            # Getting started guides
│   ├── getting-started.mdx
│   └── first-integration.mdx
├── authentication/        # Auth methods
│   ├── overview.mdx
│   ├── api-keys.mdx
│   ├── oauth.mdx
│   ├── sessions.mdx
│   └── scopes.mdx
├── integrations/          # Third-party integrations
│   ├── zapier.mdx
│   ├── make.mdx
│   ├── n8n.mdx
│   ├── webhooks.mdx
│   └── langchain.mdx
├── guides/                # Best practices
│   ├── rag-best-practices.mdx
│   └── caching.mdx
├── api-reference/         # API documentation
│   ├── introduction.mdx
│   ├── datasets/
│   ├── documents/
│   ├── mcp/
│   └── webhooks/
└── support/               # Support resources
    ├── troubleshooting.mdx
    ├── rate-limits.mdx
    ├── faq.mdx
    ├── security.mdx
    └── cors.mdx
```

## Publishing Changes

This documentation is automatically deployed via Mintlify's GitHub integration:

1. Make changes to `.mdx` files or `docs.json`
2. Commit and push to the repository
3. Changes are automatically deployed to production

**Production URL:** (To be configured in Mintlify dashboard)

## Writing Documentation

### MDX Format

All documentation files use [MDX](https://mdxjs.com/), which allows you to use JSX components within Markdown:

```mdx
---
title: 'Page Title'
description: 'Page description for SEO'
---

# Heading

Regular markdown content.

<Card title="Click me" icon="rocket" href="/link">
  Card content
</Card>
```

### Mintlify Components

Available components:

- `<Card>` - Clickable cards with icons
- `<CardGroup>` - Group multiple cards
- `<Accordion>` - Collapsible sections
- `<AccordionGroup>` - Group accordions
- `<Tabs>` - Tabbed content
- `<CodeGroup>` - Multi-language code examples
- `<ParamField>` - API parameter documentation
- `<ResponseField>` - API response documentation
- `<Warning>`, `<Info>`, `<Tip>` - Callout boxes

See [Mintlify Components](https://mintlify.com/docs/components) for full list.

### Code Examples

Use code groups for multi-language examples:

````mdx
<CodeGroup>

```bash cURL
curl -X GET https://api.fltr.com/v1/datasets
```

```python Python
import requests
response = requests.get("https://api.fltr.com/v1/datasets")
```

```javascript JavaScript
const response = await fetch("https://api.fltr.com/v1/datasets");
```

</CodeGroup>
````

## Configuration

### Navigation (docs.json)

Update `docs.json` to modify:
- Navigation structure
- Branding (colors, logo, favicon)
- Tabs (Documentation vs API Reference)
- Footer links
- SEO settings

### Anchors

Mintlify supports custom anchor links in navigation for external resources:

```json
{
  "name": "GitHub",
  "icon": "github",
  "url": "https://github.com/fltr"
}
```

## SEO & Metadata

Each page should include frontmatter:

```yaml
---
title: 'Page Title'              # Required
description: 'Page description'  # Required (for SEO)
---
```

## Troubleshooting

### Dev Server Not Running

```bash
mint update
```

Ensure you have the latest version of the CLI.

### Page Shows 404

Make sure:
1. File is in the correct directory
2. Page is added to `docs.json` navigation
3. Running `mint dev` in the correct directory (where `docs.json` is located)

### Changes Not Showing

1. Restart dev server (`Ctrl+C` then `mint dev`)
2. Clear browser cache
3. Check for syntax errors in MDX

## Resources

- [Mintlify Documentation](https://mintlify.com/docs)
- [MDX Documentation](https://mdxjs.com/)
- [Mintlify Discord](https://discord.gg/mintlify)

## Contributing

When adding new documentation:

1. Create `.mdx` file in appropriate directory
2. Add frontmatter (title, description)
3. Add to `docs.json` navigation
4. Test locally with `mint dev`
5. Commit and push changes

For major changes, consider:
- Updating related pages
- Adding cross-references
- Including code examples
- Testing all links

## Support

Questions about the documentation?

- **FLTR Support:** support@fltr.com
- **Mintlify Support:** https://mintlify.com/docs
