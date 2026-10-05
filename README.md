# MaxInTheCloud - DevOps Portfolio

Portfolio of Max Cerny — Senior DevOps / DevSecOps & Cloud Platform Engineer focused on AI-driven automation. Built with Hugo.

## Tech Stack
- **Hugo** - Static site generator
- **Tailwind CSS** - Utility-first CSS framework (CDN)
- **GitHub Actions** - Automated deployment
- **GitHub Pages** - Hosting

## Features
- Responsive design with mobile menu
- Dynamic availability status system
- Experience timeline driven by `data/experience.yaml`
- SecOps Scanner product landing page at `/secops/` and `/cs/secops/` (shared `static/secops/secops.css`)
- Case study pages
- SEO optimized with Open Graph tags
- Custom 404 page
- Smooth scroll navigation

## CV Download Sign-up
Clicking a CV download link shows an optional MailerLite sign-up for availability updates (once per browser).
It is enabled by filling `params.mailerlite.accountId` and `params.mailerlite.formId` in `config.toml`;
both IDs are in the form action of a MailerLite embedded form (`https://assets.mailerlite.com/jsonp/<accountId>/forms/<formId>/subscribe`).

## Local Development

```bash
# Run Hugo development server
hugo server -D

# Build for production
hugo --minify
```

## Deployment
This site is automatically deployed to GitHub Pages via GitHub Actions on every push to the main branch.

## Domain Configuration
Custom domain: maxinthecloud.com

## License
© 2026 MaxInTheCloud. All rights reserved.
