# charlibot.github.io pages

Built with Hugo.

Locally `hugo server -D` and visit http://localhost:1313/.

## Writing

Start a new page with `hugo new posts/my-first-post.md`.
When it's ready for prime-time, change `draft` to `false` in the metadata section at the top of the file.

## Themes

So much choice. Some decent ones I've found so far:

- [Noteworthy](https://themes.gohugo.io/hugo-theme-noteworthy/)
- [Vitae](https://themes.gohugo.io/hugo-vitae/)
- [Bear](https://themes.gohugo.io/hugo-bearblog/)
- [Ezhil](https://github.com/vividvilla/ezhil)

Settled on Noteworthy for now with some minor cosmetic changes:

- Changed colours to purples in themes/noteworthy/assets/css/main.scss:
  - `#63BDA2` -> `#5816BF`
  - `#359377` -> `#350F7F`
- Removed `Noto` fonts from `$serif` and `$sans-serif`
