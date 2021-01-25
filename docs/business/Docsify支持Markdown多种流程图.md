[TOC]

## 目标

*  实现docsify支持Markdown多种流程图

  参考：[使用 docsify 并定制以使它更强大](https://ld246.com/article/1553507125889)

## docsify

官方文档：https://docsify.js.org/#/

docsify 可以快速帮你生成文档网站。不同于 GitBook、Hexo 的地方是它不会生成静态的 `.html` 文件，所有转换工作都是在运行时。如果你想要开始使用它，只需要创建一个 `index.html` 就可以开始编写文档并直接[部署在 GitHub Pages](https://docsify.js.org/#/zh-cn/deploy)。

### Markdown  文档

一般语法docsify 都是支持的，但是**对于Markdown 需要支持流程图需要加入配置才能生效**，修改index.html 进行配置。

### 支持 mermaid

官方文档：https://docsify.js.org/#/zh-cn/markdown?id=%e6%94%af%e6%8c%81-mermaid

这个是官方上配置的，可以发现很简单。

引入支持mermaid 的css 和 js。并配置js代码。js代码的逻辑也很简单，判断语法格式是 mermaid ，就把 code （代码）使用 mermaid  的js 执行以下，拼装一个 div 加入当前页。

```js
// Import mermaid
//  <link rel="stylesheet" href="//cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.css">
//  <script src="//cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.js"></script>
 
var num = 0;
mermaid.initialize({ startOnLoad: false });
 
window.$docsify = {
  markdown: {
    renderer: {
      code: function(code, lang) {
          // 判断语言是否为 mermaid
        if (lang === "mermaid") {
            // 将div 返回
          return (
            '<div class="mermaid">' + mermaid.render('mermaid-svg-' + num++, code) + "</div>"
          );
        }
        return this.origin.code.apply(this, arguments);
      }
    }
  }
}
```


![sOQrzF.png](https://s3.ax1x.com/2021/01/25/sOQrzF.png)

### 支持 DOT 语言作图

参考：https://ld246.com/article/1553507125889

仿照官方文档对 mermaid 的支持，进行配置

```js
<script src="https://cdn.bootcss.com/viz.js/1.8.0/viz.js"></script>
 
if (lang === "dot") {
    return (
    '<div class="viz">'+ Viz(code, "SVG")+'</div>'
    );
}
 
 
```

### 支持 LaTex 数学公式

```js
<link href="https://cdn.bootcss.com/KaTeX/0.10.0/katex.min.css" rel="stylesheet">
<script src="https://cdn.bootcss.com/KaTeX/0.10.0/katex.min.js"></script>
 
 
if (lang === "tex") {
    return (
    '<span class="tex">'+ katex.renderToString(code, {
        throwOnError: false
    })+'</span>'
    );
}
 
 
```

### 支持 Flow 流程图

https://github.com/adrai/flowchart.js官方地址

参考 flowchart 的例子：https://github.com/adrai/flowchart.js/blob/master/example/index.html 实现加入 flow 的语法支持

具体可以代码参考：https://github.com/gengzi/Code-animal/blob/master/index.html

```js
  <script src="//cdnjs.cloudflare.com/ajax/libs/raphael/2.3.0/raphael.min.js"></script>
  <script src="//cdnjs.cloudflare.com/ajax/libs/jquery/1.11.0/jquery.min.js"></script>
  <script src="//cdn.bootcdn.net/ajax/libs/flowchart/1.15.0/flowchart.js"></script>
 
 
<script>
            var flownow = 0;
            var canvasnow = 0;
            window.onload = function () {
            for(var i = 0;i<flownow;i++){
            var cd = document.getElementById("gengzi_code_"+i),
                    chart;
                    var code = cd.value;
                    chart = flowchart.parse(code);
                    chart.drawSVG('canvas'+i, {
                      // 'x': 30,
                      // 'y': 50,
                      'line-width': 3,
                      'maxWidth': 3,//ensures the flowcharts fits within a certian width
                      'line-length': 50,
                      'text-margin': 10,
                      'font-size': 14,
                      'font': 'normal',
                      'font-family': 'Helvetica',
                      'font-weight': 'normal',
                      'font-color': 'black',
                      'line-color': 'black',
                      'element-color': 'black',
                      'fill': 'white',
                      'yes-text': 'yes',
                      'no-text': 'no',
                      'arrow-end': 'block',
                      'scale': 1,
                      'symbols': {
                        'start': {
                          'font-color': 'red',
                          'element-color': 'green',
                          'fill': 'yellow'
                        },
                        'end':{
                          'class': 'end-element'
                        }
                      }
                    });
            }
            };
 
        </script>
 
 
if (lang === "flow") {
              return (
                '<div><textarea id="gengzi_code_'+(flownow++)+'" style="width: 100%;display:none;" rows="11"  >' + code + "</textarea></div><div id='canvas"+(canvasnow++)+"'></div>"
              );
            }
 
 
```

![sOQDRU.png](https://s3.ax1x.com/2021/01/25/sOQDRU.png)