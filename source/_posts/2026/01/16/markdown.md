---
title: Markdown
date: 2026-01-16 21:43:02
tags:
---

{% mermaid graph TD %}
    A-->B;
    A-->C;
    B-->D;
    C-->D;
{% endmermaid %}

```mermaid
graph TD;
    A-->B;
    A-->C;
    B-->D;
    C-->D;
```

note、label、button 和 tabs

<!-- more -->

# Note

`default, primary, success, info, warning, danger`

{% note default %}
我是 Markdown Note
{% endnote %}

{% note primary %}
我是 Markdown Note
{% endnote %}

{% note success %}
我是 Markdown Note
{% endnote %}

{% note info %}
我是 Markdown Note
{% endnote %}

{% note warning %}
我是 Markdown Note
{% endnote %}

{% note danger %}
我是 Markdown Note
{% endnote %}

{% note info no-icon %}
我是 Markdown Note
{% endnote %}

# Label

{% label default @default %}

{% label primary @primary %}

{% label success @success %}

{% label info @info %}

{% label warning @warning %}

{% label danger @danger %}

# Tabs

{% tabs Tabs, 1 %}
<!-- tab Daihatsu Hijet Caddie@fa fa-car -->
{% asset_img "Daihatsu Hijet Caddie.jpg" 480 "Daihatsu Hijet Caddie.jpg'Daihatsu Hijet Caddie.jpg'" %}
<!-- endtab -->
<!-- tab Mazda Flair Wagon Tough Style@fa fa-car -->
{% asset_img "Mazda Flair Wagon Tough Style.png" 480 "Mazda Flair Wagon Tough Style'Mazda Flair Wagon Tough Style'" %}
<!-- endtab -->
<!-- tab Suzuki Spacia Gear@fa fa-car -->
{% asset_img "Suzuki Spacia Gear.jpg" 480 "Suzuki Spacia Gear.jpg'Suzuki Spacia Gear.jpg'" %}
<!-- endtab -->
{% endtabs %}

# Button

{% button /about/, about, fas fa-info-circle, 關於我 %}
{% button <https://www.google.com/>, 前往 Google, fab fa-google, 搜索引擎 %}

---

```text Hexo Tag Plugins
{% asset_img "Mazda Flair Wagon Tough Style.png" 480 "Mazda Flair Wagon Tough Style'Mazda Flair Wagon Tough Style'" %}
```

{% asset_img "Mazda Flair Wagon Tough Style.png" 480 "Mazda Flair Wagon Tough Style'Mazda Flair Wagon Tough Style'" %}

```html HTML & Hexo Tag Plugins
<img src="{% asset_path 'Mazda Flair Wagon Tough Style.png' %}"
alt="Mazda Flair Wagon Tough Style"
title="Mazda Flair Wagon Tough Style"
width="480"
style="max-width: 100%; height: auto;">
```

<img src="{% asset_path 'Mazda Flair Wagon Tough Style.png' %}"
alt="Mazda Flair Wagon Tough Style"
title="Mazda Flair Wagon Tough Style"
width="480"
style="max-width: 100%; height: auto;">
