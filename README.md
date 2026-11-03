# Security Notebook
Jekyll blog served by GitHub Pages.

## Write a post
1. Copy `_drafts/post-template.md` to `_posts/YYYY-MM-DD-slug.md`.
2. Fill it in, run the pre-publish check in the template. Set `categories:` to one folder slug from `_data/folders.yml` (tools, homelab, detection, lessons); the homepage folders fill themselves.
3. `git add . && git commit && git push`; Pages rebuilds in ~1 minute.

## Safety hook
`cp scripts/pre-commit .git/hooks/pre-commit` blocks commits containing private IPs, keys or likely tokens.
Remember: git history is permanent, so catch it before commit, not after.

## Folders
The homepage is a set of folders defined in `_data/folders.yml`. To add one: add an entry there, then add `folders/<slug>.md` (copy an existing one). Theme lives in `_layouts/` and `assets/css/theme.css`.
