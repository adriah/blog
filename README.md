# sigkill.no

A personal landing page and notes archive, built with Hugo. The design is **The last signal**: bone-white typography, a red waveform that falls silent, and a dark editorial layout for occasional writing.

The four original concepts are saved in [docs/site-concepts.md](docs/site-concepts.md).

## Develop

Install a current Hugo release (validated with 0.165.0). The Bear Cub theme remains a Git submodule; initialize it if cloning afresh:

```sh
git submodule update --init --recursive
hugo server -D
```

Open the URL Hugo prints, normally `http://localhost:1313/`.

## Write

Articles stay in `content/blog/`, and existing `/blog/` URLs are preserved. The section is labelled **Notes**. Add a note with:

```sh
hugo new content blog/my-note.md
```

Set its title, date, description and optional tags in the front matter, write Markdown, and remove the draft flag (`draft = true` in TOML) when ready. Existing notes use TOML front matter as does the current `archetypes/default.md`.

`content/about.md` supplies the Elsewhere page. `content/_index.md` preserves the original introduction for reference; the new landing page is composed in `layouts/index.html`.

## Design

- `assets/signal.css`: shared palette, typography, landing scene, archive, article and responsive styles.
- `layouts/partials/signal.html`: decorative waveform.
- `layouts/_default/`: shared shell, archive, article and code-block layouts.
- `static/favicon.svg`: signal termination mark.

The production site needs no JavaScript, webfonts, third-party requests or application server. Each 13-second cycle draws the signal left to right in a 1.3-second burst, sends a glow along the full path over 10.4 seconds, then erases the line left to right over 1.3 seconds before repeating. It respects reduced-motion preferences. Code blocks are keyboard-focusable and horizontally scrollable. RSS is available at `/blog/index.xml`; tags and the existing article URLs remain available.

## Build and deploy

```sh
hugo --minify
```

Hugo writes the static site to `public/`. The existing SourceHut `.build.yml` builds and deploys this directory to sigkill.no with rsync.

For an isolated validation build that does not touch existing generated files:

```sh
hugo --minify --destination /private/tmp/sigkill-build
```

No Node dependencies or asset-generation services are required.

A separate, owner-only Sites preview is registered in `.openai/hosting.json`. To build its static output, run `hugo --minify --destination dist --baseURL https://sigkill-last-signal.mnemonic-2945.chatgpt.site/`. Production deployment to sigkill.no still uses the SourceHut workflow above.
