+++
date = '2026-06-21T18:47:39+07:00'
draft = false
title = 'Deploy Static Site With Hugo and Cloudflare Workers'
tags = ['hugo', 'cloudflare', 'cloudflare-workers', 'static-site', 'papermod', 'github', 'devops']
+++
In this post, I’ll walk through how I deployed my personal static blog using [Hugo](https://gohugo.io/), the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme, [GitHub](https://github.com/), and [Cloudflare Workers](https://developers.cloudflare.com/workers/).

> **A note on Workers vs. Pages:** Cloudflare is consolidating static site hosting from Cloudflare Pages into Cloudflare Workers. As of 2025, [Workers Static Assets](https://developers.cloudflare.com/workers/static-assets/) is the recommended way to host new static sites, and Pages is in maintenance mode. In the dashboard both still live under the unified **Workers & Pages** section, which is why the deploy commands below use `wrangler deploy` (a Workers command) rather than the older Pages workflow.

This is the setup I use for my own blog:

* GitHub repository: [vancanhuit/personal-blog](https://github.com/vancanhuit/personal-blog)
* Live site: [blog.canhdinh.com](https://blog.canhdinh.com/)
* Static site generator: [Hugo](https://gohugo.io/)
* Theme: [PaperMod](https://github.com/adityatelange/hugo-PaperMod)
* Hosting/deployment: [Cloudflare Workers (Static Assets)](https://developers.cloudflare.com/workers/static-assets/)

The final Cloudflare build and deploy commands are:

```bash
git submodule update --init --recursive && hugo --gc --minify
```

```bash
npx wrangler deploy --assets=public --name personal-blog --compatibility-date 2026-06-21
```

No `wrangler.jsonc` file is required in the repository for this setup.

## Why Hugo?

[Hugo](https://gohugo.io/) is a static site generator written in Go. It is fast, simple, and works very well for technical blogs.

A static blog is a good fit for my use case because I mostly write posts that contain:

* Markdown content
* Source code examples
* DevOps notes
* Infrastructure experiments

Since Hugo generates plain HTML, CSS, JavaScript, images, and fonts, the final site is easy to host on Cloudflare.

## Why PaperMod?

I chose [PaperMod](https://github.com/adityatelange/hugo-PaperMod) because it is clean, fast, and works well for technical writing.

Some features I wanted:

* Clean blog layout
* Dark/light mode
* Table of contents
* Reading time
* Tags and categories
* Good source code block rendering
* Code copy button
* Minimal maintenance

For a technical blog, PaperMod is a practical choice because it stays out of the way and lets the content stand out.

## Create the Hugo site

Create a new Hugo project:

```bash
hugo new site personal-blog
cd personal-blog
git init
```

Then add PaperMod as a Git submodule:

```bash
git submodule add https://github.com/adityatelange/hugo-PaperMod themes/PaperMod
```

Because PaperMod is added as a submodule, the repository will include a `.gitmodules` file.

You can check it with:

```bash
cat .gitmodules
```

Example:

```ini
[submodule "themes/PaperMod"]
 path = themes/PaperMod
 url = https://github.com/adityatelange/hugo-PaperMod
```

## Configure Hugo

Here is a simplified `hugo.toml` configuration:

```toml
baseURL = "https://blog.canhdinh.com/"
locale = "en-us"
title = "Canh Dinh"
theme = "PaperMod"

[pagination]
pagerSize = 10

[params]
env = "production"
description = "Notes on DevOps practices, self-hosting and software delivery."
author = "Canh Dinh"

defaultTheme = "auto"
disableThemeToggle = false

ShowReadingTime = true
ShowShareButtons = false
ShowPostNavLinks = true
ShowBreadCrumbs = true
ShowCodeCopyButtons = true
ShowWordCount = true
ShowRssButtonInSectionTermList = true
ShowToc = true
TocOpen = false
UseHugoToc = true

[[params.socialIcons]]
name = "github"
url = "https://github.com/vancanhuit"

[[menu.main]]
identifier = "posts"
name = "Posts"
url = "/posts/"
weight = 10

[[menu.main]]
identifier = "tags"
name = "Tags"
url = "/tags/"
weight = 20

[[menu.main]]
identifier = "archives"
name = "Archives"
url = "/archives/"
weight = 30

[markup]
[markup.highlight]
codeFences = true
guessSyntax = true
lineNos = true
lineNumbersInTable = true
noClasses = false
```

The most important values are:

```toml
baseURL = "https://blog.canhdinh.com/"
theme = "PaperMod"
```

The `baseURL` should match the production domain.

## Add a blog post

Create a new post:

```bash
hugo new content posts/hello-world.md
```

Example post:

````markdown
---
title: "Hello World"
date: 2026-06-21T10:00:00+07:00
draft: false
tags: ["hugo", "cloudflare", "devops"]
categories: ["devops"]
showToc: true
---

This is my first post using Hugo, PaperMod, GitHub, and Cloudflare.

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello from my blog")
}
```
````

Make sure the post has:

```yaml
draft: false
```

If `draft` is set to `true`, Hugo will not publish it in production builds.

## Run locally

For local development, run:

```bash
hugo server -D --disableFastRender
```

Then open:

```text
http://localhost:1313
```

The `-D` flag includes draft posts.

If the site looks stale after changing CSS, fonts, or theme files, clean the generated output:

```bash
rm -rf public resources/_gen
hugo --gc --minify
hugo server -D --disableFastRender
```

## Self-host fonts

I use Maple Mono NL for code blocks.

Font files are stored under:

```text
static/fonts/maple-mono/
```

For example:

```text
static/fonts/maple-mono/MapleMonoNL-Regular.ttf.woff2
static/fonts/maple-mono/MapleMonoNL-Bold.ttf.woff2
static/fonts/maple-mono/MapleMonoNL-Italic.ttf.woff2
```

Then I define the font in:

```text
assets/css/extended/custom-fonts.css
```

Example:

```css
@font-face {
  font-family: "Maple Mono NL";
  src: url("/fonts/maple-mono/MapleMonoNL-Regular.ttf.woff2") format("woff2");
  font-weight: 400;
  font-style: normal;
  font-display: swap;
}

@font-face {
  font-family: "Maple Mono NL";
  src: url("/fonts/maple-mono/MapleMonoNL-Bold.ttf.woff2") format("woff2");
  font-weight: 700;
  font-style: normal;
  font-display: swap;
}

code,
pre,
kbd,
samp,
.highlight pre,
.highlight code,
.chroma,
.chroma code {
  font-family: "Maple Mono NL", ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", monospace;
  font-feature-settings: "liga" 0, "calt" 0;
  font-variant-ligatures: none;
}
```

I disable ligatures because I prefer code to display exactly as typed.

## Push to GitHub

Commit and push the project:

```bash
git add .
git commit -m "Initial Hugo blog with PaperMod"
git branch -M main
git remote add origin https://github.com/vancanhuit/personal-blog.git
git push -u origin main
```

My repository is available here:

```text
https://github.com/vancanhuit/personal-blog
```

## Configure Cloudflare deployment

In the Cloudflare dashboard, go to **Workers & Pages**, create a new Worker, connect the GitHub repository, and configure the project with these commands.

### Build command

```bash
git submodule update --init --recursive && hugo --gc --minify
```

This does two things:

1. Fetches the PaperMod theme submodule.
2. Builds and minifies the Hugo site into the `public/` directory.

The `git submodule update --init --recursive` part is important because PaperMod is stored as a Git submodule.

### Deploy command

```bash
npx wrangler deploy --assets=public --name personal-blog --compatibility-date 2026-06-21
```

This deploys the generated `public/` directory as static assets.

In this setup, I do not need a `wrangler.jsonc` file in the repository because the deploy command provides the required Workers deployment options directly:

```bash
--assets=public
--name personal-blog
--compatibility-date 2026-06-21
```

## Environment variables

I also recommend setting `HUGO_VERSION` in Cloudflare:

```text
HUGO_VERSION = 0.163.3
```

Using a fixed Hugo version helps avoid unexpected build differences between local and Cloudflare environments.

## Normal publishing workflow

After the initial setup, publishing a new post is simple.

Create a post:

```bash
hugo new content posts/my-new-post.md
```

Preview locally:

```bash
hugo server -D --disableFastRender
```

When ready, commit and push:

```bash
git add .
git commit -m "Add new blog post"
git push
```

Cloudflare will automatically rebuild and redeploy the site.

## Troubleshooting

### Theme is missing

If Cloudflare cannot find PaperMod, make sure the theme submodule is committed:

```bash
git submodule status
cat .gitmodules
```

Also make sure the build command includes:

```bash
git submodule update --init --recursive
```

### Code blocks look broken

Clean Hugo’s generated files locally:

```bash
rm -rf public resources/_gen
hugo --gc --minify
hugo server -D --disableFastRender
```

Also check the Markdown code fence format:

````markdown
```go
fmt.Println("hello")
```
````

### New font does not appear

Hard refresh the browser:

```text
Ctrl + Shift + R
```

On macOS:

```text
Cmd + Shift + R
```

If fonts are cached aggressively, rename the font folder or font files and update the CSS paths.

### Deployment fails after successful build

If the log shows that Hugo built successfully but deployment failed, check the deploy command.

For this setup, use:

```bash
npx wrangler deploy --assets=public --name personal-blog --compatibility-date 2026-06-21
```

not just:

```bash
npx wrangler deploy
```

## Useful links

* [Hugo official website](https://gohugo.io/)
* [Hugo documentation](https://gohugo.io/documentation/)
* [PaperMod GitHub repository](https://github.com/adityatelange/hugo-PaperMod)
* [PaperMod installation guide](https://github.com/adityatelange/hugo-PaperMod/wiki/Installation)
* [Cloudflare Workers](https://developers.cloudflare.com/workers/)
* [Cloudflare Workers static assets](https://developers.cloudflare.com/workers/static-assets/)
* [Migrate from Pages to Workers](https://developers.cloudflare.com/workers/static-assets/migration-guides/migrate-from-pages/)
* [Wrangler deploy command](https://developers.cloudflare.com/workers/wrangler/commands/#deploy)

## Final setup

The final setup is:

```text
Hugo static site
PaperMod theme
GitHub repository
Cloudflare build command:
  git submodule update --init --recursive && hugo --gc --minify

Cloudflare deploy command:
  npx wrangler deploy --assets=public --name personal-blog --compatibility-date 2026-06-21

No wrangler.jsonc required
```
