---
layout: blog
title: Hello 2025!
date: 2025-01-01T11:46:15-06:00
lastMod: 2025-01-01T14:11:02-06:00
categories:
  - log-entry
tags:
  - website
  - hugo
  - css
  - obsidian
  - sass
  - dart-sass
  - cloudflare
  - github
  - docker
  - bulma
  - markdown
description: Kicking off 2025 with a barely noticeable change to this blog.
disableComments: true
draft: false
---

Starting off 2025 with a small but major update to this blog. I've spent the past few days working on some needed maintenance for the theme and how this blog is deployed and figured I'd capture this information in the first blog post of the year.

## Updating Blog Post Creation workflow

How I have this blog structured is that I utilized 3 separate Github repositories to ultimately build the blog with [Hugo](https://gohugo.io/)

- [Main Build](https://github.com/sinicide/huynguyen-blog-build)
- [Theme](https://github.com/sinicide/hugo-theme-hello-bulma)
- [Content](https://github.com/sinicide/huynguyen-blog-content)

This allows for flexibility if I one day decided I wanted to move to a different Static Site Generator or if I want change up the theme without specifically affecting the other repos. Obviously I could keep everything in a single repo, but I really want to utilize git's changelog to track markdown publicly.

Hugo can generate blog posts via the following command.

```
hugo new content <content_location>/blog_post.md
```

This will generate a markdown file using pre-determined frontmatter at the location specified by the path. While this is nice, it requires me to run Hugo to do this and it also requires me to manually update the `lastMod` frontmatter whenever I make edits. I include this particular frontmatter because I value being transparent when it comes to information edits. This allows readers of my blog, who are observant to then check out the changelog link at the very bottom of this post to see what has been changed and when. This way I can utilize git's changelog as opposed to keeping track elsewhere. Why is this even important? Well I believe public information can change over time and everyone deserves to understand those changes to better stay informed and formulate their own opinions on that information.

Instead of using Hugo to generate new blog posts, I've opted to use an [Obsidian](https://obsidian.md/) vault to really power up my blog post creation flow.

## Why use Obsidian?

I've been using Obsidian for note taking in my private and professional life. At it's base level, it's just a super neat markdown note taking app, but with a few community plugins it allows me to automate blog post generation as well as `lastMod` frontmatter update on changes.

In particular I'm using the following 2 Community Plugins

- [Templater](https://github.com/SilentVoid13/Templater)
- [Update time on edit](https://github.com/beaussan/update-time-on-edit-obsidian)

This gives a fairly nice gui experience as I can accomplish a lot of different workflows via the Command Palette. This allows me to automate including specific Hugo shortcodes from a templates folder and/or create a blog post based on a skeleton markdown file.

What's neat is that the Obsidian configuration itself now lives in this content repo as a dot folder, which is handy because Hugo will ignore this directory in content. However the templates that I use are in a regular folder, which means I also needed to update the Main Build repo to ignore it as non-content otherwise it'll break when trying to compile the site.

This required a new inclusion of content [module](https://gohugo.io/hugo-modules/configuration/#module-configuration-mounts) in Hugo's configuration yaml to exclude it from being processed. What's cool about this is that I could also specify different content directories within the same repo for different environment builds if I wanted. But for now, I only need it as a way to exclude a specific directory.

```yaml
module:
  mounts:
    - source: content
      target: content
      excludeFiles:
        - "templates/**"
```

This was all done and completed with the last [post]({{< ref "/blog/2024/12/26/favorite-thangs-in-2024/index" >}} "Favorite Thangs in 2024") I made.

## Updating the Theme

Now let's talk about what I updated for the theme. In case you're not aware, my theme is based on a no longer maintained public theme called [Hello Friend](https://github.com/panr/hugo-theme-hello-friend) which I've refactored with the [Bulma](https://bulma.io/) Framework into what I've cutely named [Hello Bulma](https://github.com/sinicide/hugo-theme-hello-bulma). Dragon Ball Z was a big part of my childhood, so I have a soft spot for this CSS framework.

- Updated Font Awesome Free 6 to `6.7.2`, prior version used was `6.4.2` (so not a major jump)
- Updated Bulma CSS Framework to v1
- Update local Hugo Container to use version `0.140.1` in Main Build repo

### Updating the Dockerfile

In the main repo, I have a local Dockerfile, which I use to build a local Hugo container to stand up my website for local testing. This allows me to not have to install Hugo locally, because I run Archlinux (by the way), which means that I'm at the mercy of a Rolling Release structure for packages. So if I install Hugo locally via pacman, it'll be updated whenever there's a new update. That's not what I want for anything I push to production, anything that is going to be considered "production" needs to be deterministic and remain on consistent versions.

So let's go over some of the major changes to my Dockerfile

```dockerfile
ENV HUGO_VERSION 0.140.1
...
ENV DART_SASS_VERSION 1.83.0
ENV DART_SASS_FILENAME dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz
ENV DART_SASS_URL https://github.com/sass/dart-sass/releases/download/${DART_SASS_VERSION}/${DART_SASS_FILENAME}

# download Dart Sass
RUN cd /tmp && \
curl -O -L ${DART_SASS_URL} && \
tar -xf ${DART_SASS_FILENAME} && \
cp -r /tmp/dart-sass/* /usr/local/bin && \
rm -rf /tmp/dart-sass*
```

- First and foremost is setting the `HUGO_VERSION` to a newer version
- I've also included Dart Sass installation

The inclusion of Dart Sass is necessary here because by default Hugo uses LibSass to transpile Sass to CSS and LibSass is deprecated by the Sass team in favor of Dart Sass as of 2020. Unfortunately the extended Hugo release packages doesn't contain this so I'm forced to manually install this for local development. Luckily for Cloudflare Pages (where this blog is hosted), I don't have to worry about this as Dart Sass is pre-installed in the Cloudflare Workers when generating the static files for Hugo.

So for the most part I just copy-pasted what was in [Hugo's Documentation](https://gohugo.io/hugo-pipes/transpile-sass-to-css/#dart-sass) opting for Dart Sass version `1.83.0`

The use of Dart Sass is also crucial here because Bulma V1 is built with it according to their [Migration documentation](https://bulma.io/documentation/start/migrating-to-v1/#what-changes).

### Migrating to Bulma v1

As with any major software changes, it's always good to read any official documentation to account for any breaking changes. Luckily Bulma provides us 2 major recommendations/changes.

- `Tiles` are deprecated in favor of `Smart Grid` (I don't use either so this is irrelevant to me)
- Use of `@import` is not recommended

Now I've been using `@import` to import specific components from the Bulma sass directory to build the CSS as I have my own modifications and additions. So this means I need to migrate from `@import` to `@forward` or `@use` in the main file that my Theme points to for css compiling.

Unfortunately Font Awesome 6 still uses `@import` in it's scss files, so I'll be keeping this around until it's change.

But now my `bulma.scss` file looks like

```scss
// bulma.io v1.0.3
@use "../bulma/sass/<component>" (
	//my overrides
);
...
@forward "../bulma/sass/<component>";
...

// CUSTOM
@forward "./<components>";
...

// Font Awesome 6.7.2
@import "../fontawesome/<filename>.scss";
...
```

Apparently the ordering of `@use` , `@forward` , and `@import` here is important. I ran into some errors when compiling the CSS, so learned a lesson on that ordering. Had to spend a bit of time reading through the [Sass documentation](https://sass-lang.com/documentation/at-rules/forward/) on the usage of `@forward` and doing a bit of back and forth testing to ensure everything compiled and rendered like before.

Finally one more thing to note is how do we tell Hugo to use Dart Sass to compile these Sass files.

```htmx
<!-- Theme CSS -->
{{ $opts := dict "transpiler" "dartsass" }}
{{ $style := resources.Get "scss/custom/bulma.scss" | toCSS $opts | minify | fingerprint }}
<link rel="stylesheet" href="{{ $style.RelPermalink }}" integrity="{{ $style.Data.Integrity }}" crossorigin="anonymous" />
```

This follows [Hugo](https://gohugo.io/hugo-pipes/transpile-sass-to-css/#usage)'s Guidance on the use of `toCSS` with optional flags, unless specified it'll default to use LibSass.

## Pushing to Production

With the theme update it was time to create a Pull Request to push the changes to master branch, which in turned triggered a commit on the Main Build repo with the new theme changes and is watched by Cloudflare for any changes to my Main Build repo to trigger a new deployment.

So as you can see pretty much everything is intact. One minor change I did decide to do was change the strong text colors for headers and links to a white-ish color instead of the previous dark grey-ish ones, which should help with readability I suppose.

Anyhow, Happy New Years! Here's to accomplishing more things in 2025!
