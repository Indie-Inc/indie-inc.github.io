title: How to Setup Hexo
date: 2015-11-21 23:26:55
tags: ["hexo"]
author: tejitak
---

### Installation
Prerequisite: Node

``` bash
$ npm install -g hexo
```

### Setup Repository
``` bash
$ git clone https://github.com/Indie-Inc/indie-inc.github.io.git
$ cd Indie-Inc.github.io.git
$ npm install
```

### Write a Post

``` bash
$ hexo new “your xxxxx title"
```

Edit the post in source/_posts/your-xxxxx-title.md

More info: [Writing](http://hexo.io/docs/writing.html)

### Check on Local Hexo Server
``` bash
$ hexo server
```
Access to [http://localhost:4000/](http://localhost:4000/)

More info: [Server](http://hexo.io/docs/server.html)

### Deploy the Post
Deploy with generating static files
``` bash
$ hexo deploy -g
```
More info: [Generating](http://hexo.io/docs/generating.html), [Deployment](http://hexo.io/docs/deployment.html)

### Push the Posts to Repository
``` bash
$ git push origin hexo
```
