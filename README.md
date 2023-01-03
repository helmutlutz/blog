# myblog
Contains the articles that I write

# Basic Setup
- If you start from scratch, this is the guide I followed for the Jekyll installation on Windows: https://jekyllrb.com/docs/installation/windows/
- Next you need to setup github-pages with jekyll. Follow this guide: https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll#creating-your-site
- Adding content is described in this article: https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/adding-content-to-your-github-pages-site-using-jekyll

- Running the page locally did not quite work as in the guide above, you have to add webrick in between: 
    ```
    bundle install
    bundle add webrick
    bundle exec jekyll serve
    ```