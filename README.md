# Security Notebook
Jekyll blog served by GitHub Pages.

## Write a post
1. Copy `_drafts/post-template.md` to `_posts/YYYY-MM-DD-slug.md`.
2. Fill it in, run the pre-publish check in the template.
3. `git add . && git commit && git push`; Pages rebuilds in ~1 minute.

## Safety hook
`cp scripts/pre-commit .git/hooks/pre-commit` blocks commits containing private IPs, keys or likely tokens.
Remember: git history is permanent, so catch it before commit, not after.
