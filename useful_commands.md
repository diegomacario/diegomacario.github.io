```
jekyll new . --force

bundle exec jekyll serve --livereload

_config.yml
	title
	email
	description
	baseurl: ""
	url: diegomacario.github.io

Gemfile
	Comment "gem "jekyll"
	Uncomment "gem "github-pages""

bundle install

bundle update

bundle install

bundle update github-pages

open $(bundle info --path minima)

xdc-open $(bundle info --path minima)

minima -> layouts

Remove theme: minima from yaml if you gonna use custom
Remove gem "minima" in Gemfile
bundle update
```
