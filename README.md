# Satyam Srivastava Portfolio

Personal website for `satyam-53.github.io`, built with Jekyll and deployed through GitHub Pages.

## Structure

- `_data/profile.yml` contains the homepage content: profile links, experience, projects, publications, awards, and leadership.
- `_layouts/home.html` renders the portfolio page.
- `_includes/header.html` and `_includes/footer.html` hold shared shell UI.
- `assets/css/main.css` contains the full visual system.
- `index.md` is the canonical homepage entrypoint.

## Local Development

This repo uses Ruby/Jekyll through Bundler. In this workspace, keep Ruby commands inside the conda `base` environment:

```bash
conda run -n base /Users/satyam53/.gem/ruby/2.6.0/bin/bundle exec jekyll serve
```

If dependencies are missing, install them before serving:

```bash
conda run -n base gem install --user-install bundler:2.3.3
conda run -n base env BUNDLE_PATH=/private/tmp/satyam-site-bundle /Users/satyam53/.gem/ruby/2.6.0/bin/bundle install
```

Current note: the system Ruby visible from conda `base` is Ruby 2.6.10. If Bundler resolves gems that require Ruby 2.7+, install a newer Ruby inside `base` or use a newer Ruby manager before running the Jekyll build locally.

## Updating Content

Edit `_data/profile.yml` for most website updates. When a resume PDF is ready, add it at `assets/files/resume.pdf` and set:

```yaml
resume_available: true
```
