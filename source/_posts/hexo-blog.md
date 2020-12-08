---
title: Blog & Slides
layout: slides
date: 2020-12-03
tags:
  - blog
  - slides 
  - Hexo 
  - Jekyll 
  - GitHub Pages
  - revealjs 
slide:
  theme: moon
  autoPlayMedia: true
---

## Blog & Slides

### 方镇澎

2020/12/3

----

## Outline

* Static Website and Blog
  * Github Page
  * Static Site Generator: Jekyll, Hexo...
* Slides on Website
  * Revealjs

----

## Static Website and Blog

A static web page is a web page that is delivered to the user's web browser exactly as stored.

-- Wikipedia <!-- .element: style="text-align: right" -->

静态网站是指全部由HTML（标准通用标记语言的子集）代码格式页面组成的网站，所有的内容包含在网页文件中。

-- 百度百科 <!-- .element: style="text-align: right" -->

---

## [GitHub Pages](https://pages.github.com/)

Websites for you and your projects.

1. Write an *index.html* file to `https://github.com/username/username.github.io`

2. Read from `https://username.github.io`

---

## Static Site Generator

![Site Generators](site-generators.PNG)
<!-- .element: class="r-stretch" -->

Data Source: <https://jamstack.org/generators/>

---

## Play with Jekyll

1. Fork a Jekyll theme repository
2. Write blogs in `_post` folder
3. Push to `https://github.com/username/username.github.io`
4. Read from `https://username.github.io`

---

## My Jekyll website

![My Jekyll website](Jekyll-site.PNG)
<!-- .element: class="r-stretch" -->

---

## Jekyll is fine until

I want to present my slides on my blog website!
<!-- .element: class="fragment" data-fragment-index="1" -->

----

## Slides on Website

----

## PPT? Revealjs!

![Revealjs logo](https://hakim-static.s3.amazonaws.com/reveal-js/logo/v1/reveal-black-text.svg) <!-- .element: width="450" -->

1. Write presentations using Markdown
2. Anything you can do on the web, you can do in your presentation

---

## [revealjs.com](https://revealjs.com/)
<!-- .element: style="position: fixed;top: 0;left: 0;" -->
<!-- .slide: data-background-iframe="https://revealjs.com/" data-background-interactive -->

---

## Revealjs Feature

1. Vertical Slide
2. Overiew Mode
3. Fullscreen Mode
4. PDF Export `?print-pdf`
5. Search plugin

---

## Slides.com

the official reveal.js presentation editor.

<video data-autoplay src="https://static.slid.es/site/homepage/v3/homepage-editor-1440x900.mp4"></video>

----

## Play with Hexo

I need a Javascript static site generator to use reveal.js easily.

1. Jekyll: Ruby *NO* <!-- .element: style="color:red" -->
2. Hexo: Javascript *YES* <!-- .element: style="color:green" -->
  
---

## Install Hexo

```bash
npm install hexo-cli -g
hexo init blog
cd blog
npm install
```

---

## Install Hexo theme

```bash
npm install hexo-theme-melody
```

---

## Use Melody Theme

[Melody](https://github.com/Molunerfinn/hexo-theme-melody) has Revealjs embeded.

Edit `_config.yml`

```yml
theme: melody
```

---

## Copy theme file

```bash
mv ./node_modules/hexo-theme-melody/_config.yml _config.melody.yml
```

---

## Write blog

```bash
hexo new post <title>
```

Then write blog in source/_posts/*title*.md

---

## Run Hexo locally

```bash
hexo server
```

---

## Deploy to GitHub

```bash
hexo g
hexo d
```

Or use [Travis CI](https://hexo.io/zh-cn/docs/github-pages.htmlhttps://travis-ci.com/)

![2020-12-08T154612](2020-12-08T154612.png)
<!-- .element: class="r-stretch" -->

---

## My local editor VSCode

![2020-12-04T145007](2020-12-04T145007.png)
<!-- .element: class="r-stretch" -->

1. *vscode-hexo-utils* plugin: enhance Markdown preview.

2. *Paste Image* plugin: paste image by ctrl-c/ctrl-alt-v.

---

## My Hexo website
<!-- .element: style="position: fixed;top: 0;left: 0;" -->
<!-- .slide: data-background-iframe="https://fzp.github.io/" data-background-interactive -->

----

## Why Blog?

Sharing knowledge, gaining a sense of identity and *learning* <!-- .element: style="color:red" -->.

<br/>

Feynman Technique:
<!-- .element: style="text-align: left" -->

Learn by teaching someone else a topic in simple terms.

----

## [Blog Now!](fzp.github.io)