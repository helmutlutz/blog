---
layout: post
title:  "Setting up my first Jekyll blog"
date:   2023-01-03 20:55:50 +0100
tags: infrastructure
---
# Setting up my first Jekyll blog
These are the steps that I followed:  
- Starting from scratch on Windows, I followed this guide for the Jekyll installation: [Jekyll installation guide][jekyll-installation]
- Next you need to setup github-pages with jekyll. Follow this guide: [Github pages guide][ghpages-setup]
- Adding content is described in this article: [Adding content][adding-content-to-ghpages]
  
# Issues
- Running the page locally did not quite work as in the guide above, you have to add webrick in between: 
    ```
    bundle install
    bundle add webrick
    bundle exec jekyll serve
    ```
  
- Another issue was that the css was not rendering properly online - locally it did. The solution was to add an url and baseurl to the _config.yml (actually this was described in the guide as optional step, just didn't read the guide careful enough):  
    ```
    baseurl: "/blog" # the subpath of your site, e.g. /blog
    url: "https://helmutlutz.github.io" # the base hostname & protocol for your site, e.g. http://example.com
    ```


[jekyll-installation]: https://jekyllrb.com/docs/installation/windows/
[ghpages-setup]: https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll#creating-your-site
[adding-content-to-ghpages]: https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/adding-content-to-your-github-pages-site-using-jekyll