---
title: "{{ replace .File.ContentBaseName "-" " " | title }}"
date: {{ .Date }}
draft: true
tags: []
summary: ""
# math: true # uncomment to enable KaTeX math rendering
# cover:
#   image: "/images/covers/my-cover.png" # put the file in static/images/covers/
#   alt: "cover image description"
---

Write your post here. Everything above `draft: true` must be filled in;
remove `draft: true` (or set it to false) when the post is ready to publish.
