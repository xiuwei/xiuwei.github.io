# Blog Power By Hugo

## Quick start

Learn to create a Hugo site in minutes.

### Add content 

Add a new page to your site.

```bash
hugo new content posts/my-first-post.md
```
Hugo created the file in the content/posts directory. Open the file with your editor.

```md
+++
title = 'My First Post'
date = 2024-01-14T07:07:07+01:00
draft = true
+++
## Introduction

This is **bold** text, and this is *emphasized* text.

Visit the [Hugo](https://gohugo.io) website!
```

Save the file, then start Hugo’s development server to view the site. You can run either of the following commands to include draft content.

```bash
hugo server --buildDrafts
hugo server -D
```

View your site at the URL displayed in your terminal. Keep the development server running as you continue to add and change content.

When satisfied with your new content, set the front matter draft parameter to false.
Add a new page to your site.

### Publish the site 

n this step you will publish your site, but you will not deploy it.

When you publish your site, Hugo creates the entire static site in the public directory in the root of your project. This includes the HTML files, and assets such as images, CSS files, and JavaScript files.

When you publish your site, you typically do not want to include draft, future, or expired content. The command is simple.

```bash
hugo
```

## Basic usage

Hugo’s command line interface (CLI) is fully featured but simple to use, even for those with limited experience working from the command line.

### Display available commands 

To see a list of the available commands and flags:

```bash
hugo help
```

To get help with a subcommand, use the --help flag. For example:

```bash
hugo server --help
```

### Draft, future, and expired content

Hugo allows you to set `draft`, `date`, `publishDate`, and `expiryDate` in the front matter of your content. By default, Hugo will not publish content when:

- The `draft` value is true
- The `date` is in the future
- The `publishDate` is in the future
- The `expiryDate` is in the past

You can override the default behavior when running hugo or hugo server with command line flags:

```bash
hugo --buildDrafts    # or -D
hugo --buildExpired   # or -E
hugo --buildFuture    # or -F
```