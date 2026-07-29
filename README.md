# gauravnavgire.github.io

Personal site and blog of **Gaurav Navgire**, published at
[gauravnavgire.github.io](https://gauravnavgire.github.io).

Built with [Jekyll](https://jekyllrb.com) and the
[Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) theme, and deployed
automatically to GitHub Pages via GitHub Actions on every push to `main`.

## Local development

Install dependencies:

```bash
bundle install
```

Serve the site locally with livereload:

```bash
bash tools/run.sh
```

The site will be available at `http://127.0.0.1:4000`.

## Build and test

Run a production build and validate the output with
[html-proofer](https://github.com/gjtorikian/html-proofer):

```bash
bash tools/test.sh
```

## License

This work is published under the [MIT License](LICENSE). The Chirpy theme
itself is also MIT-licensed — see the
[theme repository](https://github.com/cotes2020/jekyll-theme-chirpy) for
details.
