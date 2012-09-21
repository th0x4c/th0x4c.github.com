---
layout: post
title: "Octopress をセットアップ"
date: 2012-09-04 23:15
comments: true
categories: 
---
Jekyll + Octopress を使って Github Pages にブログを開設してみた。
[Octopress](http://octopress.org) の [Documentation のページ](http://octopress.org/docs/) に従えばよい。
ruby 1.9.3 が必要だが、今回は [JRuby](http://jruby.org) を使用した。

## 使用マシン

* MacBook Pro Retina ディスプレイモデル(Mid 2012)
* Mountain Lion (OS X 10.8.1)

## JRuby のインストール

~/local 配下に zip を解凍してインストール

    $ cd ~/local/
    $ curl -O http://jruby.org.s3.amazonaws.com/downloads/1.7.0.preview2/jruby-bin-1.7.0.preview2.zip
    $ unzip jruby-bin-1.7.0.preview2.zip
    $ cd ~/local/bin/
    $ ln -s ../jruby-1.7.0.preview2/bin/jruby .

## Octopress のセットアップ

[Octopress のセットアップ手順](http://octopress.org/docs/setup/) に従う。

    $ cd ~/Documents/
    $ git clone git://github.com/imathis/octopress.git octopress
    $ cd octopress/
    $ jruby -S gem install bundler
    expr: syntax error
    Fetching gem metadata from http://rubygems.org/.......
    Using rake (0.9.2.2)
    Using RedCloth (4.2.9)
    Installing posix-spawn (0.3.6) with native extensions
    Gem::Installer::ExtensionBuildError: ERROR: Failed to build gem native extension.

            /Users/hashi/local/bin/../jruby-1.7.0.preview2/bin/jruby extconf.rb
    NotImplementedError: C extension support is not enabled. Pass -Xcext.enabled=true to JRuby or set JRUBY_OPTS or modify .jrubyrc to enable.

       (root) at /Users/hashi/local/bin/../jruby-1.7.0.preview2/lib/ruby/shared/mkmf.rb:8
      require at org/jruby/RubyKernel.java:1024
       (root) at /Users/hashi/local/bin/../jruby-1.7.0.preview2/lib/ruby/shared/rubygems/custom_require.rb:1
       (root) at extconf.rb:1


    Gem files will remain installed in /Users/hashi/local/jruby-1.7.0.preview2/lib/ruby/gems/shared/gems/posix-spawn-0.3.6 for inspection.
    Results logged to /Users/hashi/local/jruby-1.7.0.preview2/lib/ruby/gems/shared/gems/posix-spawn-0.3.6/ext/gem_make.out
    An error occurred while installing posix-spawn (0.3.6), and Bundler cannot continue.
    Make sure that `gem install posix-spawn -v '0.3.6'` succeeds before bundling.

なんかエラーとなったので、エラーメッセージの指示通りに `JRUBY_OPTS` を設定したらうまくいった。

    $ export JRUBY_OPTS="-Xcext.enabled=true"
    $ jruby -S bundle install
    $ jruby -S rake install

追加でインストールされたコマンドが使用できるようにシンボリックリンクを作成。
(もちろん、JRuby を解凍したフォルダ配下の `bin` ディレクトリに `PATH` が通ってれば不要。)

    $ cd ~/local/bin
    $ ln -s ../jruby-1.7.0.preview2/bin/jekyll .
    $ ln -s ../jruby-1.7.0.preview2/bin/compass .
    $ ln -s ../jruby-1.7.0.preview2/bin/rackup .

## Github Pages の設定

引き続き [Octopress の Github Pages へのデプロイ手順](http://octopress.org/docs/deploying/github/) に従う。
まず、`username.github.com` というレポジトリを[新規で作成](https://github.com/repositories/new)する。
ここで、次の `rake setup_github_pages` の `Repository url:` で HTTPS での指定(`https://github.com/username/username.github.com.git`)ができなかったため、レポジトリに [SSH key でのアクセス設定](https://help.github.com/articles/generating-ssh-keys)した上で、`git@github.com:username/username.github.com.git` を指定する必要があった。

    $ cd ~/Documents/octopress
    $ jruby -S rake setup_github_pages
    expr: syntax error
    Enter the read/write url for your repository
    (For example, 'git@github.com:your_username/your_username.github.com)
    Repository url: git@github.com:th0x4c/th0x4c.github.com.git
    ...

これで次のことが行われるとのこと。

1. Ask you for your Github Pages repository url.
2. Rename the remote pointing to imathis/octopress from ‘origin’ to ‘octopress’
3. Add your Github Pages repository as the default origin remote.
4. Switch the active branch from master to source.
5. Configure your blog’s url according to your repository.
6. Setup a master branch in the _deploy directory for deployment.

最後に以下を実行すれば `_deploy/` ディレクトリにファイルが生成され、Github のレポジトリに push され、Github Pages に反映される。

    $ jruby -S rake generate
    $ jruby -S rake deploy

あとは、Octopress の以下のページを参照のこと。

* [Configure your blog](http://octopress.org/docs/configuring/)

とりあえず、`_config.yml` の `title:`, `author:` ぐらいを変更した。

## ブログ発行

[Start blogging with Octopress](http://octopress.org/docs/blogging/)を参照。

基本的に `rake new_post["Zombie Ninjas Attack: A survivor's retrospective"]` (`Zombie ...`はタイトル)を実行して、`source/_posts` 配下のファイルを編集すればよい。

`rake preview` で `http://localhost:4000` でプレビューを確認して、`rake gen_deploy` で Github Pages にデプロイ。

    $ jruby -S rake new_post["Setup Octopress"]
    $ emacs -nw source/_posts/2012-09-04-setup-octopress.markdown
    $ jruby -S rake preview
    $ jruby -S gen_deploy
