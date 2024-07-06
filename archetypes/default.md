
---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
draft: false
categories: [ "默认分类"]
isCJKLanguage: true
slug: "{{ substr (md5 (printf "%s%s" .Date (replace .TranslationBaseName "-" " " | title))) 4 8 }}"
toc: false
mermaid: false
fancybox: false
blueprint: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
# UEVersion: 5.3.2 
---

