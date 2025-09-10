---
title: bangumi收藏卡片
date: 2024-07-16 18:43:55
tags: [github, 动画]
---

## 简介

一个github actions用于生成[bangumi](https://bgm.tv)收藏信息卡片。

效果预览:

![p](https://github.com/Kom3ng/Kom3ng/raw/main/bgm-card.svg)

## Github Actions 使用示例

你只需要在你要使用该 Action 的仓库中，配置 `bgm-user-id` 和 `bgm-img-path` 即可，其它可以不用动。
```yaml
name: Bgm-Data-Sync
on:
  # 每日零点执行（这个配置最小间隔5分钟）
  schedule: [{ cron: '0 0 * * *' }]
  # 手动触发
  workflow_dispatch:
  # master/main 分支有提交时执行
  push: {branches: ["master", "main"]}

jobs:
  bgm-sync:
    runs-on: ubuntu-latest
    name: 每日同步BGM收藏卡片
    permissions:
      contents: write
    steps:
      - name: Bgm Collection Card
        id: bgm
        uses: Kom3ng/bangumi-action@latest
        with:
          github-token: '${{secrets.GITHUB_TOKEN}}'
          bgm-user-id: 'astrack'
          bgm-img-path: 'bgm/card.svg'
          show-animes: true
          show-games: true
          show-mangas: true
          show-characters: true

      - name: Get the output image url
        run: echo "图片生成的链接地址 ${{ steps.bgm.outputs.message }}"
```

## 详细参数

- ### `github-token`
  Action Token 固定值：`${{secrets.GITHUB_TOKEN}}` 

- ### `bgm-user-id`

  Bangumi.TV 的用户 ID，如果你已经设置了用户名，请使用用户名，如 `xiaoyvyv`。

- ### `bgm-img-path`

  生成成功后，上传的当前仓库的路径，如 `bgm/collection.svg` 则会上传到当前仓库 `bgm` 文件夹下，需要以 `.svg` 结尾。

- ### `show-animes`
  
  是否生成动画收藏信息。

- ### `show-games`

  是否生成游戏收藏信息。

- ### `show-mangas`

  是否生成书籍收藏信息。

- ### `show-characters`

  是否生成角色收藏信息。

## 相关

[项目仓库地址](https://github.com/Kom3ng/bangumi-action)

[原项目仓库地址](https://github.com/xiaoyvyv/bangumi-action)