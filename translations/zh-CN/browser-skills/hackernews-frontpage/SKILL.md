---
name: hackernews-frontpage
description: 抓取 Hacker News 首页(标题、点赞数、评论数)。
host: news.ycombinator.com
trusted: true
source: human
version: 1.0.0
args: []
triggers:
  - scrape hacker news frontpage
  - scrape hn frontpage
  - get hn top stories
  - latest hacker news stories
---

# Hacker News 首页抓取器

抓取 Hacker News (`news.ycombinator.com`) 首页并以 JSON 格式返回前 30 条故事。每条故事包含其排名、标题、链接 URL、点赞数和评论数。

## 使用方法

```
$ $B skill run hackernews-frontpage
{
  "stories": [
    { "rank": 1, "title": "...", "url": "...", "points": 412, "comments": 87 },
    ...
  ],
  "count": 30
}
```

## 工作原理

1. 通过守护进程导航到 `https://news.ycombinator.com`。
2. 读取页面 HTML。
3. 将每个故事行(HN 稳定的 `tr.athing` 结构)解析为类型化的 `Story` 记录。
4. 在标准输出上输出单个 JSON 文档。

## 为什么这是参考技能

`hackernews-frontpage` 是最小的有趣浏览器技能:无需认证、HTML 稳定、输出确定、易于文件固件测试。Phase 1 的每个组件(SDK、作用域令牌、三层查找、生成生命周期)都通过 `$B skill run hackernews-frontpage` 和捆绑的 `script.test.ts` 进行了测试。

当 HN HTML 发生变化导致我们的选择器失效时,测试会在用户注意到之前针对捕获的固件失败。这就是重点所在。