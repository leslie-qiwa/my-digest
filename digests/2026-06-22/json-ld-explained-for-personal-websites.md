# JSON-LD explained for personal websites

# 面向个人网站的 JSON-LD 详解

JSON-LD，全称 JSON 链接数据（JSON Linked Data），是一种向网页添加结构化数据的格式。它可以帮助网络爬虫理解网站的语义结构，使你有资格获得更丰富的链接预览，甚至可能提升搜索排名。

自从我第一篇描述构建本站的文章发布以来，已经过去了 4 个月，Wakatime 估计我已经花了约 100 小时编写代码，还不包括研究和测试的时间。从那以后，这个网站经过了大量打磨，包括在每个页面上添加了 JSON-LD。

## JSON-LD 基础

要在页面中添加 JSON-LD，请在 `<head>` 部分的某处添加以下内容：

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": "WebSite",
      "@id": "https://hawksley.dev/#website",
      "url": "https://hawksley.dev/",
      "name": "Ethan Hawksley"
    },
    // 在此处插入更多节点。
  ]
}
</script>
```

让我们逐一解析各部分的作用。

```html
<script type="application/ld+json">
```

这声明了一个 MIME 类型为 `application/ld+json` 的新脚本。由于指定了此类型，浏览器的 JS 引擎不会执行它。像 Googlebot 这样的专用爬虫会查找这些元素并解析其内容。

```json
{
  "@context": "https://schema.org"
}
```

这里初始化了一个 JSON 对象，并将 `@context` 属性设置为 `https://schema.org`。在 JSON-LD 中，数据的结构由分配适当的上下文来决定。网络爬虫以 Schema.org（在新标签页中打开）为标准，该标准定义了 JSON 中所有有效的键值对。

现在我们已经定义了 JSON-LD 所遵循的模式，可以开始描述我们的网页了！

```json
{
  "@graph": [
    {
      "@type": "WebSite",
      "@id": "https://hawksley.dev/#website",
      "url": "https://hawksley.dev/",
      "name": "Ethan Hawksley"
    }
    // 在此处插入更多节点。
  ]
}
```

JSON-LD 文档可以被看作一个带标签的有向图，存储在 `@graph` 下。该图包含多个节点，通过有向弧相互连接。

节点具有以下属性：

- `@type` — 描述节点是什么，例如 `WebSite` 或 `SoftwareApplication`
- `@id` — 节点的唯一标识符，通常是末尾带有唯一哈希值的 URL
- 属性 — 描述节点特征的键/值对

在上面的示例中，类型是 `WebSite`，ID 是 `https://hawksley.dev/#website`，它有两个属性：`url` 和 `name`。

网络爬虫可以跨多个页面合并具有相同 ID 的节点属性。然而，只读取单个页面的抓取器（如大语言模型）不会合并属性。当 JSON-LD 在多个页面中复用时，把握这一平衡非常重要。最佳实践是让 ID 为一个 URL 后跟一个哈希值，例如 `#website`，以唯一标识该节点。

尽管 Schema.org 上下文定义了许多类型的节点，但本指南只涵盖对 SEO 有明显影响的节点。如果你有兴趣了解更多，可以搜索语义网（Semantic Web）——这是一个有趣的深坑。

接下来让我们看看网站的每个页面应该包含哪些节点。对于每种类型，我都附上了本站的 JSON-LD，你可以直接复制粘贴并编辑以适配自己的网站。

## WebSite

你之前已经看过 `WebSite` 的摘要了！现在来看完整版本：

```json
{
  "@type": "WebSite",
  "@id": "https://hawksley.dev/#website",
  "url": "https://hawksley.dev/",
  "name": "Ethan Hawksley",
  "alternateName": ["hawksley.dev", "Hawksley"],
  "description": "Ethan Hawksley 的个人网站和技术博客……",
  "inLanguage": "en-GB",
  "publisher": {
    "@id": "https://hawksley.dev/#person"
  },
  "image": {
    "@type": "ImageObject",
    "@id": "https://hawksley.dev/#website-image",
    "url": "https://hawksley.dev/logo-square.png",
    "caption": "Ethan Hawksley Logo"
  }
}
```

`WebSite` 说明了网站的元数据，为爬虫提供了如何展示你的网站的提示。

在这里，你可以看到 Google 将 name 字段解读为域名的代表，并据此为搜索结果添加了相应的标签。

虽然 `WebSite` 适用于每个页面，但你不需要在每个页面都包含完整版本。域名的根页面应该详细填写，但其他页面使用精简版本是完全可以的：

```json
{
  "@type": "WebSite",
  "@id": "https://hawksley.dev/#website",
  "url": "https://hawksley.dev/",
  "name": "Ethan Hawksley"
}
```

这为单页爬虫提供了足够的上下文以正确命名网站，但不需要完整的详细信息。

## WebPage

`WebPage` 描述当前页面，但需要将其与 `BlogPosting`（稍后介绍）等其他类型区分开来。`WebPage` 代表页面本身，即 HTML，它包含页面的内容。

```json
{
  "@type": "WebPage",
  "@id": "https://hawksley.dev/blog/hack-club-campfire/#webpage",
  "url": "https://hawksley.dev/blog/hack-club-campfire/",
  "isPartOf": {
    "@id": "https://hawksley.dev/#website"
  },
  "name": "赢得 Hack Club Campfire 黑客马拉松",
  "inLanguage": "en-GB",
  "breadcrumb": {
    "@id": "https://hawksley.dev/blog/hack-club-campfire/#breadcrumb"
  }
}
```

`WebPage` 有更具体的子类型。在本文中，我将介绍 `ProfilePage` 和 `CollectionPage`。你可以在 Schema.org 对 WebPage 的定义（在新标签页中打开）底部找到其他不太常见的子类型。

## Person

个人网站上每个页面都应包含的另一个节点是 `Person`。它描述了你是谁，Google 将其作为内容质量指标的一部分。大语言模型爬虫也越来越多地使用它来决定在回答中引用谁。

与 `WebSite` 不同，`Person` 作为上下文足够重要，你应该将其包含在网站的所有页面上。

**警告 — 内容较长！**

```json
{
  "@type": "Person",
  "@id": "https://hawksley.dev/#person",
  "url": "https://hawksley.dev/",
  "name": "Ethan Hawksley",
  "alternateName": "ethanhawksley",
  "givenName": "Ethan",
  "familyName": "Hawksley",
  "description": "详细描述",
  "disambiguatingDescription": "简短描述",
  "jobTitle": "计算机科学学生",
  "knowsLanguage": "en-GB",
  "knowsAbout": [
    // 关键词
  ],
  "nationality": {
    "@type": "Country",
    "name": "United Kingdom"
  },
  "homeLocation": {
    "@type": "Place",
    "address": {
      "@type": "PostalAddress",
      "addressCountry": "GB"
    }
  },
  "affiliation": {
    "@type": "HighSchool",
    "url": "https://www.alcestergs.co.uk",
    "name": "Alcester Grammar School",
    "sameAs": [
      "https://www.wikidata.org/wiki/Q4713005",
      "https://en.wikipedia.org/wiki/Alcester_Grammar_School"
    ]
  },
  "alumniOf": [
    {
      "@type": "HighSchool",
      "url": "https://www.brookeweston.org",
      "name": "Brooke Weston Academy",
      "sameAs": [
        "https://www.wikidata.org/wiki/Q4974495",
        "https://en.wikipedia.org/wiki/Brooke_Weston_Academy"
      ]
    }
  ],
  "image": {
    "@type": "ImageObject",
    "@id": "https://hawksley.dev/#person-image",
    "url": "https://hawksley.dev/ethan-hawksley.png",
    "caption": "Ethan Hawksley",
    "width": 1536,
    "height": 1536
  },
  "sameAs": [
    "https://github.com/ethan-hawksley",
    "https://www.linkedin.com/in/ethanhawksley",
    "https://lobste.rs/~ethanhawksley",
    "https://news.ycombinator.com/user?id=ethanhawksley"
    // 等等
  ]
}
```

`Person` 有很多属性！我发现在填写时，描述越详细越好。让我们看看最重要的属性：

- `url` — 指向你的根页面，锚定该节点。
- `name`、`givenName`、`familyName` — 清楚地描述你的名字。
- `image` — 最好是你的照片，或你所属的标志。将你与一张规范的图像关联起来。
- `sameAs` — 对于消歧义极为有用，尤其是当你有一个常见名字时。它清晰地告诉爬虫你的其他个人资料页面在哪里，让它们能够在多个页面上构建你的知识图谱表示。

`Person` 的其他属性可以添加更多细节，但并非严格必要。你可以根据需要精简，只会产生较小的影响。

## ProfilePage
