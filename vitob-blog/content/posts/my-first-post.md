+++
title = 'My First Post'
date = 2024-06-01T18:19:45-07:00
draft = false
+++


This is my first blog using Hugo generated contents backed by github.io.

It records how to add new blog article using markdown.

## Steps to add a blog article
1. Go to synced down [blog_markdown](https://github.com/jinchenglee/blog_markdown).
2. Create new markdown file.
```
cd blog_markdown/vitob-blog
hugo new content blog/xxx.md
```
3. Edit xxx.md
4. Preview by running '$> hugo server -D'

## Continue; Steps to publish
5. If satisfied with preview at http://localhost:1313/, changing 'draft = true' to 'draft = false' in xxx.md. Then run '$> hugo' to generate contents under public subdirectory.
6. Copy or move public/\* to synced down [github.io](git@github.com:jinchenglee/jinchenglee.github.io.git) repository and then push it to github.
