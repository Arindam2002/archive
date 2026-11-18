# archive

Live site: **https://arindam2002.github.io/archive/**

A personal knowledge base of interview prep material and the posts that grow
out of it. It is a static site built with [Quartz](https://quartz.jzhao.xyz/)
and published from this repo with GitHub Pages on every push to `main`.

## What is on the site

- **Study** (`content/study/`): topic write-ups on backend engineering,
  system design, data structures and algorithms, and LLM infrastructure. Each
  one has an overview diagram, a flashcard deck, and links to the topics it
  builds on, so the graph view shows how the material hangs together. These
  pages are generated, not hand-edited here: they come out of
  [gig-radar](https://github.com/Arindam2002/gig-radar), a job-hunt tool whose
  daily routine writes and deepens study topics from real job descriptions.
  Only topics that have been read and approved in that tool get exported, and
  a topic that changed after approval is held back until it is reviewed again.
- **Blog** (`content/blog/`): posts written by hand, as plain markdown with
  `title`, `date` and `tags` frontmatter. A post can link a study topic with a
  wikilink (`[[some-topic-slug]]`) and it shows up in the same graph. Set
  `draft: true` to keep a post out of the build.

## How content gets here

```
gig-radar study folder  --(publish export)-->  content/study/   --(push)-->  GitHub Pages
your editor             --(write markdown)-->  content/blog/    --(push)-->  GitHub Pages
```

The export owns `content/study/` through a manifest it writes next to the
pages, and it deletes pages that are no longer approved. It never touches
anything else in `content/`.

## Running it locally

```bash
npm ci
npx quartz plugin restore
npx quartz build --serve
```

Then open http://localhost:8080/. The deploy workflow runs the same build on
every push to `main` and publishes `public/` to GitHub Pages.

## Diagrams in blog posts

Blog diagrams are drawn in Excalidraw and embedded as exported SVGs:

- `diagram-sources/<post-slug>/` holds the editable `.excalidraw` scenes. It
  sits outside `content/`, so it is never published.
- `content/blog/diagrams/<post-slug>/` holds the exported `.svg` files the
  post embeds.

To change a diagram, edit its `.excalidraw` file and export it again with the
exporter from the gig-radar checkout:

```bash
node <gig-radar>/tools/export_diagram.mjs diagram-sources/<post-slug>/<name>.excalidraw content/blog/diagrams/<post-slug>/<name>.svg
```

Embed it with a path from the content root, and put an italic line directly
under the image for a caption:

```markdown
![Alt text describing the diagram](blog/diagrams/<post-slug>/<name>.svg)
*Caption text.*
```

## Layout

```
content/
  index.md        landing page
  study/          exported topics, diagrams/ next to them (generated)
  blog/           hand-written posts
    diagrams/     exported SVGs, one folder per post
diagram-sources/  editable Excalidraw scenes, one folder per post (not published)
private/          local notes, git-ignored
quartz.*          site configuration and layout
.github/workflows/deploy.yml   builds and publishes on push to main
```
