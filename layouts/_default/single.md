---
title: "{{ .Title }}"
date: {{ .Date.Format "2006-01-02T15:04:05-07:00" }}
author: "{{ .Params.author | default .Site.Params.author }}"
categories: [{{ range .Params.categories }}"{{ . }}"{{ end }}]
tags: [{{ range .Params.tags }}"{{ . }}"{{ end }}]
---

{{ .RawContent }}
