This is a Software Engineering blog by Michael Nguyen.

[https://vitawebsitedesign.github.io/blog/](https://vitawebsitedesign.github.io/blog/).

[![Jekyll Deploy](https://github.com/vitawebsitedesign/blog/actions/workflows/deploy-v3.yml/badge.svg)](https://github.com/vitawebsitedesign/blog/actions/workflows/deploy-v3.yml)

# Getting started
To build+serve locally:
```bash
bundle exec jekyll serve --livereload
```

# Deployment
1. Changes into the `master` branch triggers a GitHub action, that will update the gh-pages branch.
1. The automatic `gh-pages` branch change triggers a 2nd GitHub action to deploy to the configured GitHub pages URL.
