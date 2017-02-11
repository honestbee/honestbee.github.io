# honestbee tech blog

This blog is powered by [Jekyll](https://jekyllrb.com/) and [github pages](https://pages.github.com/).

### Download blog
```
git clone git@github.com:honestbee/honestbee.github.io.git
bundle install
```

### Add article

To add an article to the blog, please branch from master, add it in `_posts`, then issue a PR and self merge. Test the blog on your machine before merging:

```
bundle exec jekyll serve
# => Now browse to http://localhost:4000
```

### Authors Bio

To add your authors bio, Add your name to `_config.yml > authors` key. Please follow the format from existing keys. Restart the server to see the changes.

Full documentation for Jekyll [here](https://jekyllrb.com/docs/home/)
