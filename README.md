# ChrisRanAn's Blog

Personal blog and study notes.

Live site: <https://chrisranan1.github.io/blog/>

## Writing a new post

1. Create a file in `_posts/` named `YYYY-MM-DD-slug.md`.
2. Add front matter, for example:

   ```yaml
   ---
   title: "My new post"
   date: 2026-08-23
   categories: [solana, notes]
   tags: [runtime, svm]
   excerpt: "One-sentence teaser."
   ---
   ```

3. Write in Markdown. Fenced code blocks (```rust) get syntax highlighting.
4. `git add`, `git commit`, `git push` — GitHub Pages rebuilds automatically.

## Local preview

```sh
bundle install
bundle exec jekyll serve
# http://127.0.0.1:4000/blog/
```

Built with [Jekyll](https://jekyllrb.com/) and the [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/) theme.
