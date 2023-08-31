# pdfjs-dist 在vite4.x 下的使用问题

![pic20230831](https://cherish-1256678432.cos.ap-nanjing.myqcloud.com/typora/pic20230831.jpg)

## 背景

> 项目技术栈：vue3 + vite + ts
> 
> pdfjs-dist版本：2.9.359

最近将项目从js升级到ts了，主要过程虽然踩坑不少，但是结果还是好的，顺利升级完成。

因为某些版本方面的原因，vitest 的vite.config.ts配置一直提示ts异常，解决方案是：升级vitest到3.x, 升级 vite （从vite 2.7 升级到 4.x）。 升级后 pdf浏览组件 出现一些问题。

先看升级前的写法：
```html
<template>
  <div class="pdf-container">
    <canvas v-for="pageIndex in pdfPages" :id="`pdf-canvas-${pageIndex}`"  :key="pageIndex" />
  </div>
</template>
<script setup lang="ts">
import * as PDFJS from 'pdfjs-dist/legacy/build/pdf.js'
// 这里导入的是 pdf.worker.entry.js
import * as PdfWorker from 'pdfjs-dist/legacy/build/pdf.worker.entry.js'
import { nextTick, ref, Ref, watch } from 'vue'
import { Log } from '@/utils'
import { isEmpty, debounce } from 'lodash-es'
import { ElNotification } from 'element-plus'


const props:any = defineProps({
  pdf: {
    required: true
  }
})

let pdfDoc:any = null
const pdfPages:Ref = ref(0)
const pdfScale:Ref = ref(1.3)
const loadFile = async (url:any) => {
  // 设定pdfjs的 workerSrc 参数
  PDFJS.GlobalWorkerOptions.workerSrc = PdfWorker
  const loadingTask = PDFJS.getDocument(url)
  loadingTask.promise.then(async (pdf:any) => {
    pdfDoc = pdf // 保存加载的pdf文件流
    pdfPages.value = pdfDoc.numPages // 获取pdf文件的总页数
    await nextTick(() => {
      renderPage(1) // 将pdf文件内容渲染到canvas
    })
  }).catch((error:any) => {
    Log.Warn(`[eCloudFilm] pdfReader loadFile error: ${error}`)
    ElNotification.error({
      message: '获取pdf报告失败'
    })
  })
}

const renderPage = (num:any) => {
  pdfDoc.getPage(num).then((page:any) => {
    page.cleanup()
    const canvas:any = document.getElementById(`pdf-canvas-${num}`)
    if (canvas) {
      const ctx = canvas.getContext('2d')
      const dpr = window.devicePixelRatio || 1
      const bsr = ctx.webkitBackingStorePixelRatio ||
                  ctx.mozBackingStorePixelRatio ||
                  ctx.msBackingStorePixelRatio ||
                  ctx.oBackingStorePixelRatio ||
                  ctx.backingStorePixelRatio ||
                  1
      const ratio = dpr / bsr
      const viewport = page.getViewport({ scale: pdfScale.value })
      canvas.width = viewport.width * ratio
      canvas.height = viewport.height * ratio
      canvas.style.width = viewport.width + 'px'
      canvas.style.height = viewport.height + 'px'
      ctx.setTransform(ratio, 0, 0, ratio, 0, 0)
      const renderContext = {
        canvasContext: ctx,
        viewport: viewport
      }
      page.render(renderContext)
      if (num < pdfPages.value) {
        renderPage(num + 1)
      }
    }
  })
}

const debouncedLoadFile = debounce((pdf:any) => loadFile(pdf), 1000)
watch(() => props.pdf, (newValue:any) => {
  !isEmpty(newValue) && debouncedLoadFile(newValue)
}, {
  immediate: true
})
</script>

```
pdfjs-dist 导入部分为：
```ts
import * as PDFJS from 'pdfjs-dist/legacy/build/pdf.js'
// 这里导入的是 pdf.worker.entry.js
import * as PdfWorker from 'pdfjs-dist/legacy/build/pdf.worker.entry.js'
```
升级到vite 4.x 后，因为 vite 的某些原因，

```ts
import * as PdfWorker from 'pdfjs-dist/legacy/build/pdf.worker.entry.js'
```
这样导入程序编译会直接报错，pdf.worker.entry.js 内容如下：
```ts
(typeof window !== "undefined"
  ? window
  : {}
).pdfjsWorker = require("./pdf.worker.js");

```
已知报错跟这个文件的模块化方式有关，具体原因不详述。
（vite会将各类其他标准模块化的文件转化为 es 标准模式， 这个文件在转化的过程中出现问题，不过在vite2.7中正常，可能跟vite的某些变更有关，此处不深究）。

通过查看 .vite 下的编译缓存文件，对 vite2.7 下编译文件的对比，

- vite2.7 打包的 pdf.worker.entry.js

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c98f29b8855546d9a3234d7e79290487~tplv-k3u1fbpfcp-watermark.image?)


- vite 4.43 打包的 pdf.worker.entry.js

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f14f30d47e5e498fbc0af1c51a395baf~tplv-k3u1fbpfcp-watermark.image?)

- vite4.43 打包的 pdf.worker.js

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cc5cdad59cd246aeb5efbf0f649a1da5~tplv-k3u1fbpfcp-watermark.image?)

通过对比，发现 vite2.7 打包的 pdf.worker.entry.js 和 vite4.43 打包的 pdf.worker.js 文件基本一致，都是 esmodule 形式的模块，因此改引入 `pdf.worker.entry.js` 为 `pdf.worker.js`, 修改后果然能正常编译。

但是，pdf 不能正常显示，报如下错误：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7125476d631643479efe69397252d601~tplv-k3u1fbpfcp-watermark.image?)

```js
Error: Setting up fake worker failed: 
"Cannot read properties of undefined (reading 'WorkerMessageHandler')".
```

## 原因分析


vite2.7 中 pdf.worker.entry.js 除了将pdf.worker.js 转成esmodule后，还将 pdfjsWorker 手动挂载到 window 对象上，而vite4.3中，没有此操作。
但是，在pdfjs的渲染过程中，代码中用到了 window.pdfjsWorker 对象，因此，在 vite 4.3 下通过直接导入 `pdf.worker.js` 使用时，程序会抛出异常。


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c968b06500b44891a76d0b6120b3cdf5~tplv-k3u1fbpfcp-watermark.image?)

## 解决方案
vite 4.3 下，在程序中手动挂载 pdfjsWorker 对象到 window上。
```js
// In vite4, it is not support to import pdfWorker from pdf.worker.entry.js // So we should to set window.pdfjsWorker ourself window.pdfjsWorker = PdfWorker
window.pdfjsWorker = PdfWorker
```

以下是修改后的实现：

```html
<template>
  <div class="pdf-container">
    <canvas v-for="pageIndex in pdfPages" :id="`pdf-canvas-${pageIndex}`"  :key="pageIndex" />
  </div>
</template>
<script setup lang="ts">
import * as PDFJS from 'pdfjs-dist/legacy/build/pdf.js'
import * as PdfWorker from 'pdfjs-dist/build/pdf.worker.js'
import { nextTick, ref, Ref, watch } from 'vue'
import { Log } from '@/utils'
import { isEmpty, debounce } from 'lodash-es'
import { ElNotification } from 'element-plus'

const props:any = defineProps({
  pdf: {
    required: true
  }
})

// In vite4, it is not support to import pdfWorker from pdf.worker.entry.js
// So we should to set window.pdfjsWorker ourself
window.pdfjsWorker = PdfWorker
let pdfDoc:any = null
const pdfPages:Ref = ref(0)
const pdfScale:Ref = ref(1.3)
const loadFile = async (url:any) => {
  // 设定pdfjs的 workerSrc 参数
  PDFJS.GlobalWorkerOptions.workerSrc = PdfWorker
  const loadingTask = PDFJS.getDocument(url)
  loadingTask.promise.then(async (pdf:any) => {
    pdfDoc = pdf // 保存加载的pdf文件流
    pdfPages.value = pdfDoc.numPages // 获取pdf文件的总页数
    await nextTick(() => {
      renderPage(1) // 将pdf文件内容渲染到canvas
    })
  }).catch((error:any) => {
    Log.Warn(`[eCloudFilm] pdfReader loadFile error: ${error}`)
    ElNotification.error({
      message: '获取pdf报告失败'
    })
  })
}

const renderPage = (num:any) => {
  pdfDoc.getPage(num).then((page:any) => {
    page.cleanup()
    const canvas:any = document.getElementById(`pdf-canvas-${num}`)
    if (canvas) {
      const ctx = canvas.getContext('2d')
      const dpr = window.devicePixelRatio || 1
      const bsr = ctx.webkitBackingStorePixelRatio ||
                  ctx.mozBackingStorePixelRatio ||
                  ctx.msBackingStorePixelRatio ||
                  ctx.oBackingStorePixelRatio ||
                  ctx.backingStorePixelRatio ||
                  1
      const ratio = dpr / bsr
      const viewport = page.getViewport({ scale: pdfScale.value })
      canvas.width = viewport.width * ratio
      canvas.height = viewport.height * ratio
      canvas.style.width = viewport.width + 'px'
      canvas.style.height = viewport.height + 'px'
      ctx.setTransform(ratio, 0, 0, ratio, 0, 0)
      const renderContext = {
        canvasContext: ctx,
        viewport: viewport
      }
      page.render(renderContext)
      if (num < pdfPages.value) {
        renderPage(num + 1)
      }
    }
  })
}

const debouncedLoadFile = debounce((pdf:any) => loadFile(pdf), 1000)
watch(() => props.pdf, (newValue:any) => {
  !isEmpty(newValue) && debouncedLoadFile(newValue)
}, {
  immediate: true
})
</script>

```

## TODO
后面有时间给 vite 提个issue
