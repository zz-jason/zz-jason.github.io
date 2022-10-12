# zz-jason.github.io

https://zz-jason.github.io is my personal blog, it's a Hugo generated static site. This readme is a reminder to me for how to set-up environment and write blogs.

## Download the source and submodules

```sh
git clone https://github.com/zz-jason/zz-jason.github.io.git
cd zz-jason.github.io
git submodule update --init
```

## New post

```sh
cd zz-jason.github.io
docker run --rm -it -v $(pwd):/src -p 1313:1313 klakegg/hugo:0.80.0 new posts/your-post-name.md
```

## Preview changes

```sh
cd zz-jason.github.io
docker run --rm -it -v $(pwd):/src -p 1313:1313 klakegg/hugo:0.80.0 server
```

## Publish post

The [GitHub Action](https://github.com/zz-jason/zz-jason.github.io/blob/master/.github/workflows/gh-pages.yml) will automatically generate the static site, just push the changes to this repo and trigger the GitHub Action.

```
git commit -asm "new blog"
git push
```
