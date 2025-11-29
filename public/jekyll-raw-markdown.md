---
title: Jekyllでpostにあるマークダウンをレンダリングなしで出力したい
tags:
  - Ruby
  - Markdown
  - Jekyll
  - unrendered
private: false
updated_at: '2021-04-06T14:30:19+09:00'
id: f44f5f04f8b6ee2f1318
organization_url_name: null
slide: false
ignorePublish: false
---
Jekyllはpostにあるマークダウンを`{{ content }}`にレンダリングして埋め込んでくれるのですが、このマークダウンをRAWデータとして埋め込みたいときの解決策

調べても出てこなかったので残しておきます。

# やり方

なにもしない"無"のレンダラーを自作します。

`_plugins`というフォルダを作ります。そこに適当なRubyファイルを作り、以下のコードを打ちます。

```ruby:_plugins/unrendered.rb
class Jekyll::Converters::Markdown::ProcessorUnrendered
  def initialize(config)
  end
  def convert(content)
    content
  end
end
```

次に、`_config.yml`でこのレンダラーを使うように設定します。

```yml:_config.yml
markdown: ProcessorUnrendered
```

以上です。

注意ですが、Front Matterは出力されません。
つまり、

```markdown
---
layout: default
---

# Hello
[Google](https://google.com)
```

というファイルは

```
# Hello
[Google](https://google.com)
```

このように出力されます。

# 何に使うの？

スライド資料を作れる[`Reveal.js`](https://revealjs.com/)と組み合わせるときに使うといい感じになります。

```html:slide.html
<div class="reveal">
  <div class="slides">
    <section data-markdown data-separator="^\r?\n---\r?\n$" data-separator-vertical="^\r?\n--\r?\n$">
      <script type="text/template">
        {{ content }}
      </script>
    </section>
  </div>
</div>
```

こんな感じで使う

# 参考にしたものなど

[Markdown Options](https://jekyllrb.com/docs/configuration/markdown/#custom-markdown-processors)
公式のやつです。カスタムmarkdownプロセッサについて詳しく書かれています


以上
