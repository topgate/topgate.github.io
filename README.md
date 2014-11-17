# 社員の皆様へ

[jekyll](http://jekyllrb.com/docs/quickstart/)というツールを使ってます。
GitHub側で自動的にビルドされてデプロイされます。そんな機能あったんやな。

## 準備

1. Ruby入れる
2. `gem install bundler` する
3. `bundle install` とか `bundle update` とかする
4. `bundle exec jekyll serve` する
5. たぶん [http://localhost:4000/](http://localhost:4000/) とかにアクセスする

## 原稿を書こう！

1. ブランチをmaster以外にする(トピックブランチを作る)
2. _posts/ 配下にmarkdownで原稿を書く
3. `bundle exec jekyll serve` する
4. たぶん [http://localhost:4000/](http://localhost:4000/) とかで動作確認する
5. pull request 出す
6. 誰かにレビューしてもらってmergeされたら10分くらいで反映されるはず
