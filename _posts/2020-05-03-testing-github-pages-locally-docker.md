---
title: "Testing a GitHub Pages site locally using Docker"
categories:
  - blog
---

I wanted to be able to test changes to my site before pushing them to the live site on GitHub. I'm sure there are better ways to do this, but I found a quick and easy way is to use a docker container.

After researching this, I chose Bret Fisher's [jekyll-serve](https://hub.docker.com/r/bretfisher/jekyll-serve) image as it worked out of the box with the [Minimal Mistakes](https://github.com/mmistakes/minimal-mistakes) theme I am using.

Just create a docker-compose.yml file in the directory of your GitHub Pages site:

```yaml
version: '2.4'

services:
  jekyll:
    image: bretfisher/jekyll-serve
    volumes:
      - .:/site
    ports:
      - '80:4000'
```

Then as long as you have docker installed, type `docker-compose up` and you will have your GitHub Pages site running locally. Simple!

Note: When you save or create a file in your site directory the Docker container will regenerate your site (apart from _config.yml). If you don't want this to happen, for example for changes to README.md, you can edit your _config.yml file and specify files to be excluded:

```yaml
exclude:
  - docker-compose.yml
  - README.md
  - site.code-workspace
```

Also, I use Visual Studio Code, which I usually have set to Auto Save my code. However, when editing my site, I have this turned off, otherwise, the docker container will regenerate the site after every small change.

Thanks for reading, I hope this has been useful to you.
