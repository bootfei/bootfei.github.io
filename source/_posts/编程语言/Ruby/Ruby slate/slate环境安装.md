---
title: slate环境安装
date: 2021-07-23 18:14:44
tags:
---



## Dependencies

Minimally, you will need the following:

- [Ruby](https://www.ruby-lang.org/en/) >= 2.5
- [Bundler](https://bundler.io/)
- [NodeJS](https://nodejs.org/en/)
- [Git](https://git-scm.com/)



First, install [homebrew](https://brew.sh/), then install xcode command line tools:

```
xcode-select --install
```

Agree to the Xcode license:

```
sudo xcodebuild -license
```

Install nodejs runtime:

```
brew install node
```

Update RubyGems and install bundler:

```
gem update --system
gem install bundler
```



ruby版本升级

```
brew upgrade ruby
```

brew的ruby和内置ruby冲突，使用brew的ruby

```
echo ''  >>> ~/.bash_profile
```

## Running slate

You can run slate in two ways, either as a server process for development, or just build html files.

To do the first option, run:

```
bundle exec middleman server
```

and you should see your docs at [http://localhost:4567](http://localhost:4567/). Whoa! That was fast!