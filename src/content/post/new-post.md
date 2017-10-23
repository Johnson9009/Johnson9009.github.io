---
title: "2017 10 23 New Post"
date: 2017-10-23T20:10:52+08:00
lastmod: 2017-10-23T20:10:52+08:00
tags: []
categories: ["C/C++"]
author: "Johnson Zhang"
---

# Instruction of Create a New Post
1. Enter the root directory of our site.
2. Create the post.
   
       Below command will generate a new post whose type is set to `post` automatically. So we can see it on Archives page, the directory name must be `post`. The title of this post will be set to `post name` automatically and the slug of this post will be `post-name`, maybe we need change the slug with front matter of this post. The slug should be a string which is a lower case version of title name and replace all white space with `-` and replace any special character with its corresponding English word. For example, the slug of post `test#test.md` should be `test-sharp-test`.
    
    ``` shell
    hugo new post/post-name.md
    ```
<!--more-->
