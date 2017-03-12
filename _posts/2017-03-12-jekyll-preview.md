---
layout: post
title: 用 Jekyll 预览博客
tags: tool
category: tool
---



好久没写博客，Jekyll 也没了，无法预览：

```sh
$ gem install jekyll
ERROR:  Could not find a valid gem 'jekyll' (>= 0), here is why:
          Unable to download data from http://ruby.sdutlinux.org/ - server did not return a valid file (http://ruby.sdutlinux.org/latest_specs.4.8.gz)
```



反正 Ruby 也太旧了，重装看一下行不行：

```sh
# 安装 RVM (Ruby Version Manager)
$ \curl -L https://get.rvm.io | bash -s stable

# 根据提示操作 (Load RVM into a shell session *as a function*)
$ echo "source ~/.profile" >> /Users/xxx/.bash_profile

# 更新
$ rvm get stable --autolibs=enable
$ rvm reload                             # 也可以关闭终端后重新开

# 安装最新版 Ruby
$ rvm list known
$ rvm install ruby-2.4
$ ruby -v
ruby 2.4.0p0 (2016-12-24 revision 57164) [x86_64-darwin16]

# 更新 gem
$ gem -v
2.6.10

$ gem update --system
ERROR:  While executing gem ... (Gem::RemoteFetcher::FetchError)
    server did not return a valid file (http://ruby.sdutlinux.org/specs.4.8.gz)

$ gem install jekyll
ERROR:  Could not find a valid gem 'jekyll' (>= 0), here is why:
          Unable to download data from http://ruby.sdutlinux.org/ - server did not return a valid file (http://ruby.sdutlinux.org/specs.4.8.gz)
```



还是得翻：

```sh
$ export http_proxy=127.0.0.1:64000
$ export https_proxy=127.0.0.1:64000

$ ping ruby.sdutlinux.org
PING sdutlinux.org (69.39.236.56): 56 data bytes
64 bytes from 69.39.236.56: icmp_seq=0 ttl=43 time=262.186 ms

$ gem update --system
ERROR:  While executing gem ... (Gem::RemoteFetcher::FetchError)
    too many connection resets (http://ruby.sdutlinux.org/specs.4.8.gz)

$ unset export http_proxy
$ unset export https_proxy
```



速度太慢，换国内的源：

```sh
$ gem sources --remove http://ruby.sdutlinux.org/
$ gem sources -a https://ruby.taobao.org/
$ gem sources -l
*** CURRENT SOURCES ***
https://ruby.taobao.org/

$ gem update --system
Latest version currently installed. Aborting.

$ gem install jekyll
```



预览一下博客：

```sh
$ jekyll serve
Configuration file: /Users/xxx/xxx/ericzhong.github.io/_config.yml
       Deprecation: Auto-regeneration can no longer be set from your configuration file(s). Use the --[no-]watch/-w command-line option instead.
       Deprecation: The 'pygments' configuration option has been renamed to 'highlighter'. Please update your config file accordingly. The allowed values are 'rouge', 'pygments' or null.
Configuration file: /Users/zwj/working/ericzhong.github.io/_config.yml
       Deprecation: Auto-regeneration can no longer be set from your configuration file(s). Use the --[no-]watch/-w command-line option instead.
       Deprecation: The 'pygments' configuration option has been renamed to 'highlighter'. Please update your config file accordingly. The allowed values are 'rouge', 'pygments' or null.
             Theme: value of 'theme' in config should be String to use gem-based themes, but got Hash
            Source: /Users/xxx/xxx/ericzhong.github.io
       Destination: /Users/xxx/xxx/ericzhong.github.io/_site
 Incremental build: disabled. Enable with --incremental
      Generating... 
  Dependency Error: Yikes! It looks like you don't have rdiscount or one of its dependencies installed. In order to use Jekyll as currently configured, you'll need to install this gem. The full error message from Ruby is: 'cannot load such file -- rdiscount' If you run into trouble, you can find helpful resources at https://jekyllrb.com/help/! 
  Conversion error: Jekyll::Converters::Markdown encountered an error while converting '_posts/2013-09-04-install-openstack-from-source.md':
                    rdiscount
             ERROR: YOUR SITE COULD NOT BE BUILT:
                    ------------------------------------
                    rdiscount

$ gem install rdiscount
$ jekyll serve
```



处理一下警告信息 (`_config.yml`)，新版本语法变了：

```yaml
#auto: true
#pygments: true
highlighter: pygments
```



语法高亮工具 [Pygemnts](http://pygments.org/languages/) 不支持关键字 `Bash`，只能用 `Bash shell scripts`，为了在 Typora 编辑器上也能高亮，就用 `sh` 吧，这样两边就都支持了。
