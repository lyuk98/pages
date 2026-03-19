---
tags:
  - Blog
  - Listed
  - Standard Notes
date: '2025-03-11'
title: 'Getting started with Listed'
---

Hello to whoever is reading the first post of this blog! Neither writing nor showing my work to public is within my comfort zone, but it is what I aim to overcome, starting with this post.

# My introduction to Standard Notes and Listed

I have been an everyday user of Proton's products, and I was introduced to Standard Notes when [they announced that they are "joining forces"](https://proton.me/blog/proton-standard-notes-join-forces). My needs for personal note-taking were already satisfied with basic products like Google Keep, but having used Notion for team projects in the past, I was convinced to try it out.

While getting used to the new service, I discovered Listed blogging platform. I have previously been advised to write my technical experiences to keep records, and it seemed like a good place to do so.

# Setting up the blog

Creating a blog was easy, as I simply had to navigate to preferences for Standard Notes and create a new author under "Listed" section.

![image](https://images.lyuk98.com/5e68a288-1de3-44c0-8df1-97c0e4a1815d.avif)

I then enabled custom CSS to use fonts other than what Listed uses by default. Following [the guide](https://standardnotes.com/help/66/how-do-i-change-the-colors-fonts-and-general-layout-of-my-listed-blog), I wrote and published a note with a little bit of CSS.

```css
---
metatype: css
---

/* Import the fonts */
@import url("https://rsms.me/inter/inter.css");
@import url("https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/static/pretendard-dynamic-subset.min.css");
@import url("https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/variable/pretendardvariable-dynamic-subset.min.css");

/* Declare the font family */
.h1,
.h2,
.h3,
.h4,
.h5,
body {
	font-family: Inter, 'Pretendard', sans-serif;
  font-feature-settings: 'liga' 1, 'calt' 1; /* fix for Chrome */
}
@supports (font-variation-settings: normal) {
  .h1,
  .h2,
  .h3,
  .h4,
  .h5,
  body { font-family: InterVariable, 'Pretendard Variable', sans-serif; }
}
```

I looked around to find an image-hosting service for embedding images in posts. After taking a look at recommendations [a guide](https://notes.colfer.net/44029/how-to-add-images-to-your-listed-to-blog-post) provided, I decided to use [Horizon](https://horizon.pics/).

# To do

## Using a custom domain

My domain did not link to a proper web page during years of its usage, so I wished to use it to access my blog. However, for reasons unknown to me, I kept getting emails that the integration was unsuccessful.

> Hi there, your Listed domain settings were not configured properly. Please make sure you have an A record pointing to **18.205.249.107**, then **submit your domain request again via your author settings**. You'll know you have it configured correctly if when you visit your custom domain, it shows an *Invalid Certificate error*.

With my personality, I was hesitant to ask for help, and the problem remains unsolved for almost about a year. I recently came across [a post on Reddit](https://www.reddit.com/r/StandardNotes/comments/1j46kwg/listedto_custom_domains/), where one user claimed disabling DNSSEC helped them; I then immediately visited my domain name provider to switch it off, but the problem still persisted.

# Wrapping up...

I am with hopes that I will stay committed to keep this blog updated for the foreseeable future. I am simply hoping for now, but time will tell...
