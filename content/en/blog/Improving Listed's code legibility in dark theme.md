---
tags:
  - Blog
  - Listed
  - Rouge
date: '2025-12-04'
title: Improving Listed's code legibility in dark theme
---

It has been some time since I started writing on [Listed](https://listed.to/ "Listed — Welcome to your new public journal."). Although the development around the blogging platform seems to be slow-paced (which I feel is an understatement) and some note-taking services like [Anytype](https://anytype.io/ "The Everything App") also offer [publication of notes](https://doc.anytype.io/anytype-docs/getting-started/web-publishing "Publish | Anytype Docs"), I have yet to find a compelling reason nor devote my time for migration.

That said, there is one thing I do want to see an improvement to.

# The problem

Listed supports dark theme, which is not something I see in every blogging platforms I come across. However, I include a lot of code snippets, and switching to dark theme meant significant parts of the post become less legible.

| Light theme | Dark theme |
| --- | --- |
| ![A code snippet in light theme](https://images.lyuk98.com/8c3777af-8d67-4418-8716-5fc3d867f808.avif) | ![A code snippet in dark theme. Most notably, the equal sign in dark colour is less visible because its color did not change between themes.](https://images.lyuk98.com/9bd75d64-545e-4fa2-ab43-31f31df89e6a.avif) |

---

# Finding the code-highlighting library

I have [tried reading the code](https://lyuk98.com/61351/using-custom-domain-with-listed "Using custom domain with Listed") of the blogging platform before, so I thought I might as well do the same thing again. With `git clone`, [the repository](https://github.com/standardnotes/listed "standardnotes/listed: Create an online publication with automatic email newsletters. https://listed.to") was made ready.

```
[lyuk98@framework:~]$ git clone https://github.com/standardnotes/listed.git
[lyuk98@framework:~]$ cd listed/
[lyuk98@framework:~/listed]$ git switch --detach d7e82ea3725148d1dbc5960aa147b1aa4da8a215
[lyuk98@framework:~/listed]$ code .
```

My first theory was that Listed depends on a code-highlighting library. To find if there is any, I first searched for anything about `highlight`, which led me to `yarn.lock` that [contains dependencies of interest](https://github.com/standardnotes/listed/blob/d7e82ea3725148d1dbc5960aa147b1aa4da8a215/yarn.lock#L21-L33 "listed/yarn.lock at d7e82ea3725148d1dbc5960aa147b1aa4da8a215 · standardnotes/listed").

```
"@babel/code-frame@^7.0.0", "@babel/code-frame@^7.10.3":
  version "7.10.3"
  resolved "https://registry.yarnpkg.com/@babel/code-frame/-/code-frame-7.10.3.tgz#324bcfd8d35cd3d47dae18cde63d752086435e9a"
  integrity sha512-fDx9eNW0qz0WkUeqL6tXEXzVlPh6Y5aCDEZesl0xBGA8ndRukX91Uk44ZqnkECp01NAZUdCAl+aiQNGi0k88Eg==
  dependencies:
    "@babel/highlight" "^7.10.3"

"@babel/code-frame@^7.10.4":
  version "7.10.4"
  resolved "https://registry.yarnpkg.com/@babel/code-frame/-/code-frame-7.10.4.tgz#168da1a36e90da68ae8d49c0f1b48c7c6249213a"
  integrity sha512-vG6SvB6oYEhvgisZNFRmRCUkLz11c7rp+tbNTynGqc6mS1d5ATd/sGyV6W0KZZnXRKMTzZDRgQT3Ou9jhpAfUg==
  dependencies:
    "@babel/highlight" "^7.10.4"
```

It seemed like [@babel/code-frame](https://babeljs.io/docs/babel-code-frame "@babel/code-frame · Babel") is what makes the code colourful. However, just to be sure, I read the raw response of my blog post and located a code snippet.

```
[lyuk98@framework:~]$ curl https://lyuk98.com/66674/building-obscure-packages-with-nix | less
```

```html
<div class="highlight"><pre class="highlight nix"><code><span class="c"># Zen Browser</span>
<span class="nv">zen-browser</span> <span class="o">=</span> <span class="p">{</span>
  <span class="nv">url</span> <span class="o">=</span> <span class="s2">"github:0xc000022070/zen-browser-flake"</span><span class="p">;</span>
  <span class="nv">inputs</span> <span class="o">=</span> <span class="p">{</span>
    <span class="nv">nixpkgs</span><span class="o">.</span><span class="nv">follows</span> <span class="o">=</span> <span class="s2">"nixpkgs"</span><span class="p">;</span>
    <span class="nv">home-manager</span><span class="o">.</span><span class="nv">follows</span> <span class="o">=</span> <span class="s2">"home-manager"</span><span class="p">;</span>
  <span class="p">};</span>
<span class="p">};</span>
</code></pre></div>
```

Unlike my assumption, the response was already processed in some way, containing `class` values that were cryptic to the uninformed like me. Perhaps the highlighting is done by something else.

In the end, even if I find multiple possible fixes, the only one I could perform would be to [create custom CSS styles](https://standardnotes.com/help/66/how-do-i-change-the-colors-fonts-and-general-layout-of-my-listed-blog "How do I change the colors, fonts, and general layout of my Listed blog?"). As such, I aimed to find the part that themes the `<span>` elements.

The stylesheet containing the theme definition seemed to be generated on the fly, especially given that the code in question is over 2,000 lines.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/d148a120-e4cb-4dbd-9177-6c0e6e734b6a.avif">
  <img src="https://images.lyuk98.com/fd39e312-adb5-4973-be41-481a30bbfd78.avif" alt="Style Editor section of the Web Developer Tools. A stylesheet named &quot;application-75392...bae5d8c04dfa.css&quot; is selected, partially showing its content from lines 2364 to 2376." title="The stylesheet">
</picture>

I was unsure where it is created, but by pure luck, I was able to locate [the part in question](https://github.com/standardnotes/listed/blob/d7e82ea3725148d1dbc5960aa147b1aa4da8a215/app/assets/stylesheets/application.scss "listed/app/assets/stylesheets/application.scss at d7e82ea3725148d1dbc5960aa147b1aa4da8a215 · standardnotes/listed") within Listed's code.

```scss
/*
 * This is a manifest file that'll be compiled into application.css, which will include all the files
 * listed below.
 *
 * Any CSS and SCSS file within this directory, lib/assets/stylesheets, vendor/assets/stylesheets,
 * or any plugin's vendor/assets/stylesheets directory can be referenced here using a relative path.
 *
 * You're free to add application-wide styles to this file and they'll appear at the bottom of the
 * compiled file so the styles you add here take precedence over styles defined in any other CSS/SCSS
 * files in this directory. Styles in this file should be added after the last require_* statement.
 * It is generally better to create a new file per style scope.
 *
 *= require_self
 */

@import "stylekit";
@import "rouge";

#main-container {
  display: flex;
  min-height: 100%;
  flex-direction: column;
}

#content-container {
  height: 100%;
  flex: 1;
}
```

[StyleKit](https://github.com/standardnotes/StyleKit "standardnotes/StyleKit: CSS library for Standard Notes app and extensions"), which seemed to be an archived project by Standard Notes, did not contain something interesting. However, [Rouge](https://rouge.jneen.net/ "Rouge"), a "code highlighter", did.

# La logithèque Rouge

As I (hopefully correctly) found what does the code highlighting, I moved on to the next step. My guess was that since Listed depends on [an old version of the library](https://github.com/standardnotes/listed/blob/d7e82ea3725148d1dbc5960aa147b1aa4da8a215/Gemfile.lock#L247 "listed/Gemfile.lock at d7e82ea3725148d1dbc5960aa147b1aa4da8a215 · standardnotes/listed") (3.26.1 versus 4.6.1 at the time of writing), a newer version would be prepared for the dark theme. To find out, I made a local copy of [their code](https://github.com/rouge-ruby/rouge "rouge-ruby/rouge: A pure Ruby code highlighter that is compatible with Pygments") for reading.

```
[lyuk98@framework:~]$ git clone https://github.com/rouge-ruby/rouge.git
[lyuk98@framework:~]$ cd rouge/
[lyuk98@framework:~/rouge]$ git switch --detach 7a879833337f68fd358c350366db3f24cf441ed7
[lyuk98@framework:~/rouge]$ code .
```

Upon closer look, I found the cryptic `class` values in [the library's code](https://github.com/rouge-ruby/rouge/blob/7a879833337f68fd358c350366db3f24cf441ed7/lib/rouge/token.rb "rouge/lib/rouge/token.rb at 7a879833337f68fd358c350366db3f24cf441ed7 · rouge-ruby/rouge") as well as [its wiki](https://github.com/rouge-ruby/rouge/wiki/List-of-tokens "List of tokens · rouge-ruby/rouge Wiki").

> | *Token name* | *Token shortname* | *Description* |
> | --- | --- | --- |
> | Text | | Any type of text data |
> | Text.Whitespace | w | Specially highlighted whitespace |
> | Error | err | Lexer errors |
> | Other | x | Token for data not matched by a parser (e.g. HTML markup in PHP code) |
> | Keyword | k | Any keyword |
> | Keyword.Constant | kc | Keywords that are constants |
> | Keyword.Declaration | kd | Keywords used for variable declaration (e.g. var in javascript) |
> | Keyword.Namespace | kn | Keywords used for namespace declarations |
> | Keyword.Pseudo | kp | Keywords that aren't really keywords |
> | Keyword.Reserved | kr | Keywords which are reserved (such as end in Ruby) |
> | Keyword.Type | kt | Keywords wich refer to a type id (such as int in C) |
> | Name | n | Variable/function names |
> | Name.Attribute | na | Attributes (in HTML for instance) |
> | Name.Builtin | nb | Builtin names which are available in the global namespace |
> | Name.Builtin.Pseudo | bp | Builtin names that are implicit (such as self in Ruby) |
> | Name.Class | nc | For class declaration |
> | Name.Constant | no | For constants |
> | Name.Decorator | nd | For decorators in languages such as Python or Java |
> | Name.Entity | ni | Token for entitites such as &nbsp; in HTML |
> | Name.Exception | ne | Exceptions and errors (e.g. ArgumentError in Ruby) |
> | Name.Function | nf | Function names |
> | Name.Property | py | Token for properties |
> | Name.Label | nl | For label names |
> | Name.Namespace | nn | Token for namespaces |
> | Name.Other | nx | For other names |
> | Name.Tag | nt | Tag mainly for markup such as XML or HTML |
> | Name.Variable | nv | Token for variables |
> | Name.Variable.Class | vc | Token for class variables (e.g. @@var in Ruby) |
> | Name.Variable.Global | vg | For global variables (such as $LOAD_PATH in Ruby) |
> | Name.Variable.Instance | vi | Token for instance variables (such as @var in Ruby) |
> | Literal | l | Any literal (if not further defined) |
> | Literal.Date | ld | Date literals |
> | Literal.String | s | String literals |
> | Literal.String.Backtick | sb | String enclosed in backticks |
> | Literal.String.Char | sc | Token type for single characters |
> | Literal.String.Doc | sd | Documentation strings (such as in Python) |
> | Literal.String.Double | s2 | Double quoted strings |
> | Literal.String.Escape | se | Escaped sequences in strings |
> | Literal.String.Heredoc | sh | For "heredoc" strings (e.g. in Ruby) |
> | Literal.String.Interpol | si | For interpoled part in strings (e.g. in Ruby) |
> | Literal.String.Other | sx | Token type for any other strings (for example %q{foo} string constructs in Ruby) |
> | Literal.String.Regex | sr | Regular expressions literals |
> | Literal.String.Single | s1 | Single quoted strings |
> | Literal.String.Symbol | ss | Symbols (such as :foo in Ruby) |
> | Literal.Number | m | Any number literal (if not further defined) |
> | Literal.Number.Float | mf | Float numbers |
> | Literal.Number.Hex | mh | Hexadecimal numbers |
> | Literal.Number.Integer | mi | Integer literals |
> | Literal.Number.Integer.Long | il | Long interger literals |
> | Literal.Number.Oct | mo | Octal literals |
> | Literal.Number.Hex | mx | Hexadecimal literals |
> | Literal.Number.Bin | mb | Binary literals |
> | Operator | o | Operators (commonly +, -, /, *) |
> | Operator.Word | ow | Word operators (e.g. and) |
> | Punctuation | p | Punctuation which is not an operator |
> | Comment | c | Single ligne comments |
> | Comment.Multiline | cm | Mutliline comments |
> | Comment.Preproc | cp | Preprocessor comments such as <% %> in ERb |
> | Comment.Single | c1 | Comments that end at the end of the line |
> | Comment.Special | cs | Special data in comments such as @license in Javadoc |
> | Generic | g | Unstyled token |
> | Generic.Deleted | gd | Token value as deleted |
> | Generic.Emph | ge | Token value as emphasized |
> | Generic.Error | gr | Token value as an error message |
> | Generic.Heading | gh | Token value as a headline |
> | Generic.Inserted | gi | Token value as inserted |
> | Generic.Output | go | Marked as a program output |
> | Generic.Prompt | gp | Marked as a command prompt |
> | Generic.Strong | gs | Mark the token value as bold (for rst lexer) |
> | Generic.Subheading | gu | Marked as a subheadline |
> | Generic.Traceback | gt | Mark the token as a part of an error traceback |
> | Generic.Lineno | gl | Line numbers |

There were multiple themes with different colour values, making it slightly challenging to specify one to apply as a solution. Fortunately, I [found a clue](https://github.com/standardnotes/listed/blob/d7e82ea3725148d1dbc5960aa147b1aa4da8a215/app/assets/stylesheets/rouge.css.erb "listed/app/assets/stylesheets/rouge.css.erb at d7e82ea3725148d1dbc5960aa147b1aa4da8a215 · standardnotes/listed") that suggests [the GitHub theme](https://rouge-ruby.github.io/docs/Rouge/Themes/Github.html "Class: Rouge::Themes::Github — Documentation by YARD 0.9.37") is used by Listed.

```erb
<% require 'rouge' %>
<%= Rouge::Themes::Github.new.render %>
```

Even better, that specific theme gained support for dark theme with [an update](https://github.com/rouge-ruby/rouge/pull/1918 "Update GitHub theme, add dark mode by dunkmann00 · Pull Request #1918 · rouge-ruby/rouge"). What was now left to do was to write some CSS.

# Applying the theme

Closely following [the theme definition](https://github.com/rouge-ruby/rouge/blob/7a879833337f68fd358c350366db3f24cf441ed7/lib/rouge/themes/github.rb "rouge/lib/rouge/themes/github.rb at 7a879833337f68fd358c350366db3f24cf441ed7 · rouge-ruby/rouge"), I wrote a not-so-insignificant amount of CSS. Because [the aforementioned update](https://github.com/rouge-ruby/rouge/pull/1918 "Update GitHub theme, add dark mode by dunkmann00 · Pull Request #1918 · rouge-ruby/rouge") contains more than just adding new colours, I rewrote styles from scratch.

```css
/* Light theme */
:root {
	/* Primer primitives */
	--P_RED_0: #ffebe9;
	--P_RED_5: #cf222e;
	--P_RED_7: #82071e;
	--P_ORANGE_6: #953800;
	--P_GREEN_0: #dafbe1;
	--P_GREEN_6: #116329;
	--P_BLUE_6: #0550ae;
	--P_BLUE_8: #0a3069;
	--P_PURPLE_5: #8250df;
	--P_GRAY_0: #f6f8fa;
	--P_GRAY_5: #6e7781;
	--P_GRAY_9: #24292f;

	/* Palettes */
	--palette-comment: var(--P_GRAY_5);
	--palette-constant: var(--P_BLUE_6);
	--palette-entity: var(--P_PURPLE_5);
	--palette-heading: var(--P_BLUE_6);
	--palette-keyword: var(--P_RED_5);
	--palette-string: var(--P_BLUE_8);
	--palette-tag: var(--P_GREEN_6);
	--palette-variable: var(--P_ORANGE_6);
	--palette-fgDefault: var(--P_GRAY_9);
	--palette-bgDefault: var(--P_GRAY_0);
	--palette-fgInserted: var(--P_GREEN_6);
	--palette-bgInserted: var(--P_GREEN_0);
	--palette-fgDeleted: var(--P_RED_7);
	--palette-bgDeleted: var(--P_RED_0);
	--palette-fgError: var(--P_GRAY_0);
	--palette-bgError: var(--P_RED_7);
}

@media (prefers-color-scheme: dark) {
	/* Dark theme */
	:root {
		/* Primer primitives */
		--P_RED_0: #ffdcd7;
		--P_RED_3: #ff7b72;
		--P_RED_7: #8e1519;
		--P_RED_8: #67060c;
		--P_ORANGE_2: #ffa657;
		--P_GREEN_0: #aff5b4;
		--P_GREEN_1: #7ee787;
		--P_GREEN_8: #033a16;
		--P_BLUE_1: #a5d6ff;
		--P_BLUE_2: #79c0ff;
		--P_BLUE_5: #1f6feb;
		--P_PURPLE_2: #d2a8ff;
		--P_GRAY_0: #f0f6fc;
		--P_GRAY_1: #c9d1d9;
		--P_GRAY_3: #8b949e;
		--P_GRAY_8: #161b22;

		/* Palettes */
		--palette-comment: var(--P_GRAY_3);
		--palette-constant: var(--P_BLUE_2);
		--palette-entity: var(--P_PURPLE_2);
		--palette-heading: var(--P_BLUE_5);
		--palette-keyword: var(--P_RED_3);
		--palette-string: var(--P_BLUE_1);
		--palette-tag: var(--P_GREEN_1);
		--palette-variable: var(--P_ORANGE_2);
		--palette-fgDefault: var(--P_GRAY_1);
		--palette-bgDefault: var(--P_GRAY_8);
		--palette-fgInserted: var(--P_GREEN_0);
		--palette-bgInserted: var(--P_GREEN_8);
		--palette-fgDeleted: var(--P_RED_0);
		--palette-bgDeleted: var(--P_RED_8);
		--palette-fgError: var(--P_GRAY_0);
		--palette-bgError: var(--P_RED_7);
	}
}

/* Text */
.highlight,
.highlight .w {
	color: var(--palette-fgDefault);
	background-color: var(--palette-bgDefault);
}

/* Keyword */
.highlight .k,
.highlight .kd,
.highlight .kn,
.highlight .kp,
.highlight .kr,
.highlight .kt,
.highlight .kv {
	color: var(--palette-keyword);
}

/* Generic.Error */
.highlight .gr {
	color: var(--palette-fgError);
}

/* Generic.Deleted */
.highlight .gd {
	color: var(--palette-fgDeleted);
	background-color: var(--palette-bgDeleted);
}

.highlight .nb, /* Name.Builtin */
.highlight .nc, /* Name.Class */
.highlight .no, /* Name.Constant */
.highlight .nn /* Name.Namespace */ {
	color: var(--palette-variable);
}

.highlight .sr, /* Literal.String.Regex */
.highlight .na, /* Name.Attribute */
.highlight .nt /* Name.Tag */ {
	color: var(--palette-tag);
}

/* Generic.Inserted */
.highlight .gi {
	color: var(--palette-fgInserted);
	background-color: var(--palette-bgInserted);
}

/* Generic.EmphStrong */
.highlight .ges {
	font-style: italic;
	font-weight: bold;
}

.highlight .kc, /* Keyword.Constant */
.highlight .l, /* Literal */
.highlight .ld,
.highlight .m,
.highlight .mb,
.highlight .mf,
.highlight .mh,
.highlight .mi,
.highlight .il,
.highlight .mo,
.highlight .mx,
.highlight .sb, /* Literal.String.Backtick */
.highlight .bp, /* Name.Builtin.Pseudo */
.highlight .ne, /* Name.Exception */
.highlight .nl, /* Name.Label */
.highlight .py, /* Name.Property */
.highlight .nv, /* Name.Variable */
.highlight .vc,
.highlight .vg,
.highlight .vi,
.highlight .vm,
.highlight .o, /* Operator */
.highlight .ow {
	color: var(--palette-constant);
}

.highlight .gh, /* Generic.Heading */
.highlight .gu /* Generic.Subheading */ {
	color: var(--palette-heading);
	font-weight: bold;
}

/* Literal.String */
.highlight .s,
.highlight .sa,
.highlight .sc,
.highlight .dl,
.highlight .sd,
.highlight .s2,
.highlight .se,
.highlight .sh,
.highlight .sx,
.highlight .s1,
.highlight .ss {
	color: var(--palette-string);
}

.highlight .nd, /* Name.Decorator */
.highlight .nf, /* Name.Function */
.highlight .fm {
	color: var(--palette-entity);
}

/* Error */
.highlight .err {
	color: var(--palette-fgError);
	background-color: var(--palette-bgError);
}

.highlight .c, /* Comment */
.highlight .ch,
.highlight .cd,
.highlight .cm,
.highlight .cp,
.highlight .cpf,
.highlight .c1,
.highlight .cs,
.highlight .gl, /* Generic.Lineno */
.highlight .gt /* Generic.Traceback */ {
	color: var(--palette-comment);
}

.highlight .ni,/* Name.Entity */
.highlight .si /* Literal.String.Interpol */ {
	color: var(--palette-fgDefault);
}

/* Generic.Emph */
.highlight .ge {
	color: var(--palette-fgDefault);
	font-style: italic;
}

/* Generic.Strong */
.highlight .gs {
	color: var(--palette-fgDefault);
	font-weight: bold;
}

/* Missing tokens */
.highlight .esc,
.highlight .x,
.highlight .n,
.highlight .nx,
.highlight .p,
.highlight .pi,
.highlight .g,
.highlight .go,
.highlight .gp {
	color: var(--palette-fgDefault);
}
```

When [the change was applied](https://standardnotes.com/help/66/how-do-i-change-the-colors-fonts-and-general-layout-of-my-listed-blog "How do I change the colors, fonts, and general layout of my Listed blog?"), though, no token-specific colours were applied to any part of the code. Later on, I realised that some sort of sanitation process removed variable declarations from the stylesheet, rendering it pretty much useless.

```css
/* Light theme */
:root {
	/* Primer primitives */
	
	
	
	
	
	
	
	
	
	
	
	

	/* Palettes */
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
}
```

I was left with an unfavourable option, but I had no other choice; I replaced `var()` with the actual values.

```css
/* Text */
.highlight,
.highlight .w {
	color: #24292f;
	background-color: #f6f8fa;
}

/* Keyword */
.highlight .k,
.highlight .kd,
.highlight .kn,
.highlight .kp,
.highlight .kr,
.highlight .kt,
.highlight .kv {
	color: #cf222e;
}

/* Generic.Error */
.highlight .gr {
	color: #f6f8fa;
}

/* Generic.Deleted */
.highlight .gd {
	color: #82071e;
	background-color: #ffebe9;
}

.highlight .nb, /* Name.Builtin */
.highlight .nc, /* Name.Class */
.highlight .no, /* Name.Constant */
.highlight .nn /* Name.Namespace */ {
	color: #953800;
}

.highlight .sr, /* Literal.String.Regex */
.highlight .na, /* Name.Attribute */
.highlight .nt /* Name.Tag */ {
	color: #116329;
}

/* Generic.Inserted */
.highlight .gi {
	color: #116329;
	background-color: #dafbe1;
}

/* Generic.EmphStrong */
.highlight .ges {
	font-style: italic;
	font-weight: bold;
}

.highlight .kc, /* Keyword.Constant */
.highlight .l, /* Literal */
.highlight .ld,
.highlight .m,
.highlight .mb,
.highlight .mf,
.highlight .mh,
.highlight .mi,
.highlight .il,
.highlight .mo,
.highlight .mx,
.highlight .sb, /* Literal.String.Backtick */
.highlight .bp, /* Name.Builtin.Pseudo */
.highlight .ne, /* Name.Exception */
.highlight .nl, /* Name.Label */
.highlight .py, /* Name.Property */
.highlight .nv, /* Name.Variable */
.highlight .vc,
.highlight .vg,
.highlight .vi,
.highlight .vm,
.highlight .o, /* Operator */
.highlight .ow {
	color: #0550ae;
}

.highlight .gh, /* Generic.Heading */
.highlight .gu /* Generic.Subheading */ {
	color: #0550ae;
	font-weight: bold;
}

/* Literal.String */
.highlight .s,
.highlight .sa,
.highlight .sc,
.highlight .dl,
.highlight .sd,
.highlight .s2,
.highlight .se,
.highlight .sh,
.highlight .sx,
.highlight .s1,
.highlight .ss {
	color: #0a3069;
}

.highlight .nd, /* Name.Decorator */
.highlight .nf, /* Name.Function */
.highlight .fm {
	color: #8250df;
}

/* Error */
.highlight .err {
	color: #f6f8fa;
	background-color: #82071e;
}

.highlight .c, /* Comment */
.highlight .ch,
.highlight .cd,
.highlight .cm,
.highlight .cp,
.highlight .cpf,
.highlight .c1,
.highlight .cs,
.highlight .gl, /* Generic.Lineno */
.highlight .gt /* Generic.Traceback */ {
	color: #6e7781;
}

.highlight .ni,/* Name.Entity */
.highlight .si /* Literal.String.Interpol */ {
	color: #24292f;
}

/* Generic.Emph */
.highlight .ge {
	color: #24292f;
	font-style: italic;
}

/* Generic.Strong */
.highlight .gs {
	color: #24292f;
	font-weight: bold;
}

/* Missing tokens */
.highlight .esc,
.highlight .x,
.highlight .n,
.highlight .nx,
.highlight .p,
.highlight .pi,
.highlight .g,
.highlight .go,
.highlight .gp {
	color: #24292f;
}

@media (prefers-color-scheme: dark) {
	/* Text */
	.highlight,
	.highlight .w {
		color: #c9d1d9;
		background-color: #161b22;
	}

	/* Keyword */
	.highlight .k,
	.highlight .kd,
	.highlight .kn,
	.highlight .kp,
	.highlight .kr,
	.highlight .kt,
	.highlight .kv {
		color: #ff7b72;
	}

	/* Generic.Error */
	.highlight .gr {
		color: #f0f6fc;
	}

	/* Generic.Deleted */
	.highlight .gd {
		color: #ffdcd7;
		background-color: #67060c;
	}

	.highlight .nb, /* Name.Builtin */
	.highlight .nc, /* Name.Class */
	.highlight .no, /* Name.Constant */
	.highlight .nn /* Name.Namespace */ {
		color: #ffa657;
	}

	.highlight .sr, /* Literal.String.Regex */
	.highlight .na, /* Name.Attribute */
	.highlight .nt /* Name.Tag */ {
		color: #7ee787;
	}

	/* Generic.Inserted */
	.highlight .gi {
		color: #aff5b4;
		background-color: #033a16;
	}

	/* Generic.EmphStrong */
	.highlight .ges {
		font-style: italic;
		font-weight: bold;
	}

	.highlight .kc, /* Keyword.Constant */
	.highlight .l, /* Literal */
	.highlight .ld,
	.highlight .m,
	.highlight .mb,
	.highlight .mf,
	.highlight .mh,
	.highlight .mi,
	.highlight .il,
	.highlight .mo,
	.highlight .mx,
	.highlight .sb, /* Literal.String.Backtick */
	.highlight .bp, /* Name.Builtin.Pseudo */
	.highlight .ne, /* Name.Exception */
	.highlight .nl, /* Name.Label */
	.highlight .py, /* Name.Property */
	.highlight .nv, /* Name.Variable */
	.highlight .vc,
	.highlight .vg,
	.highlight .vi,
	.highlight .vm,
	.highlight .o, /* Operator */
	.highlight .ow {
		color: #79c0ff;
	}

	.highlight .gh, /* Generic.Heading */
	.highlight .gu /* Generic.Subheading */ {
		color: #1f6feb;
		font-weight: bold;
	}

	/* Literal.String */
	.highlight .s,
	.highlight .sa,
	.highlight .sc,
	.highlight .dl,
	.highlight .sd,
	.highlight .s2,
	.highlight .se,
	.highlight .sh,
	.highlight .sx,
	.highlight .s1,
	.highlight .ss {
		color: #a5d6ff;
	}

	.highlight .nd, /* Name.Decorator */
	.highlight .nf, /* Name.Function */
	.highlight .fm {
		color: #d2a8ff;
	}

	/* Error */
	.highlight .err {
		color: #f0f6fc;
		background-color: #8e1519;
	}

	.highlight .c, /* Comment */
	.highlight .ch,
	.highlight .cd,
	.highlight .cm,
	.highlight .cp,
	.highlight .cpf,
	.highlight .c1,
	.highlight .cs,
	.highlight .gl, /* Generic.Lineno */
	.highlight .gt /* Generic.Traceback */ {
		color: #8b949e;
	}

	.highlight .ni,/* Name.Entity */
	.highlight .si /* Literal.String.Interpol */ {
		color: #c9d1d9;
	}

	/* Generic.Emph */
	.highlight .ge {
		color: #c9d1d9;
		font-style: italic;
	}

	/* Generic.Strong */
	.highlight .gs {
		color: #c9d1d9;
		font-weight: bold;
	}

	/* Missing tokens */
	.highlight .esc,
	.highlight .x,
	.highlight .n,
	.highlight .nx,
	.highlight .p,
	.highlight .pi,
	.highlight .g,
	.highlight .go,
	.highlight .gp {
		color: #c9d1d9;
	}
}
```

# The result

The change was definitely noticeable. I could finally read code more comfortably.

| Light theme | Dark theme |
| --- | --- |
| ![A code snippet in light theme](https://images.lyuk98.com/8c7476c7-2a88-4536-801c-824ecf3fca43.avif) | ![A code snippet in dark theme](https://images.lyuk98.com/220a5dc4-6b87-40a1-92e0-d42d494c5792.avif) |

It was one less reason for switching the note-taking service. However, it was not a proper solution; the developers could implement a much more permanent one.

Anyway, I was glad that this problem is finally over with.
