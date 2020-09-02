[TOC]

## 目标

* 了解HTML 转PDF一些方式

  参考：[java实现HTML转PDF ](https://my.oschina.net/960823/blog/1588166)

  [Headless Chrome 入门](https://zhuanlan.zhihu.com/p/29207391) 建议阅读

  [利用Chrome Headless模式,网页转PDF](https://blog.csdn.net/xcl168/article/details/75675781)

  [java实现HTML转PDF ](https://my.oschina.net/960823/blog/1588166)

  [Java操作wkhtmltopdf实现Html转PDF](https://www.cnblogs.com/xionggeclub/p/6144241.html)

  [最好用Html转pdf的工具——wkhtmltopdf](https://blog.csdn.net/qq_14873105/article/details/51394026)

## HTML 转 PDF

在某些业务场景下，需要把表单转换为pdf导出。

根据网上的一些参考，大致有三种实践方式，当然还有其他的

1. itextpdf类库
2. wkhtmltopdf 软件
3. Chrome  Headless（无头浏览器）

### textpdf类库

参考：[java实现HTML转PDF ](https://my.oschina.net/960823/blog/1588166)

官方地址：[itextpdf](https://itextpdf.com/)

特点：对于html格式检测非常严格

html 头部必须声明，例如

```
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">* <html lang="en" xmlns="http://www.w3.org/1999/xhtml">
```

标签都要加</xx>作为结束符,如果html不符合规定，转换时会报错。

* 针对中文转换问题

  下载对应的中文字体（ttf）

  在页面中需要指定字体样式

  ```css
  font-family:SimSun;
  ```

#### 代码实践：

环境：spring boot 2.2.7 jdk1.8  windows10 （linux建议测试，可能一些命令参数需要调整）

pom：针对jar的版本，可以再参考下官网

可以参考：[codycopy](https://github.com/gengzi/codecopy)

```xml
        <!-- https://mvnrepository.com/artifact/com.itextpdf/itext-asian -->
        <dependency>
            <groupId>com.itextpdf</groupId>
            <artifactId>itext-asian</artifactId>
            <version>5.2.0</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/com.itextpdf/itextpdf -->
        <dependency>
            <groupId>com.itextpdf</groupId>
            <artifactId>itextpdf</artifactId>
            <version>5.5.13</version>
        </dependency>
        <dependency>
            <groupId>org.xhtmlrenderer</groupId>
            <artifactId>flying-saucer-pdf</artifactId>
            <version>9.0.7</version>
        </dependency>
```

工具类：

[fun.gengzi.codecopy.business.utilstest.utils.HtmlToPdfUtils](https://github.com/gengzi/codecopy/tree/master/src/main/java/fun/gengzi/codecopy/business/utilstest/utils)

注意要下载需要的字体文件，复制到 resources/font 以便使用

```java
/**
     * html文本转换成PDF
     * <p>
     * 由于 ITextpdf 对html 检测非常严格，html 头部必须声明
     * <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
     * <html lang="en" xmlns="http://www.w3.org/1999/xhtml">
     * <p>
     * 以及其他的都要加</>结束，所以一般的页面都将不支持 转换 pdf
     *
     * @param url            网址
     * @param pdfPath        pdf生成路径
     * @param fontFamilyEnum {@link FontFamilyEnum}  中文字体信息
     */
    public static void htmlTextToPdf(URL url, String pdfPath, FontFamilyEnum fontFamilyEnum) {
        String body = HttpRequest.get(url.toString()).execute().body();
        htmlTextToPdf(body, pdfPath, fontFamilyEnum);
    }
 
 
    /**
     * html文本转换成PDF
     *
     * @param file           html 文件
     * @param pdfPath        pdf生成路径
     * @param fontFamilyEnum {@link FontFamilyEnum}  中文字体信息
     */
    public static void htmlTextToPdf(File file, String pdfPath, FontFamilyEnum fontFamilyEnum) {
        try {
            htmlTextToPdf(FileUtil.toString(file), pdfPath, fontFamilyEnum);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
 
 
    /**
     * html文本转换成PDF
     *
     * @param htmlText       html字符串
     * @param pdfPath        pdf生成路径
     * @param fontFamilyEnum {@link FontFamilyEnum}  中文字体信息
     */
    public static void htmlTextToPdf(String htmlText, String pdfPath, FontFamilyEnum fontFamilyEnum) {
        logger.info("html转pdf入参,htmltext: {} , pdfpath:{}", htmlText, pdfPath);
        // 为了支持中文，检查html内容中是否包含 font-family:SimSun
        boolean fontFamilyFlag = htmlText.contains("font-family");
        boolean fontNameFlag = htmlText.contains(fontFamilyEnum.getFontName());
        if (!fontFamilyFlag || !fontNameFlag) {
            logger.warn("--请注意，当前转换的html未包含字体文件，中文可能转换失败--");
        }
        try {
            TimeInterval timer = DateUtil.timer();
            OutputStream outputStream = new FileOutputStream(pdfPath);
            ITextRenderer iTextRenderer = new ITextRenderer();
            //解决中文字体，需要单独下载字体
            //同时在前端样式中加入font-family:SimSun;
            ITextFontResolver fontResolver = iTextRenderer.getFontResolver();
            fontResolver.addFont(Objects.requireNonNull(ClassLoader.getSystemClassLoader().getResource("font/" + fontFamilyEnum.fontFileName)).getPath(),
                    BaseFont.IDENTITY_H, BaseFont.NOT_EMBEDDED);
            iTextRenderer.setDocumentFromString(htmlText);
            iTextRenderer.layout();
            iTextRenderer.createPDF(outputStream);
            logger.info("转换耗时(毫秒)：{}", timer.interval());
            outputStream.close();
        } catch (IOException | DocumentException e) {
            throw new RrException("PDF转换异常", e);
        }
    }
 
    /**
     * 字体枚举类
     */
    public enum FontFamilyEnum {
 
        SIMSUN("SimSun", "simsun.ttf");
 
        // 字体名称
        private String fontName;
        // 字体文件
        private String fontFileName;
 
        FontFamilyEnum(String fontName, String fontFileName) {
            this.fontName = fontName;
            this.fontFileName = fontFileName;
        }
 
        public String getFontName() {
            return fontName;
        }
 
        public String getFontFileName() {
            return fontFileName;
        }
    }
```

 

### wkhtmltopdf 软件

参考：[Java操作wkhtmltopdf实现Html转PDF](https://www.cnblogs.com/xionggeclub/p/6144241.html)

[最好用Html转pdf的工具——wkhtmltopdf](https://blog.csdn.net/qq_14873105/article/details/51394026) 建议阅读

官方地址：[wkhtmltopdf](https://wkhtmltopdf.org/) 建议看下官方文档，非常简单明了

使用方式：

1. 下载wkhtmltopdf 对应系统的软件安装，基本的系统都支持。[下载地址](https://wkhtmltopdf.org/downloads.html)

2. 配置wkhtmltopdf 为全局的环境变量，方便使用（自行百度）

3. 代码调用命令行，执行转换命令

   ```
   wkhtmltopdf https://www.baidu.com baidu.pdf
   ```

#### 代码实践：

环境：spring boot 2.2.7 jdk1.8  windows10

配置全局变量：系统-高级系统设置 将wkhtmltopdf 的bin 目录配置到 path 中

代码参考：[fun.gengzi.codecopy.business.utilstest.utils.HtmlToPdfUtils](https://github.com/gengzi/codecopy/tree/master/src/main/java/fun/gengzi/codecopy/business/utilstest/utils)

java调用cmd 执行命令，使用了工具类 hutool

```java
 
    /**
     * 使用wkhtmltopdf 将html转pdf
     * <p>
     * 注意，需要安装  wkhtmltopdf 该软件
     * 需要将 wkhtmltopdf 配置到环境变量中，以便使用
     * 在测试时，注意当前执行main方法的环境变量中，是否包含了 wkhtmltopdf 的配置
     * idea会读取系统环境变量的配置，但是刚修改的好像不能及时更新，可以重新idea 来帮助其更新配置
     * 具体，可点击edit configuration 配置运行界面，的 Enviaorment variables 中查看
     * <p>
     * 页面信息越复杂，内容越多，转换时间越慢
     *
     * @param url     网址
     * @param pdfPath 路径
     */
    public static void WKhtmlTextToPdfBy(String url, String pdfPath) {
        TimeInterval timer = DateUtil.timer();
        if (StringUtils.isAnyBlank(url)) {
            throw new RrException("url参数或pdfpath参数缺少！");
        }
        String execStr = "cmd.exe /c wkhtmltopdf  %s  %s ";
        execStr = String.format(execStr, url, pdfPath);
        logger.info("execStr:{}", execStr);
        List<String> infos = RuntimeUtil.execForLines(execStr);
        infos.forEach(info -> logger.info("命令执行结果：{}", info));
        logger.info("转换耗时(毫秒)：{}", timer.interval());
    }
```

* 一些问题和解决方法

  参考:[Cannot run program "xxx": CreateProcess error=2](https://www.cnblogs.com/xjl456852/p/5839745.html)

  在代码执行命令行时，出现错误 提示 wkhtmltopdf 不是内部命令，外部命令。

  我以为是配置的全局变量没有生效，后来发现好像是因为win10的原因，使用 powershell 执行的命令。使用cmd 命令窗口就没有问题。 所以在执行命令行，加入

  ```shell
  # /c 是执行命令完成后关闭命令行窗口
  cmd.exe /c
  ```

  [Error: Could not save image](https://github.com/wkhtmltopdf/wkhtmltopdf/issues/2122) 如果出现这个错误，应该也是 powershell  导致的。

 

### Chrome  Headless

参考：[Headless Chrome 入门](https://zhuanlan.zhihu.com/p/29207391) 建议阅读

[利用Chrome Headless模式,网页转PDF](https://blog.csdn.net/xcl168/article/details/75675781)

介绍：

在 Chrome 59　中开始搭载 [Headless Chrome](https://link.zhihu.com/?target=https%3A//chromium.googlesource.com/chromium/src/%2B/lkgr/headless/README.md)。这是一种在无需显示*headless*的环境下运行 Chrome 浏览器的方式。从本质上来说，就是不用 chrome 浏览器来运行 Chrome 的功能！它将 Chromium 和 Blink 渲染引擎提供的所有现代 Web 平台的功能都带入了命令行。

无需显示*headless*的浏览器对于自动化测试和不需要可视化 UI 界面的服务器环境是一个很好的工具。例如，你可能需要对真实的网页运行一些测试，创建一个 PDF，或者只是检查浏览器如何呈现 URL。

（其实就是使用chrome浏览器，将网页转换为pdf。当你右键点打印，可以选择另存为PDF。这个工具应该更加适用于自动化测试（Selenium WebDriver  ChromeDriver））

使用方式：

1. 下载安装goole Chrome （Mac 和 Linux 上的 Chrome 59以上版本，windows 需要 60 版本以上）

2. 配置Chrome 为全局的环境变量，方便使用（自行百度）

3. 代码调用命令行，执行转换命令

   ```shell
   # 会输出到命令行的目录
   chrome --headless --disable-gpu --print-to-pdf https://www.chromestatus.com/
 
   # 指定目录输出pdf
   chrome --headless --disable-gpu --print-to-pdf="D://xx/x.pdf" https://www.chromestatus.com/
 
   # 参数解释
   headless 无头模式
   disable-gpu 禁用gpu
   ```

 

#### 代码实践：

环境：spring boot 2.2.7 jdk1.8  windows10

配置全局变量：系统-高级系统设置 将Chrome.exe 所在目录配置到 path 中

代码参考：[fun.gengzi.codecopy.business.utilstest.utils.HtmlToPdfUtils](https://github.com/gengzi/codecopy/tree/master/src/main/java/fun/gengzi/codecopy/business/utilstest/utils)

java调用cmd 执行命令，使用了工具类 hutool

```java
    /**
     * 使用Chrome  Headless 无头浏览器，完成对html转换pdf
     *
     * 需要安装 goole chrome 浏览器，版本在 60 以上，linux 在 59以上
     * 调用命令实现转换
     * @param url     网址
     * @param pdfPath 路径
     */
    public static void ChromhtmlTextToPdfBy(String url, String pdfPath) {
        TimeInterval timer = DateUtil.timer();
        if (StringUtils.isAnyBlank(url)) {
            throw new RrException("url参数或pdfpath参数缺少！");
        }
        // --window-size=800x1000
        // --virtual-time-budget=10000
        String execStr = "cmd.exe /c chrome --headless --run-all-compositor-stages-before-draw  --disable-gpu --print-to-pdf=\"%s\"  %s";
        execStr = String.format(execStr, pdfPath, url);
        logger.info("execStr:{}", execStr);
        List<String> infos = RuntimeUtil.execForLines(execStr);
        infos.forEach(info -> logger.info("命令执行结果：{}", info));
        logger.info("转换耗时(毫秒)：{}", timer.interval());
    }
```

 

## 汇总

* 针对访问url，转换pdf的。

  需要先访问url对应的页面，再渲染，再转换。如果访问url过慢，会严重影响转换速度。

* 对于复杂页面的转换，都会出现问题，内容丢失，或者叠加。

  所以尽量简化转换网页的复杂程度。不过一般也不会对特别复杂的网页转换。或者先将页面截屏，再转pdf。