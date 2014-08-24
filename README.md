wizacha.github.io
=================

Blog des développeurs de [Wizacha](https://wizacha.com).

Installation
=================
Il faut installer [Jekyll](http://jekyllrb.com/docs/installation/) (donc [Ruby](http://www.ruby-lang.org/en/downloads/) et [RubyGems](http://rubygems.org/pages/download)).
Puis se placer dans le répertoire des sources, éxecuter `jekyll serve --watch` et visiter l'url [http://0.0.0.0:4000](#).

Autre solution, installer [Docker](https://docs.docker.com/installation/#installation)
et éxecuter `sudo docker run --rm -v "$PWD:/src" -p 4000:4000 grahamc/jekyll serve --watch` depuis
le répertoire des sources.

**Note:** Pour visualiser en local les [brouillons de post](http://jekyllrb.com/docs/drafts/), il suffit de rajouter l'option `--drafts`.


