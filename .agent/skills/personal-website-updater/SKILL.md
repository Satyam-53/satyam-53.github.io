---
name: personal-website-updater
description: Use when updating Satyam Srivastava's personal GitHub Pages portfolio with new projects, experiences, publications, awards, resume links, contact info, SEO metadata, or design refinements. This skill is specific to this repo's Jekyll structure and should guide content changes through _data/profile.yml, layout/CSS validation, and employer-focused positioning.
---

# Personal Website Updater

Use this skill for any future update to this repo's personal portfolio site.

## Site Architecture

- Canonical homepage entrypoint: `index.md`
- Main layout: `_layouts/home.html`
- Shared shell: `_includes/header.html`, `_includes/footer.html`
- Primary content source: `_data/profile.yml`
- Styling: `assets/css/main.css`
- Blog placeholder: `blog/index.html`
- Generated output: `_site/` is ignored and must not be committed

Prefer updating `_data/profile.yml` for content changes. Edit layout/CSS only when the requested content cannot be represented cleanly by the existing data model or when the user asks for a visual/structural change.

## Positioning Rules

Optimize for employer visibility, especially AI/ML systems, ML infrastructure, research engineering, computer vision, video generation, and distributed systems roles.

Keep copy:
- concise, specific, and recruiter-scannable
- evidence-led, with concrete technologies, outcomes, links, venues, or metrics when provided
- truthful to supplied source material; do not invent metrics, affiliations, acceptance status, or impact
- professional rather than playful; avoid emoji-heavy or casual phrasing

Current core positioning:
- MS CS student at UC San Diego
- Graduate Student Researcher at Hao AI Lab
- working on FastVideo since March 2026
- prior production systems experience at Cohesity

## Update Workflow

1. Inspect the current source before editing:
   - `git status -sb`
   - relevant parts of `_data/profile.yml`
   - relevant layout/CSS only if the data model or visual presentation may need changes
2. Determine the right content destination:
   - new job, internship, research role, education update: `experience`
   - new project or research system: `projects`
   - paper, demo, preprint, poster, talk with a stable link: `publications`
   - scholarship, grant, award, competition result: `awards`
   - mentoring, TA, service, leadership: `leadership`
   - headline, links, resume availability, location, availability: top-level fields
   - SEO/search phrase changes: `_config.yml`
3. Normalize new entries:
   - keep titles short and searchable
   - use `context` for lab/company/course/venue context
   - include links only when stable and public
   - keep summaries to one compact sentence unless the section already supports bullets
   - prefer outcome verbs: built, shipped, optimized, evaluated, published, led, mentored
4. Preserve section balance:
   - strongest/current ML systems work first
   - production systems credibility visible in the first screen or experience section
   - publications support credibility but should not bury recruiter CTAs
5. Validate:
   - parse YAML: `ruby -e 'require "yaml"; YAML.load_file("_data/profile.yml"); YAML.load_file("_config.yml"); puts "YAML OK"'`
   - build with conda base:
     `conda run -n base env BUNDLE_PATH=/private/tmp/satyam-site-bundle ruby -S bundle exec jekyll build`
   - run `git diff --check`
6. Preview when useful:
   - `conda run -n base env BUNDLE_PATH=/private/tmp/satyam-site-bundle ruby -S bundle exec jekyll serve -H localhost -P 4000`
   - verify `http://localhost:4000`

## Resume Handling

When the user provides a resume PDF:
- place it at `assets/files/resume.pdf`
- set `resume_available: true` in `_data/profile.yml`
- keep `resume_path: "/assets/files/resume.pdf"`
- verify the homepage button points to the PDF after build

If no resume PDF is present, keep `resume_available: false` so the primary CTA stays as an email resume request.

## Design Guardrails

Follow the existing professional product-style design:
- restrained palette, high contrast, dense but readable information
- no decorative clutter, hero gimmicks, audio toggles, or placeholder-heavy sections
- responsive layout must work on mobile and desktop
- avoid adding JavaScript unless the requested feature genuinely needs it
- avoid adding a frontend framework; this is a simple Jekyll/GitHub Pages site

## Commit Hygiene

Do not commit generated `_site/` output or cache directories.

Before committing:
- confirm `git status -sb`
- ensure changed files match the requested update
- run the validation commands above where possible
- use a concise commit message such as `Update portfolio content` or `Add FastVideo project details`
