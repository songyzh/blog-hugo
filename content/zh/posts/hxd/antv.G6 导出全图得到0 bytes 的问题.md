---
title: "antv.G6 导出全图得到0 bytes 的问题"
slug: "antv-g6-export-pic-zero-bytes"
date: "2020-08-23T12:00:06+08:00"
description: ""

draft: false

hideToc: false
enableToc: true
enableTocContent: false

author: "东"
authorEmoji: ""
authorImage: ""
authorImageUrl: ""
authorDesc: ""
socialOptions:
  email: "mailto:1619882712@qq.com"
  github: "https://github.com/happyElina"
  weibo: "https://weibo.com/u/2669254565"

tags:
  - 前端
categories:
  - 技术
series:
  -

image: "https://tva1.sinaimg.cn/large/007S8ZIlgy1gi1mrea3s9j30l909xweq.jpg"

libraries:
  - katex
  - chart
  - flowchartjs
  - msc
  - mathjax
  - mermaid
  - viz
  - wavedrom
---

#####  一 问题背景

最近在使用[G6](https://antv.vision/zh) 展示树状图的时候需要有导出图片的需求，于是使用graph.downloadFullImage 方法来导出canvas 内容到一个图片，在导出的过程中发现，如果树的内容量庞大，会出现导出的图片是0kb 的情况，本文就是对此问题的一个记录。

##### 二 尝试解决过程

首先找到了源码中graph.downloadFullImage 的位置，/node_modules/_@antv_g6@3.6.1@@antv/g6/es/graph/graph.js，在调用

graph.downloadFullImage 方法前设置断点，发现进入的是压缩后文件，原来package.json 中默认入口设置的是 "main": "lib/index.js",，为了可以进入源码调试，将引入G6 处修改：

```javascript
// import G6ES from '@antv/g6';
import G6ES from '@antv/g6/es/index'; // 调试源码走这里
```

再次调试可以进入未压缩的源码中。



在调试过程中，发现不同的数据，在调用canvas.toDataURL 方法返回的内容不同，由此发现了出现问题的位置,下图分别展示了数量小和数据量大的时候调试到此处的情形,都是在node_modules/_@antv_g6@3.6.1@@antv/g6/es/graph/graph.js的downloadFullImage方法内

![small](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghzykhavbij310u0miqc5.jpg)



![big](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghzyk1xuqfj30wc0nsdom.jpg)

发现了这个问题后，决定在本地代码中验证是否是这个原因，于是在方法中自己调用canvas.toDataURL 查看：

```javascript
// 省略部分react 组件代码
let [imgSrc,setImgSrc] = useState('');

const handleMyExport = () => {
  const canvas = document.getElementsByTagName('canvas')[0];
  var dataURL = canvas.toDataURL();
  console.log(dataURL);
  setImgSrc(dataURL);
 }

return (
    <div className="App" >
      <button style={{position:"absolute",top: 0,left: '110px',zIndex: 999}} onClick={handleMyExport}>我的导出</button>
      <div ref={ref}></div>
      < img src={imgSrc} alt='' style={{position:'absolute', top: "820px"}}/>
    </div>
  );
```



在调试过程发现使用大量数据的时候，这里的canvas.toDataURL() 可以生成图片base64 编码，在img 标签中是可以展示图片内容的，这是什么原因？对比发现，这里的图片只展示的canvas 画布中的内容，远小于真正cavans 中大数据量需要展示的数据，downloadFullImage导出的图片是包括canvas 中不可见部分的内容的，由此可见是导出全图的大数量数据导致的0kb 问题。

继续查看downloadFullImage 方法，发现他在导出全图的过程是通过生成一个新canvas 标签（new GCanvas(canvasOptions); 宽高是canvas 全部内容的宽高），不是在当前graph 所拥有的canvas 标签上导出的，以下摘出downloadFullImage 方法并添加部分注释

```javascript
/**
   * 导出包含全图的图片
   * @param {String} name 图片的名称
   * @param {String} type 图片类型，可选值："image/png" | "image/jpeg" | "image/webp" | "image/bmp"
   * @param {Object} imageConfig 图片配置项，包括背景色和上下左右的 padding
   */

  Graph.prototype.downloadFullImage = function (name, type, imageConfig) {
    var _this = this;
    var bbox = this.get('group').getCanvasBBox(); // 获取画布的包围盒 包括不可见部分内容
    var height = bbox.height;
    var width = bbox.width;
    var renderer = this.get('renderer'); // canvas
    var vContainerDOM = createDom('<id="virtual-image"></div>');
    var backgroundColor = imageConfig ? imageConfig.backgroundColor : undefined;
    var padding = imageConfig ? imageConfig.padding : undefined;
    if (!padding) padding = [0, 0, 0, 0];else if (isNumber(padding)) padding = [padding, padding, padding, padding];
    var vHeight = height + padding[0] + padding[2]; // 将要生成的canvas 高度
    var vWidth = width + padding[1] + padding[3]; // 将要生成的canvas宽度
    var canvasOptions = {
      container: vContainerDOM,
      height: vHeight,
      width: vWidth
    };
    var vCanvas = renderer === 'svg' ? new GSVGCanvas(canvasOptions) : new GCanvas(canvasOptions);
    var group = this.get('group');
    var vGroup = group.clone();
    var matrix = clone(vGroup.getMatrix());
    if (!matrix) matrix = [1, 0, 0, 0, 1, 0, 0, 0, 1];
    var centerX = (bbox.maxX + bbox.minX) / 2;
    var centerY = (bbox.maxY + bbox.minY) / 2;
    mat3.translate(matrix, matrix, [-centerX, -centerY]);
    mat3.translate(matrix, matrix, [width / 2 + padding[3], height / 2 + padding[0]]);
    vGroup.resetMatrix();
    vGroup.setMatrix(matrix);
    vCanvas.add(vGroup);
    var vCanvasEl = vCanvas.get('el'); // 将要生成图片的canvas
    if (!type) type = 'image/png';
    setTimeout(function () {
      var dataURL = '';

      if (renderer === 'svg') {
       ...
      } else {
        var imageData = void 0;
        var context = vCanvasEl.getContext('2d');
        var compositeOperation = void 0;

        if (backgroundColor) {
          var pixelRatio = window.devicePixelRatio;
          imageData = context.getImageData(0, 0, vWidth * pixelRatio, vHeight * pixelRatio);
          compositeOperation = context.globalCompositeOperation;
          context.globalCompositeOperation = "destination-over";
          context.fillStyle = backgroundColor;
          context.fillRect(0, 0, vWidth, vHeight);
        }

        dataURL = vCanvasEl.toDataURL(type); // 生成图片base64 编码
      }

      // 生成a链接完成下载效果
      var link = document.createElement('a');
      var fileName = (name || 'graph') + (renderer === 'svg' ? '.svg' : "." + type.split('/')[1]);

      _this.dataURLToImage(dataURL, renderer, link, fileName);

      var e = document.createEvent('MouseEvents');
      e.initEvent('click', false, false);
      link.dispatchEvent(e);
    }, 16);
  };
```

查看canvas 的 [toDataURL](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLCanvasElement/toDataURL) 方法，发现指出'如果画布的高度或宽度是0，那么会返回字符串“`data:,”。`',再次调试，发现vCanvasEl 的宽度和高度都不为0:

![屏幕快照 2020-08-23 上午11.01.25](https://tva1.sinaimg.cn/large/007S8ZIlgy1gi0keed50xj30pc060q43.jpg)

![屏幕快照 2020-08-23 上午11.01.51](https://tva1.sinaimg.cn/large/007S8ZIlgy1gi0kejzs53j30oi0fowhv.jpg)

经查canvas 元素大小在各浏览器中都有限制，

![屏幕快照 2020-08-23 下午7.02.44](https://tva1.sinaimg.cn/large/007S8ZIlgy1gi0y8eccbxj31740rydkq.jpg)

那么针对这个限制如何应用于我们的导出需求呢？以下是我的几种方案：

1、 针对导出做限制，当大于我们的极限时，提醒用户当前图片太大，可以选择树结构的某一部分来导出。这个想法是基于我们导出图片的初衷的，初衷是为了让用户清晰的看出整个树结构，当我测试时导出的3.4M 的一张图已经大到每个节点都需要非常费力去放大才能看到，这与初衷相违背，不能为了导出而导出，产品需要考虑到这种情况通过交互去优化体验。

2 、在一定范围内考虑通过scale 方法缩放后导出，在不影响导出图片的结构清晰的前提下。

3 、还是非常想得到全图，就可以在downloadFullImage 方法的基础上，添加对图片宽高的判断和计算，比如将图片切割为上下左右4个部分，得到每个部分的canvas限制内的图片base64,上传到后端，由后端小伙伴拼接成一张图返回，此方法基于[用canvas 实现截图功能](https://blog.csdn.net/HuangsTing/article/details/106141263)，麻烦的是要根据当前canvas 的宽高来决定如何分割，总体上是可以实现的。



以上，这个问题暂时告一段落。



参考：
- https://g6.antv.vision/zh/docs/api/Graph

- https://stackoverflow.com/questions/43860035/canvas-todataurl-returns-data-when-canvas-width-height-is-too-large

- https://blog.csdn.net/HuangsTing/article/details/106141263