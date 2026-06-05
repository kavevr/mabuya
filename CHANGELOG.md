# Changelog

## Unreleased

- Migrated the theme to the latest Zola layout conventions, replacing `config.toml` with `zola.toml` and adopting the newer section, taxonomy, page, and index template structure.
- Consolidated shared template logic and metadata rendering, including summaries, updated dates, reading time, word count, author information, pagination, and generic taxonomy variables.
- Removed obsolete template files such as the old tag-specific taxonomy templates and the separate head partial after folding head markup into `base.html`.
- Updated theme documentation and compatibility notes for Zola `0.22.1`, including README references to `zola.toml`.
- Cleaned up sample content so blog content is section-backed and uses valid front matter.
- Improved URL handling by using Zola internal links and taxonomy resolution instead of hard-coded paths, including the 404 home link.
- Reworked the back-to-top control to use a standard button with an event listener, and fixed the footnote offset typo in the styles.
- Adjusted link checking so external links warn while internal link validation remains strict.
- Verified the rewrite with `zola check` and `zola build`.
