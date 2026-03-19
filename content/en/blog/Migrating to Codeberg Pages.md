---
tags:
  - Blog
  - Codeberg
  - Codeberg Pages
  - Forgejo Actions
  - Hugo
  - Listed
  - PaperMod
title: Migrating to Codeberg Pages
description: For various reaons, I migrated to Codeberg Pages for my blog.
---

If anyone is wondering, I started my writing journey with [Listed](https://listed.to/ "Listed — Welcome to your new public journal.") blogging platform by [Standard Notes](https://standardnotes.com/ "Standard Notes | End-To-End Encrypted Notes App"). I think it had served me well, but for reasons I will soon mention, I ultimately decided to find a new home.

# Why the switch?

Just about three and a half months ago, I wrote the following while [working on improving my code snippets' legibility](/blog/2025/12/improving-listeds-code-legibility-in-dark-theme/ "Improving Listed's code legibility in dark theme") on the blogging platform:

> It has been some time since I started writing on [Listed](https://listed.to/ "Listed — Welcome to your new public journal."). Although the development around the blogging platform seems to be slow-paced (which I feel is an understatement) and some note-taking services like [Anytype](https://anytype.io/ "The Everything App") also offer [publication of notes](https://doc.anytype.io/anytype-docs/getting-started/web-publishing "Publish | Anytype Docs"), I have yet to find a compelling reason nor devote my time for migration.

At that time, despite being real concerns, problems I came across during its usage were, at worst, inconveniences to me. However, as I wrote more, I felt the need for a switch; the reasons why are as follows:

## Version-controlled blog

I write blog posts in [Markdown](https://daringfireball.net/projects/markdown/ "Daring Fireball: Markdown") format. Documents using this format, however, appear in just about every open-source project's [Git](https://git-scm.com/ "Git") repositories; it was, to me, as if they were meant to be version-controlled.

Making my blog a Git-based project was, while not a priority, something I wished to do. An added benefit was that, since I always [sign Git commits](https://git-scm.com/book/en/v2/Git-Tools-Signing-Your-Work "Git - Signing Your Work") with my OpenPGP key, anyone would be able to verify that I am the actual author.

## My desire for sharing open knowledge (and its obstacle)

Another non-critical, yet concerning nonetheless, problem was that Listed puts the following text to my blog's footer:

> Copyright © 2026 이영욱

This will likely not be a problem for many, but it was not desirable to me. I started writing in the hope of sharing what I have learned, but the copyright notice was the opposite of what I wanted my work to be labelled with.

As such, I aimed to dedicate my work to the public domain during the migration. To do so, I used [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/ "Deed - CC0 1.0 Universal - Creative Commons") as a way to waive my rights.

## A series of workarounds

While previous points were more about my wish, this one pertains to actual problems. I have a few examples:

- Inline code (and possibly other formatted text) is ignored within hyperlinks. It means that writing ``[`example`](https://example.org/)`` will render the text as just "[example](https://example.org/)" (and not "[`example`](https://example.org/)").
- Some HTML tags are ignored and not rendered. That in itself is not a problem, but:
	- I do wish to use some of them for features Listed (and sometimes Markdown itself) does not support, like strikethrough text or [`<picture>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/picture "<picture>: The Picture element - HTML | MDN") elements.
	- I cannot specify which tags to whether allow or prohibit
- Same-page navigation seems to do full page loads.
- Code snippets become less legible in dark theme.

I was able to solve some of those issues, but when the solutions look something like [adding a bunch of CSS](/blog/2025/12/improving-listeds-code-legibility-in-dark-theme/#applying-the-theme "Improving Listed's code legibility in dark theme") to replace the default theme, I do not consider them to be proper fixes.

It would be great if the issues are addressed upstream, but...

## The uncertainty

Code for Listed is available at [its Git repository](https://github.com/standardnotes/listed "standardnotes/listed: Create an online publication with automatic email newsletters. https://listed.to"). [Exactly 2 commits were made](https://github.com/standardnotes/listed/commits/master/?since=2025-03-11&until=2026-03-18 "Commits · standardnotes/listed") since I published my first post about a year ago, where the more recent one was there [to remove support for custom domains](https://github.com/standardnotes/listed/commit/d7e82ea3725148d1dbc5960aa147b1aa4da8a215 "chore: remove custom domain from settings (#302) · standardnotes/listed@d7e82ea") (that I [have worked hard](/blog/2025/04/using-custom-domain-with-listed/ "Using custom domain with Listed") to enable).

Those at Standard Notes are not really known for [actively implementing new features](https://standardnotes.com/longevity "Built to Last | Standard Notes End-To-End Encrypted Notes App"), but with [old versions of dependencies](/blog/2025/04/using-custom-domain-with-listed/#:~:text=It%20looked%20like%20this%20project’s%20dependencies%20are%20quite%20old. "Using custom domain with Listed") still being used somewhere in the code, it looked closer to being unmaintained.

Another writer that I have previously come across while browsing Listed was also seen migrating away from the blogging platform, although [reasons for the switch](https://kieran.colfer.net/blog/moving-away-from-listed/ "Why I'm moving away from Listed.to | kieran.colfer.net") seemed to be a little different (but genuine nonetheless).

With [Proton](https://proton.me/ "Proton: Privacy by default")'s merger with the note-taking service, I believe those working on the product did not just abandon it, at least not without a reason. However, while I wish the best for them, I will do what I can do to make my environment favourable to me.

# Tools of choice

## [Software forge](https://en.wikipedia.org/wiki/Forge_(software) "Forge (software) - Wikipedia")

I primarily use [GitHub](https://github.com/ "GitHub · Change is constant. GitHub keeps you ahead. · GitHub") for most (if not all) of my projects. It was perfectly fine to do so again, but I wanted to give myself a new challenge with a different environment. Some things about the Microsoft-owned service, while not necessarily applicable to this scenario, also irked me enough to pursue a change:

- They do not support repositories with [SHA-256 hash function](https://git-scm.com/docs/hash-function-transition "Git - hash-function-transition Documentation"), which [GitLab managed to do](https://about.gitlab.com/blog/gitlab-now-supports-sha256-repositories/ "GitLab now supports SHA256 repositories") a while ago.
- Some parts of their infrastructure is still IPv4-only, requiring [workarounds](https://gh-v6.com/) to deploy [NixOS](https://nixos.org/ "Nix & NixOS | Declarative builds and deployments") on IPv6-only cloud instances.

[GitLab](https://about.gitlab.com/ "Finally, AI for the entire software lifecycle.") could be seen as an alternative, but I had a different idea.

I had an account at [Codeberg](https://codeberg.org/ "Codeberg.org") for a while. The service aims to be home for open-source projects, so it felt like a great place to put my contents to. They also offer [Codeberg Pages](https://codeberg.page/ "Codeberg Pages - static pages for your projects"), similar to [GitHub Pages](https://docs.github.com/pages "GitHub Pages documentation - GitHub Docs"), which became my choice of web hosting service.

However, since [contribution graph](https://docs.github.com/en/account-and-profile/concepts/contributions-on-your-profile#about-your-contribution-graph "Contributions on your profile - GitHub Docs") might be of importance, I later decided to mirror my repository from the non-profit, community-led service.

## Static site generator

[The GitHub Pages documentation](https://docs.github.com/pages "GitHub Pages documentation - GitHub Docs") suggests using [Jekyll](https://jekyllrb.com/ "Jekyll • Simple, blog-aware, static sites | Transform your plain text into static websites and blogs") static site generator, which would turn Markdown documents into HTML pages. With some people I know setting up their websites this way, it sure seemed like one of the most common ways to reach the goal.

However, I did not want to go down that road. It may be for the sake of being different, but I also had to consider the build speed.

To build the website, I planned to use [Forgejo Actions](https://forgejo.org/docs/next/user/actions/reference/ "Forgejo Actions | Reference | Forgejo – Beyond coding. We forge.") hosted by Codeberg. Unlike big corporations, though, their users [are asked](https://codeberg.org/actions/meta/src/commit/a4b2c34e7020e6372690345bdb0b9ae56d3a9820#rules-and-conditions "actions/meta: Information and discussions around hosted Forgejo Actions at Codeberg - Codeberg.org") to keep the resource usage to a minimum; although I have not tried using Jekyll at this point, resources I could find online told me that it is not the fastest option available. A reasonably fast, yet still commonly used, option would help reduce build time, lessening the burden to Codeberg's infrastructure.

That is when I was introduced to [Hugo](https://gohugo.io/ "The world's fastest framework for building websites").

> Written in Go, optimized for speed and designed for flexibility. With its advanced templating system and fast asset pipelines, Hugo renders a large site in seconds, often less.

It soon became my choice of tool to deploy my website. There were [a set of Hugo themes](https://themes.gohugo.io/ "Hugo Themes") for me to choose from, where I picked [PaperMod](https://themes.gohugo.io/themes/hugo-papermod/ "PaperMod"); it seemed simple enough and closer to what Listed offered, compared to other themes that I have come across.

# Setting up the project

I started with the static site generator's [quick start guide](https://gohugo.io/getting-started/quick-start/ "Quick start"). However, I immediately encountered a problem.

> Before you begin this tutorial you must:
>
> 1. [Install Hugo](https://gohugo.io/installation/) (extended or extended/deploy edition, v0.156.0 or later)
> 2. [Install Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
>
> You must also be comfortable working from the command line.

[Nixpkgs](https://github.com/NixOS/nixpkgs "NixOS/nixpkgs: Nix Packages collection & NixOS"), on the other hand, only offers Hugo v0.155.3 at the time of writing.

```console
[lyuk98@framework:~]$ nix run nixpkgs#hugo -- version
hugo v0.155.3+extended+withdeploy linux/amd64 BuildDate=unknown VendorInfo=nixpkgs
```

I could see that some significant changes were made since the release of v0.155, including the command to set up the directory structure. As such, I decided to grab the latest version of its binary from the [release page](https://github.com/gohugoio/hugo/releases/tag/v0.158.0 "Release v0.158.0 · gohugoio/hugo").

```console
[lyuk98@framework:~]$ nix run nixpkgs#wget -- https://github.com/gohugoio/hugo/releases/download/v0.158.0/hugo_0.158.0_linux-amd64.tar.gz

[lyuk98@framework:~]$ echo "d0d8f0735dccef76e900719a70102f269c418e010a02e3e0f9e206a208346e2f hugo_0.158.0_linux-amd64.tar.gz" | sha256sum --check
hugo_0.158.0_linux-amd64.tar.gz: OK

[lyuk98@framework:~]$ tar --extract --file hugo_0.158.0_linux-amd64.tar.gz
```

A new project was set up with the downloaded binary. The command to do so was previously known as `hugo new site`, but it was changed to a different one.

```console
[lyuk98@framework:~]$ ./hugo new project pages --format yaml
Congratulations! Your new Hugo project was created in /home/lyuk98/pages.

Just a few more steps...

1. Change the current directory to /home/lyuk98/pages.
2. Create or install a theme:
   - Create a new theme with the command "hugo new theme <THEMENAME>"
   - Or, install a theme from https://themes.gohugo.io/
3. Edit hugo.yaml, setting the "theme" property to the theme name.
4. Create new content with the command "hugo new content <SECTIONNAME>/<FILENAME>.<FORMAT>".
5. Start the embedded web server with the command "hugo server --buildDrafts".

See documentation at https://gohugo.io/.

[lyuk98@framework:~]$ cd pages/
```

I used [YAML](https://yaml.org/ "- YAML Ain't Markup Language"), since I was more familiar with it (with experiences in troubleshooting Kubernetes-related services) than [TOML](https://toml.io/ "TOML: Tom's Obvious Minimal Language").

Git was set up afterwards. Since I wanted to try its SHA-256 hash function, I ran `git init` with the option `--object-format`.

```console
[lyuk98@framework:~/pages]$ git init --object-format=sha256 --initial-branch=main
Initialized empty Git repository in /home/lyuk98/pages/.git/
```

However, when I tried adding the theme with `git submodule add`, I realised that mismatch in hash algorithm would not let me progress further.

```console
[lyuk98@framework:~/pages]$ git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
Cloning into '/home/lyuk98/pages/themes/PaperMod'...
remote: Enumerating objects: 7908, done.
remote: Counting objects: 100% (22/22), done.
remote: Compressing objects: 100% (10/10), done.
remote: Total 7908 (delta 15), reused 12 (delta 12), pack-reused 7886 (from 2)
Receiving objects: 100% (7908/7908), 9.56 MiB | 4.40 MiB/s, done.
Resolving deltas: 100% (4659/4659), done.
error: cannot add a submodule of a different hash algorithm
error: unable to index file 'themes/PaperMod/'
fatal: adding files failed
fatal: Failed to add submodule 'themes/PaperMod'
```

It was disappointing, but as using submodules was (to me) the cleanest way to add themes, I reinitialised the directory with `git init`, while using SHA-1 this time.

```console
[lyuk98@framework:~/pages]$ rm --force --recursive .git/

[lyuk98@framework:~/pages]$ git init --object-format=sha1 --initial-branch=main
Initialized empty Git repository in /home/lyuk98/pages/.git/
```

The submodule was then added, which completed successfully this time.

```console
[lyuk98@framework:~/pages]$ rm --force --recursive themes/PaperMod/

[lyuk98@framework:~/pages]$ git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

[`hugo.yaml`](https://codeberg.org/lyuk98/pages/src/commit/464b586b198cfc6e6b2497d9ec4c15211b9031bf/hugo.yaml "pages/hugo.yaml at 464b586b198cfc6e6b2497d9ec4c15211b9031bf - lyuk98/pages - Codeberg.org") was then edited while referring to documentation by both [Hugo](https://gohugo.io/documentation/ "Hugo Documentation") and [PaperMod](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-installation/ "Install / Update PaperMod | PaperMod"). Notably, [multilingual mode](https://gohugo.io/content-management/multilingual/ "Multilingual mode") was set up, the `copyright` option was edited to indicate CC0 declaration, and [Git-based metadata access](https://gohugo.io/methods/page/gitinfo/ "GitInfo") was configured.

When the project was ready, I tried running the server.

```console
[lyuk98@framework:~/pages]$ ../hugo server
```

I could now see how my future website would look like.

<picture>
  <source srcset="https://images.lyuk98.com/d579cf32-d33d-48f7-b5d3-59a9fd945ec5.avif" media="(prefers-color-scheme: dark)" type="image/avif">
  <source srcset="https://images.lyuk98.com/9208c312-52b7-4e77-bff1-bc2dcd488501.avif" type="image/avif">
  <source srcset="https://images.lyuk98.com/d579cf32-d33d-48f7-b5d3-59a9fd945ec5.webp" media="(prefers-color-scheme: dark)">
  <img src="https://images.lyuk98.com/9208c312-52b7-4e77-bff1-bc2dcd488501.webp" alt="The new website, with no post available">
</picture>

# Moving blog posts

It was time to put my posts into the new home. First, I exported them from the Standard Notes client.

<picture>
  <source srcset="https://images.lyuk98.com/f75cf2db-3645-4f0a-8387-e5f671357719.avif" media="(prefers-color-scheme: dark)" type="image/avif">
  <source srcset="https://images.lyuk98.com/eeab4f9d-a26e-4302-a6a6-719a6119aaa2.avif" type="image/avif">
  <source srcset="https://images.lyuk98.com/f75cf2db-3645-4f0a-8387-e5f671357719.webp" media="(prefers-color-scheme: dark)">
  <img src="https://images.lyuk98.com/eeab4f9d-a26e-4302-a6a6-719a6119aaa2.webp" alt="Expanded menu for a note on the Standard Notes client. The option &quot;Export&quot; is selected.">
</picture>

They were then copied to a directory for my blog posts, at [`content/en/blog`](https://codeberg.org/lyuk98/pages/src/commit/464b586b198cfc6e6b2497d9ec4c15211b9031bf/content/en/blog "pages/content/en/blog at 464b586b198cfc6e6b2497d9ec4c15211b9031bf - lyuk98/pages - Codeberg.org").

```console
[lyuk98@framework:~/pages]$ ls content/en/blog/
'Building a homelab (#1) - Creating a Kubernetes cluster with Talos Linux.md'
'Building a project (#1) - Learning about OpenStack.md'
'Building a project (#2) - Preparing a server.md'
'Building a project (#3) - Attempting to run Kubernetes and OpenStack.md'
'Building obscure packages with Nix.md'
'Getting started with Listed.md'
'Hosting Ente with Terraform, Vault, and NixOS.md'
"Improving Listed's code legibility in dark theme.md"
'Self-hosting PeerTube with Tailscale.md'
'Switching from Arch Linux to NixOS.md'
'Using GPT fdisk to recover corrupted partition table.md'
'Using Zrythm for audio production.md'
'Using custom domain with Listed.md'
```

Their [front matter](https://gohugo.io/content-management/front-matter/ "Front matter") was subsequently edited to match Hugo's specification. `date` properties were set to their publication dates, `title` to their titles (obviously), and `description` to what used to be `desc` properties. For some posts with `#` in their titles, I also manually added `slug`; it was due to problems with Codeberg Pages, where accessing affected posts throws a 404 error.

```markdown
---
date: '2025-11-05'
title: Building a project (#2) - Preparing a server
description: As an effort to host OpenStack, I prepared a device to run the service with.
slug: building-a-project-2-preparing-a-server
---
```

# Setting up Codeberg and Codeberg Pages

Looking at [Codeberg Pages](https://codeberg.page/ "Codeberg Pages - static pages for your projects"), there were three ways to deploy my website:

- legacy method
- new method without CI
- new method with CI

However, the "legacy method" was the only option I could choose, since others currently do not support custom domains.

To start, I first created a repository named `pages`.

<picture>
  <source srcset="https://images.lyuk98.com/600461c8-2eb3-45a8-b2f9-eee77a817e85.avif" media="(prefers-color-scheme: dark)" type="image/avif">
  <source srcset="https://images.lyuk98.com/8ea05d26-04f6-4cce-b899-edba4f6240e1.avif" type="image/avif">
  <source srcset="https://images.lyuk98.com/600461c8-2eb3-45a8-b2f9-eee77a817e85.webp" media="(prefers-color-scheme: dark)">
  <img src="https://images.lyuk98.com/8ea05d26-04f6-4cce-b899-edba4f6240e1.webp" alt="Repository creation page at Codeberg. Owner is set to &quot;lyuk98&quot;, and the repository name is &quot;pages&quot;.">
</picture>

I then headed to settings and enabled Forgejo Actions for the repository.

<picture>
  <source srcset="https://images.lyuk98.com/7c784f31-65a8-49ab-8d2a-e68d5ae7d82f.avif" media="(prefers-color-scheme: dark)" type="image/avif">
  <source srcset="https://images.lyuk98.com/622832ae-3f39-40b5-b65b-726b04b6ab14.avif" type="image/avif">
  <source srcset="https://images.lyuk98.com/7c784f31-65a8-49ab-8d2a-e68d5ae7d82f.webp" media="(prefers-color-scheme: dark)">
  <img src="https://images.lyuk98.com/622832ae-3f39-40b5-b65b-726b04b6ab14.webp" alt="Settings page at the pages repository. From the &quot;Overview&quot; section, the &quot;Actions&quot; option, described as &quot;Enable integrated CI/CD pipelines with Forgejo Actions&quot;, is checked together with &quot;Code&quot;.">
</picture>

## `.domains` and `_redirects`

I did not want to break existing links that were created by Listed. To keep them working even after the migration, I tried to find a way to redirect requests.

Hugo supports [aliases](https://gohugo.io/methods/page/aliases/ "Aliases"), redirecting requests on its own, but they recommended doing so at the server level for improved efficiency.

> By default, Hugo handles aliases by creating individual HTML files for each alias path. These files contain a `meta http-equiv="refresh"` tag to redirect the visitor via the browser.
>
> While functional, generating a single `_redirects` file allows your hosting provider to handle redirects at the server level. This is more efficient than client-side redirection and improves performance by eliminating the need to load a middle-man HTML page.

Codeberg Pages allow [setting up one](https://docs.codeberg.org/codeberg-pages/advanced-usage/#redirects "Advanced usage | Codeberg Documentation"), so I wrote [`_redirects`](https://codeberg.org/lyuk98/pages/src/commit/464b586b198cfc6e6b2497d9ec4c15211b9031bf/_redirects "pages/_redirects at 464b586b198cfc6e6b2497d9ec4c15211b9031bf - lyuk98/pages - Codeberg.org") containing redirect rules.

```
# Redirect from previously used URLs in the Listed blogging platform
/60208/getting-started-with-listed /blog/2025/03/getting-started-with-listed/ 301
/60219/building-a-project-1-learning-about-openstack /blog/2025/03/building-a-project-1-learning-about-openstack/ 301
/60610/self-hosting-peertube-with-tailscale /blog/2025/03/self-hosting-peertube-with-tailscale/ 301
/61071/using-gpt-fdisk-to-recover-corrupted-partition-table /blog/2025/04/using-gpt-fdisk-to-recover-corrupted-partition-table/ 301
/61351/using-custom-domain-with-listed /blog/2025/04/using-custom-domain-with-listed/ 301
/62055/switching-from-arch-linux-to-nixos /blog/2025/05/switching-from-arch-linux-to-nixos/ 301
/65671/hosting-ente-with-terraform-vault-and-nixos /blog/2025/09/hosting-ente-with-terraform-vault-and-nixos/ 301
/66674/building-obscure-packages-with-nix /blog/2025/10/building-obscure-packages-with-nix/ 301
/66893/building-a-project-2-preparing-a-server /blog/2025/11/building-a-project-2-preparing-a-server/ 301
/67660/building-a-project-3-attempting-to-run-kubernetes-and-openstack /blog/2025/12/building-a-project-3-attempting-to-run-kubernetes-and-openstack/ 301
/67775/improving-listed-s-code-legibility-in-dark-theme /blog/2025/12/improving-listeds-code-legibility-in-dark-theme/ 301
/68518/using-zrythm-for-audio-production /blog/2025/12/using-zrythm-for-audio-production/ 301
/70549/building-a-homelab-1-creating-a-kubernetes-cluster-with-talos-linux /blog/2026/02/building-a-homelab-1-creating-a-kubernetes-cluster-with-talos-linux/ 301
```

Next up was [`.domains`](https://codeberg.org/lyuk98/pages/src/commit/464b586b198cfc6e6b2497d9ec4c15211b9031bf/.domains "pages/.domains at 464b586b198cfc6e6b2497d9ec4c15211b9031bf - lyuk98/pages - Codeberg.org"), but this one was simple; I just had to add domains I want to link my pages to.

```
lyuk98.com
www.lyuk98.com
```

Any domain from the second line simply redirects to the first one. It was unfortunate that HTTP [`307 Temporary Redirect`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/307 "307 Temporary Redirect - HTTP | MDN") response was used instead of [`301 Moved Permanently`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/301 "301 Moved Permanently - HTTP | MDN"), but I could not find a way to change the status code for now.

```console
[lyuk98@framework:~]$ curl --head https://www.lyuk98.com/
HTTP/2 307 
allow: GET, HEAD, OPTIONS
cache-control: public, max-age=600
content-type: text/html; charset=utf-8
location: https://lyuk98.com/
referrer-policy: strict-origin-when-cross-origin
server: pages-server
date: Wed, 18 Mar 2026 08:54:35 GMT
```

## Writing the workflow

I started from [an example workflow](https://neohugo.github.io/host-and-deploy/host-on-codeberg-pages/#automated-deployment-using-forgejo-actions "Host on Codeberg Pages") from a page I came across. However, I had to make some adjustments.

The guide uses an image named `hugomods/hugo:exts-0.147.9`. It seemed pretty dated, so I went to [its Docker Hub page](https://hub.docker.com/r/hugomods/hugo "hugomods/hugo - Docker Image") to check for newer versions. However, newest `exts` images were still not recent enough, at v0.154.5 (possibly due to [build errors](https://github.com/hugomods/docker/issues/126#issuecomment-3922750393 "ci-0.155.x images missing · Issue #126 · hugomods/docker")).

Fortunately, after a while of searching, I found an [official GitHub Package](https://github.com/gohugoio/hugo/pkgs/container/hugo "Package hugo"), which was set as a container image for the Codeberg runner.

```yaml
jobs:
  build:
    # Use lazy Codeberg runner for lightweight tasks
    runs-on: codeberg-tiny-lazy
    container:
      # Pin version of Hugo used for building
      image: ghcr.io/gohugoio/hugo:v0.158.0
```

The workflow would first upload generated static files, which would then be downloaded by the `deploy` job. A problem with this, however, was that I needed a few more files to be present.

I added another step (after running `hugo`) to copy the necessary files. When it was done, I included an option `include-hidden-files` to make sure `.domains` is included in the artifact.

```yaml
      - name: Generate static files with Hugo
        env:
          # For maximum backward compatibility with Hugo modules
          HUGO_ENVIRONMENT: production
          HUGO_ENV: production
        run: |
          hugo \
            --gc \
            --minify
      - name: Copy files for Codeberg Pages
        run: |
          cp _redirects public/_redirects
          cp .domains public/.domains
          cp LICENSE public/LICENSE
      - name: Upload generated files
        uses: https://code.forgejo.org/actions/upload-artifact@v3
        with:
          name: Generated files
          path: public/
          include-hidden-files: true
```

The rest was pretty much the same. The `deploy` job would download the artifact and push to the `pages` branch with newly generated static contents.

The remote was added to the Git directory afterwards.

```console
[lyuk98@framework:~/pages]$ git remote add origin ssh://git@codeberg.org/lyuk98/pages.git
```

Before running `git push`, though, I had a few more things to do.

# Deploying the service

It was finally time to ~~say goodbye to Listed~~ deploy the website. Up until now, I could have backed out with no consequences; however, there was no going back after taking the following steps.

After going to my Listed blog's settings, I disabled the custom domain integration. I knew this was going to happen, but it was nevertheless hard to press that button.

<picture>
  <source srcset="https://images.lyuk98.com/6a82daf5-fbb7-4f7d-9e63-70871eb502ff.avif" type="image/avif">
  <img src="https://images.lyuk98.com/6a82daf5-fbb7-4f7d-9e63-70871eb502ff.webp" alt="Settings page for the Listed blog, where an existing custom domain integration, that says &quot;Linked to: https://lyuk98.com&quot;, is visible. A menu is expanded from there, where an option to &quot;Delete&quot; the integration can be seen.">
</picture>

When it was done, I went to my DNS provider, Cloudflare. From there, I added [the following records](https://docs.codeberg.org/codeberg-pages/using-custom-domain/#option-3%3A-a%2Faaaa-record "Using custom domains | Codeberg Documentation") to my domain:

| Type | Name | Data | Proxy status |
| --- | --- | --- | --- |
| `A` | `@` | `217.197.84.141` | DNS only |
| `AAAA` | `@` | `2a0a:4580:103f:c0de::2` | DNS only |
| `TXT` | `@` | `pages.pages.lyuk98.codeberg.page` | DNS only |
| `A` | `www` | `217.197.84.141` | DNS only |
| `AAAA` | `www` | `2a0a:4580:103f:c0de::2` | DNS only |
| `TXT` | `www` | `pages.pages.lyuk98.codeberg.page` | DNS only |

I messed up in this step, in two ways:

- I initially set the records to be "Proxied", which likely prevented proper issuance of a TLS certificate; after changing it to "DNS only" and waiting for more than 6 hours, the issue was eventually solved without any further action.
- `TXT` records were initially set to just `lyuk98.codeberg.page`; expected value is something like `[[branch.]repo.]user.codeberg.page`, but I did not know omitting `branch` causes a fallback to the repository's default branch, which is `main`.

Changes made to the local directory were then finally pushed to Codeberg, where the workflow pushed the generated website to a separate `pages` branch.

```console
[lyuk98@framework:~/pages]$ git push --set-upstream origin main
```

# Mirroring the repository

I polished a few rough edges after the successful first deployment. With the new blog now operational, I decided to mirror my repository next.

There was [a guide](https://forgejo.org/docs/latest/user/repo-mirror/ "Repository Mirrors | Forgejo – Beyond coding. We forge.") I could follow to reach my goal. Instead of creating personal access tokens, though, I chose to use SSH authentication.

For start, I created a repository at GitHub, with the same name as the one at Codeberg.

<picture>
  <source srcset="https://images.lyuk98.com/6de51736-dfd9-4abd-95f2-a5579fdb1b01.avif" media="(prefers-color-scheme: dark)" type="image/avif">
  <source srcset="https://images.lyuk98.com/f1225f63-7f21-445d-b843-f53a065c570e.avif" type="image/avif">
  <source srcset="https://images.lyuk98.com/6de51736-dfd9-4abd-95f2-a5579fdb1b01.webp" media="(prefers-color-scheme: dark)">
  <img src="https://images.lyuk98.com/f1225f63-7f21-445d-b843-f53a065c570e.webp" alt="Repository creation page at GitHub. Owner is set to &quot;lyuk98&quot;, and the repository name is &quot;pages&quot;.">
</picture>

When it was done, I went to the settings of my Codeberg repository, where I set up a push mirror.

<picture>
  <source srcset="https://images.lyuk98.com/b1bf110c-0618-43a3-82b1-b88a57dffd87.avif" media="(prefers-color-scheme: dark)" type="image/avif">
  <source srcset="https://images.lyuk98.com/ebe5e397-e3c6-4a7c-943d-916dd50dfa82.avif" type="image/avif">
  <source srcset="https://images.lyuk98.com/b1bf110c-0618-43a3-82b1-b88a57dffd87.webp" media="(prefers-color-scheme: dark)">
  <img src="https://images.lyuk98.com/ebe5e397-e3c6-4a7c-943d-916dd50dfa82.webp" alt="Settings at Codeberg repository for setting up repository mirroring. A form for creating a pushed repository is partially filled up, where &quot;Git remote repository URL&quot; is set to &quot;git@github.com:lyuk98/pages.git&quot;, &quot;Use SSH authentication&quot; and &quot;Sync when commits are pushed&quot; are checked, and &quot;Mirror interval&quot; is set to &quot;0&quot; (disabling periodic sync).">
</picture>

Doing so generated a public SSH key, which I added as a [deploy key](https://docs.github.com/authentication/connecting-to-github-with-ssh/managing-deploy-keys#deploy-keys "Managing deploy keys - GitHub Docs") in my GitHub repository.

<picture>
  <source srcset="https://images.lyuk98.com/48dafed4-9ddd-434e-876b-78a87abcbcb4.avif" media="(prefers-color-scheme: dark)" type="image/avif">
  <source srcset="https://images.lyuk98.com/64835cf8-52ff-4e15-b25b-b5f7ee79cf0b.avif" type="image/avif">
  <source srcset="https://images.lyuk98.com/48dafed4-9ddd-434e-876b-78a87abcbcb4.webp" media="(prefers-color-scheme: dark)">
  <img src="https://images.lyuk98.com/64835cf8-52ff-4e15-b25b-b5f7ee79cf0b.webp" alt="A dialog for adding a new deploy key at the GitHub repository. &quot;Title&quot; is set to &quot;Codeberg&quot;, &quot;Key&quot; is set to the public key generated by Codeberg, and &quot;Allow write access&quot; is checked.">
</picture>

# Measuring the benefits

## Setting up a Jekyll site

[Earlier in this post](#static-site-generator), I mentioned a possible performance improvement by using Hugo over Jekyll. I decided to see if the difference is noticeable, even if I do not have a lot of documents for the generator to build.

Running a Ruby gem ([like Jekyll](https://jekyllrb.com/docs/ruby-101/ "Ruby 101 | Jekyll • Simple, blog-aware, static sites")) in NixOS turned out to be a more involved process than I have previously imagined. However, the [Nixpkgs reference manual](https://nixos.org/manual/nixpkgs/stable/#developing-with-ruby "Nixpkgs Reference Manual") helped me throughout the process.

First, a new site was created.

```console
[lyuk98@framework:~]$ nix run nixpkgs#jekyll -- new myblog

[lyuk98@framework:~]$ cd myblog/
```

`Gemfile.lock` and `gemset.nix` were to be generated next. Before that, though, I commented out a line from `Gemfile` that contains `http_parser.rb`, as [an error occurs](https://github.com/nix-community/bundix/issues/105#issuecomment-1810677126 "attribute '\"http_parser.rb\"' missing · Issue #105 · nix-community/bundix") otherwise.

```console
[lyuk98@framework:~/myblog]$ sed \
  --in-place \
  --expression 's/gem "http_parser.rb"/# gem "http_parser.rb"/g' \
  Gemfile

[lyuk98@framework:~/myblog]$ nix run nixpkgs#bundler -- lock

[lyuk98@framework:~/myblog]$ nix run nixpkgs#bundix
```

A shell environment for running Jekyll was then defined. With `nix-shell`, I was now ready to build my site.

```console
[lyuk98@framework:~/myblog]$ cat << EOF > shell.nix
with (import <nixpkgs> {}); let
  gems = bundlerEnv {
    name = "myblog";
    gemdir = ./.;
  };
in
mkShell {
  packages = [
    gems
    gems.wrappedRuby
  ];
}
EOF

[lyuk98@framework:~/myblog]$ nix-shell
```

To generate a website from my blog posts, they were first copied from the Hugo project. A Markdown document generated upon the site's creation was then removed.

```console
[nix-shell:~/myblog]$ cp ~/pages/content/en/blog/* ~/myblog/_posts/

[nix-shell:~/myblog]$ cp ~/pages/content/en/about-me.md ~/myblog/_posts/

[nix-shell:~/myblog]$ rm _posts/2026-03-18-welcome-to-jekyll.markdown
```

Date was added to the start of each document's filename, since posts oddly did not appear without one. Furthermore, `layout: post` was added to each post's front matter.

```console
[nix-shell:~/myblog]$ cd _posts/

[nix-shell:~/myblog/_posts]$ for md in *.md; do mv "$md" "2026-03-18-$md"; done

[nix-shell:~/myblog/_posts]$ cd -

[nix-shell:~/myblog]$ for md in _posts/*.md; do \
  sed \
    --in-place \
    --expression "s/title:/layout: post\ntitle:/g" \
    "$md"; \
done
```

It was now time to build the whole thing. I ran `jekyll build` to do so.

```console
[nix-shell:~/myblog]$ jekyll build --profile
```

When it was done, I was somewhat surprised by the result.

```console
Build Process Summary: 

| PHASE    |   TIME |
+----------+--------+
| RESET    | 0.0000 |
| READ     | 0.0113 |
| GENERATE | 0.0003 |
| RENDER   | 0.5104 |
| CLEANUP  | 0.0007 |
| WRITE    | 0.0017 |
 

Site Render Stats: 

| Filename                                                           | Count |    Bytes |  Time |
+--------------------------------------------------------------------+-------+----------+-------+
| minima-2.5.2/_layouts/default.html                                 |    17 | 1256.60K | 0.033 |
| minima-2.5.2/_includes/head.html                                   |    17 |   34.47K | 0.020 |
| minima-2.5.2/_layouts/post.html                                    |    14 | 1178.08K | 0.016 |
| feed.xml                                                           |     1 | 1114.81K | 0.013 |
| minima-2.5.2/_includes/footer.html                                 |    17 |   19.77K | 0.006 |
| minima-2.5.2/_includes/header.html                                 |    17 |   16.82K | 0.005 |
| minima-2.5.2/_includes/social.html                                 |    17 |    6.64K | 0.004 |
| _posts/2026-03-18-Hosting Ente with Terraform, Vault, and NixOS.md |     1 |  113.84K | 0.002 |
| minima-2.5.2/_layouts/home.html                                    |     1 |    3.53K | 0.001 |
| minima-2.5.2/_layouts/page.html                                    |     2 |    1.18K | 0.000 |
 
                    done in 0.529 seconds.
 Auto-regeneration: disabled. Use --watch to enable.
```

It was pretty close to how much Hugo took to generate my site. To be sure, however, I ran `hugo build` again to see how long the Go-based site generator would take.

```console
[lyuk98@framework:~/pages]$ ../hugo build --templateMetrics
Start building sites … 
hugo v0.158.0-f41be7959a44108641f1e081adf5c4be7fc1bb63 linux/amd64 BuildDate=2026-03-16T17:42:04Z VendorInfo=gohugoio


Template Metrics:

       cumulative       average       maximum         
         duration      duration      duration  count  template
       ----------      --------      --------  -----  --------
     752.364974ms   50.157664ms   97.795613ms     15  single.html
      746.99613ms   32.478092ms   93.366651ms     23  _partials/head.html
     414.504195ms   18.021921ms   87.866032ms     23  _partials/templates/schema_json.html
     286.526687ms   71.631671ms  143.170913ms      4  rss.xml
     191.161551ms   47.790387ms   94.426211ms      4  list.html
     160.054027ms  160.054027ms  160.054027ms      1  index.json
      55.725994ms      24.334µs     753.117µs   2290  _markup/render-link.html
        30.4545ms     708.244µs    5.271358ms     43  _partials/post_meta.html
      24.771099ms    1.651406ms    4.521961ms     15  _partials/toc.html
      21.563459ms   21.563459ms   21.563459ms      1  search.html
      17.617011ms     765.957µs    1.980081ms     23  _partials/templates/opengraph.html
      11.829105ms    5.914552ms     8.17769ms      2  _default/terms.html
      11.066758ms     481.163µs    2.162197ms     23  _partials/header.html
       9.148543ms      38.278µs     631.799µs    239  _markup/render-image.html
       8.640652ms    8.640652ms    8.640652ms      1  404.html
       8.296751ms     360.728µs     1.09994ms     23  _partials/_funcs/get-page-images.html
       6.729657ms     292.593µs    1.191821ms     23  _partials/templates/twitter_cards.html
       6.696737ms     446.449µs     883.798µs     15  _partials/anchored_headings.html
       5.614319ms    5.614319ms    5.614319ms      1  _partials/home_info.html
       5.392968ms    5.392968ms    5.392968ms      1  _partials/social_icons.html
       4.933541ms     822.256µs    3.426743ms      6  _partials/svg.html
       4.843961ms     124.204µs     569.818µs     39  _partials/templates/_funcs/get-page-images.html
       2.376216ms     158.414µs     474.253µs     15  _partials/post_nav_links.html
       2.031413ms     119.494µs     280.406µs     17  _partials/breadcrumbs.html
       1.666743ms      25.253µs     144.466µs     66  _partials/author.html
       1.587784ms      69.034µs     603.984µs     23  _partials/footer.html
       1.120081ms     112.008µs     780.924µs     10  _markup/render-table.html.html
        914.008µs      39.739µs     855.377µs     23  _partials/extend_head.html
        510.797µs     510.797µs     510.797µs      1  sitemap.xml
        494.178µs      21.486µs     107.575µs     23  _partials/google_analytics.html
        330.503µs      22.033µs     194.021µs     15  _partials/edit_post.html
        296.279µs      29.627µs      180.17µs     10  _markup/render-table.rss.xml
        245.244µs      24.524µs     150.219µs     10  _markup/render-table.json.json
        176.977µs       4.115µs      20.577µs     43  _partials/cover.html
         98.243µs       6.549µs      38.413µs     15  _partials/post_canonical.html
         74.928µs       4.683µs      13.435µs     16  _partials/translation_list.html
         66.122µs      33.061µs      46.554µs      2  alias.html
         11.791µs         786ns       1.905µs     15  _partials/comments.html
          5.445µs         907ns       1.618µs      6  _partials/extend_footer.html


                  │ EN 
──────────────────┼────
 Pages            │ 27 
 Paginator pages  │  2 
 Non-page files   │  0 
 Static files     │  1 
 Processed images │  0 
 Aliases          │  2 
 Cleaned          │  0 

Total in 505 ms
```

As expected, the difference was not something that I would notice. After running both processes a few more times, Jekyll could be seen taking 500 to 520 milliseconds during each build, while Hugo took 480 to 500 milliseconds.

However, given that...

- I have configured a custom theme on my Hugo project, and
- the Jekyll site was set up just enough to run properly...

I felt it was a somewhat unfair way to compare their performance. The Ruby-based site generator was still technically slower (by about 4%), though, and I think I made a right choice overall.

## Reducing costs

Having [a Productivity or Professional plan](https://standardnotes.com/plans "Plans | Standard Notes") at Standard Notes was required to enable custom domain integration with Listed (and it is not possible at all now). The Productivity plan, which is what I am currently using, costs $63 per year; since the custom domain integration was pretty much the only thing that kept me from unsubscribing, I could reduce that yearly cost to zero by relying on generous cloud service providers. As a result, I think the amount that I have saved can now be spent on other priorities in my life at the moment.

~~But Backblaze just [increased their monthly storage price](https://www.backblaze.com/blog/backblaze-pricing-and-product-updates/ "Backblaze Pricing and Product Updates")!~~

# Finishing up

The blog is now available at [https://lyuk98.com/](https://lyuk98.com/) (and [https://lyuk98.codeberg.page/](https://lyuk98.codeberg.page/), but it redirects to the former anyway). The code is in [my repository](https://codeberg.org/lyuk98/pages "lyuk98/pages: My personal website - Codeberg.org") at Codeberg, which is mirrored to [the GitHub one](https://github.com/lyuk98/pages "lyuk98/pages: My personal website - mirror of https://codeberg.org/lyuk98/pages").

[The Listed blog instance](https://listed.to/@lyuk98 "이영욱 —") is still live at the time of writing, but I am not sure if I will want to take it down at some point in the future. If it is no longer accessible, you know why.

For now, though, I am glad that the migration turned out well.
