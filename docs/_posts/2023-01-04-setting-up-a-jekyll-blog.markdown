---
layout: post
title:  "Setting up this Jekyll blog"
date:   2023-01-03 20:55:50 +0100
categories: SetupNotes
---
This is just a plain list of steps that I followed to get this site up and running.  
- Starting from scratch on Windows, I followed this guide for the Jekyll installation: [Jekyll installation guide][jekyll-installation]
- The tutorial in [Github pages guide][ghpages-setup] describe how to set up the repostiory and how to create a site with Jekyll
  
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

# Configuring a custom domain for this blog
- Buy a custom domain from one of the large DNS providers
- In your provider account, find the domain and go to Domain settings and `Manage DNS`
- Add 4 `A` records (i.e. edit the first that is already there and add 3 more):
    - The IP addresses which have to be entered, you find in this tutorial (step 5): [Managing a custom Domain for gh-pages][managing-custom-domain]
- Edit the address in the `CNAME` row (with Name "www") to point to your gh-pages website (helmutlutz.github.io). Exclude the repository's name (`/blog`)
- Revisit the guide [Managing a custom Domain for gh-pages][managing-custom-domain], and start with step 1. Essentially you need to go to the `Settings` and `Pages` site of the repository and enter your custom Domain. This will automatically create a CNAME file pointing to your domain.
- Enforce HTTPS encryption for your site by selecting "Enforce HTTPS"
- Now you can update your local repository by pulling the changes.
- Finally you have to edit URLs in the _config.yml file:
    ```
    enforce_ssl: https://www.scrycode.de
    baseurl: ""
    url: "https://www.scrycode.de"
    ```


[jekyll-installation]: https://jekyllrb.com/docs/installation/windows/
[ghpages-setup]: https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll#creating-your-site
[adding-content-to-ghpages]: https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/adding-content-to-your-github-pages-site-using-jekyll
[managing-custom-domain]: https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site