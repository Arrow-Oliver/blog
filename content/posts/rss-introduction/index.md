---
title: "Hugo中Rss安装"
date: 2023-06-08T16:07:51+08:00
draft: false
categories: [_Misc]
tags: []
card: false
weight: 0
---

​		因为发现许多博主都有Rss这个功能，我想着跟个风也搞一个Rss订阅源。接下来记录一下安装的心路历程。

# 1

​		本想着找个其他主题模板中copy一个就能完成的事儿，没想到人家的Rss订阅源都是主题模板内置的，这一下我给我整不会了。 我只好在Google中搜索<u>“rss订阅源如何生成”</u>，发现搜出来的大多数都是使用工具来管理Rss源，也就是如何订阅他人的Rss源（Rss.xml）。或者说使用工具（例如RssHub）进行Rss订阅源的生成，但是在我搜索这个工具的时候发现人家是让先自己手动搭建一个网站，我想着这可不行啊，生成一个Rss订阅源这么麻烦吗？跟我想象的不一样啊。后来在我搜索的过程中发现我用的不是Hugo框架嘛，直接搜索<u>“hugo rss订阅源”</u>试一下。

# 2

​		没想到一搜直接就有了，发现有好多博主都有安装Rss订阅源的教程，并且Hugo官方文档也有具体的Rss生成教程，顿时感觉到会搜索跟不会搜索的差距。接下来由于Hugo官方文档为英文，理解起来略显麻烦，就跟随[博主的帖子](https://www.wangchucheng.com/zh/posts/hugo-rss-template/)进行了Rss源的生成。

# 3

​		`第一步`，先定位Rss源模板的位置放在hugo架构的哪个地方？根据官方文档的介绍可以放在以下位置，并且优先权由上到下。在这里我将模板放到了`layouts/rss.xml`

~~~文件名
[layouts/index.rss.xml
layouts/home.rss.xml
layouts/rss.xml
layouts/list.rss.xml
layouts/index.xml
layouts/home.xml
layouts/list.xml
layouts/_default/index.rss.xml
layouts/_default/home.rss.xml
layouts/_default/rss.xml
layouts/_default/list.rss.xml
layouts/_default/index.xml
layouts/_default/home.xml
layouts/_default/list.xml
layouts/_internal/_default/rss.xml]
~~~

​		`第二步`，生成xml文件后模板代码如何生成？我直接用了博主帖子中的模板代码，进行了copy。

~~~xml
{{- /* Generate RSS v2 with full page content. */ -}}
{{- /* Upstream Hugo bug - RSS dates can be in future: https://github.com/gohugoio/hugo/issues/3918 */ -}}
{{- $page_context := cond .IsHome site . -}}
{{- $pages := $page_context.RegularPages -}}
{{- $limit := site.Config.Services.RSS.Limit -}}
{{- if ge $limit 1 -}}
  {{- $pages = $pages | first $limit -}}
{{- end -}}
{{- printf "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\" ?>" | safeHTML }}
<rss version="2.0" xmlns:atom="http://www.w3.org/2005/Atom">
  <channel>
    <title>{{ if ne .Title site.Title }}{{ with .Title }}{{.}} | {{ end }}{{end}}{{ site.Title }}</title>
    <link>{{ .Permalink }}</link>
    {{- with .OutputFormats.Get "RSS" }}
      {{ printf "<atom:link href=%q rel=\"self\" type=%q />" .Permalink .MediaType | safeHTML }}
    {{ end -}}
    <description>{{ .Title | default site.Title }}</description>
    <generator>Source Themes Academic (https://sourcethemes.com/academic/)</generator>
    {{- with site.LanguageCode }}<language>{{.}}</language>{{end -}}
    {{- with site.Copyright }}<copyright>{{ replace (replace . "{year}" now.Year) "&copy;" "©" | plainify }}</copyright>{{end -}}
    {{- if not .Date.IsZero }}<lastBuildDate>{{ .Date.Format "Mon, 02 Jan 2006 15:04:05 -0700" | safeHTML }}</lastBuildDate>{{ end -}}
    {{- if .Scratch.Get "og_image" }}
    <image>
      <url>{{ .Scratch.Get "og_image" }}</url>
      <title>{{ .Title | default site.Title }}</title>
      <link>{{ .Permalink }}</link>
    </image>
    {{end -}}
    {{ range $pages }}
    <item>
      <title>{{ .Title }}</title>
      <link>{{ .Permalink }}</link>
      <pubDate>{{ .Date.Format "Mon, 02 Jan 2006 15:04:05 -0700" | safeHTML }}</pubDate>
      <guid>{{ .Permalink }}</guid>
      <description>{{ .Content | html }}</description>
    </item>
    {{ end }}
  </channel>
</rss>
~~~

​		直接使用默认模板会出现报错，也就是这个问题。这个问题我在其他博主的帖子中发现是因为主页文章中使用到了<u>“一般来说，出现这些问题是因为 Markdown 文件中存在一些特殊字符，例如 `^H`、`^E` 等字符”</u>。那么为了避免这个问题官方有两个解决方法，第一通过Hugo正则表达式替换特殊字符，第二是在Markdown文件中删除特殊字符。

~~~error
This page contains the following errors:
error on line 7750 at column 66: PCDATA invalid Char value 2
Below is a rendering of the page up to the first error.
~~~

​		这两种方法我都没有使用，我直接将`.Content`替换为`.Summary`即可只在RSS中只显示文章部分。其实其中的问题我可能也没有解决，只是瞎猫碰到死耗子，碰巧给我弄对了，以后有机会了再去研究这个问题。现在主要是想把这个订阅源做出来。

​		`第三步`，在主题的配置文件中（themes/virgo/config.toml）设置主页中Rss跳转链接。

~~~toml
[module]
    [module.hugoVersion]
        #  extended = true
        #    min = "0.55.0"
        #    max = "0.84.2"
   [menu]
         [[menu.main]]
            identifier = "rss"
            name = "RSS"
            url = "/index.xml"
            weight = 5
~~~

# 4

​		以上是记录订阅源生成的过程和想法。















