<!-- ---
title: "Welcome to My Blog"
description: "Thoughts and conclusions about learning"
layout: "hero" # <-- 启用 hero 布局
---


<div class="flex px-4 py-2 mb-8 text-base rounded-md bg-primary-100 dark:bg-primary-900">
  <span class="flex items-center ltr:pr-3 rtl:pl-3 text-primary-400">
    {{< icon "triangle-exclamation" >}}
  </span>
  <span class="flex items-center justify-between grow dark:text-neutral-300">
    <span class="prose dark:prose-invert">This is a demo of the <code id="layout">background</code> layout.</span>
    <button
      id="switch-layout-button"
      class="px-4 !text-neutral !no-underline rounded-md bg-primary-600 hover:!bg-primary-500 dark:bg-primary-800 dark:hover:!bg-primary-700"
    >
      Switch layout &orarr;
    </button>
  </span>
</div>


```shell
npx blowfish-tools
```  

{{< youtubeLite id="SgXhGb-7QbU" label="Blowfish-tools demo" >}}

 -->
---
title: "Welcome to My Blog"
description: "Thoughts and conclusions about learning"
layout: "hero"
heroStyle: "background"

# 无需在这里添加 backgroundImage！
# Hugo 会自动查找并使用您在 params.toml 中设置的 defaultBackgroundImage。

buttons:
  - name: "文章归档"
    url: "/archives"
    style: "primary"
  - name: "关于我"
    url: "/about"
    style: "secondary"
---