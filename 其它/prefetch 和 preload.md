# 让你的网站加载更快 —— Prefetch 和 Preload 技术详解

prefetch 和 preload 是浏览器提供的两种资源预加载技术，它们可以预先请求浏览器可能需要的资源，并将这些资源缓存到本地，以便在页面需要时能够更快地获取，从而显著提高网站性能，优化用户体验。这些资源可以是文本文件、图像、音频或视频等各种类型的文件。

这里提到的 **缓存** 和 服务器设置的资源缓存不一样，比如 expires、cache-control 等，这里提到的缓存只是提前加载资源缓存到本地，

这里提到的 **缓存** 只是通过预加载技术将资源提前缓存到本地，**只是将资源提前加载回来了**，至于具体的缓存策略还是有服务器决定的，比如 nginx 设置 expires 或 cache control。

prefetch 和 preload 之间的主要区别在于: 

* prefetch 利用浏览器的空闲时间，预加载将来可能会被用户访问到的资源，由于是利用浏览器的空闲时间，所以它不会影响当前页的加载性能，当然也不保证预加载的资源一定会被提前缓存，假如浏览器一直很忙
* preload 用于预加载即将被使用的资源，被标记为 preload 的资源会被优先加载，也就是说它会保证预加载的资源在使用前一定会被提前缓存到本地，所以，如果使用不当，它会影响当前页的加载性能

## prefetch

prefetch 可以帮助浏览器在页面加载之前预取用户可能需要的资源，以加快网站的整体加载速度。这些资源包括但不限于图像、脚本和样式表。

prefetch 可以使用 HTM L的 <link> 标签实现。例如，下面的代码将预取一个名为 “example.js” 的脚本文件：

```html
<link rel="prefetch" href="example.js" as="script">
```

prefetch 还可以通过 HTTP 头来实现。例如，下面的代码将使用HTTP头预取一个名为 “example.js” 的脚本文件：

```html
Link: <example.js>; rel="prefetch"
```

当浏览器遇到这些标签 或 HTTP 头时，它会预取指定的资源，并将它们存储在本地缓存中。这样，当用户浏览网站时，这些资源将能够更快地加载。

值得注意的是，prefetch 并不保证资源的加载顺序或加载时间，也不保证在需要使用之前资源一定会被缓存，因为 prefetch 是在浏览器空闲时间工作，所以如果浏览器一直忙，prefetch 的资源就没机会被加载。

## preload

preload 是一种更为复杂的资源预加载技术，它可以在页面加载时预取即将被使用的资源，以加快页面的渲染速度。这些资源包括但不限于图像、脚本和样式表。

preload 可以使用 HTML 的 <link> 标签实现。例如，下面的代码将预取一个名为 “example.css” 的样式表文件：

```html
<link rel="preload" href="example.css" as="style">
```

preload 还可以通过 HTTP 头来实现。例如，下面的代码将使用HTTP头预取一个名为 “example.js” 的脚本文件：

```html
Link: <example.js>; rel="preload"; as="script"
```

与 prefetch 不同，preload 可以确保资源的加载顺序和时间，并且这些资源在使用前一定会被缓存。但是，preload 也需要谨慎使用，因为标有 preload 的资源会被优先加载，因此它可能会影响页面的加载性能。如果您的网站中有大量资源需要预加载，可能会影响页面的渲染速度。

## as 属性

在使用 <link rel="preload"> 标签时，as 属性用于指定预加载资源的类型。它告诉浏览器如何处理预加载的资源，并在加载过程中进行优化。以下是一些常见的as属性值：

* as="script"，预加载JavaScript文件
* as="style"，预加载CSS文件
* as="font"，预加载字体文件
* as="image"，预加载图片文件
* as="audio"，预加载音频文件
* as="video"，预加载视频文件
* as="fetch"，预加载数据文件（例如JSON、XML等）。

使用正确的 as 属性可以帮助浏览器更好地优化预加载的资源，并在加载过程中提高性能。例如，如果您预加载的是 CSS 文件，则应将as属性设置为 "style"，这将使浏览器在预加载 CSS 文件时执行一些优化，例如提前解析样式并缓存它们。同样，如果您预加载的是字体文件，则应将 as 属性设置为 "font"，这将使浏览器在预加载字体文件时执行一些优化，例如提前解码字体并进行缓存。

需要注意的是，使用<link> 标签进行预加载和预取时，必须指定 as 属性来告知浏览器需要预加载或预取的资源类型。如果不指定 as 属性，浏览器将根据文件扩展名来猜测资源类型，这可能会导致预加载和预取失败，所以为了获得最佳的性能和预加载效果，建议始终使用正确的 as 属性。

> 经过实际测试，发现 preload 不使用 as 属性，观看 network 面板中资源的加载顺序，看起来 preload 像失效了，而且有时候浏览器的 console 会给出告警。

## 实战

下面我们将通过一个示例来演示 prefetch 和 preload 的相关知识点：

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <style>
    * {
      padding: 0;
      margin: 0;
    }
    body {
      display: flex;
    }
    img {
      width: 500px;
      height: 500px;
    }
  </style>
  <link rel="prefetch" href="https://p5.ssl.qhimg.com/d/inn/a382649bc56a/russian-girl.png" as="image" >
  <link rel="preload" href="https://p4.ssl.qhimg.com/d/inn/19dd63a16a5a/dog.png" as="image" >
</head>
<body style="display: flex;">
  <img src="https://p3.ssl.qhimg.com/d/inn/ab18573862d6/cat.png">
  <img src="https://p4.ssl.qhimg.com/d/inn/19dd63a16a5a/dog.png">
</body>
</html>
```

代码中有三张图片，这三张图片在代码中由上而下分别是：

* prefetch 的 russian-girl
* preload 的 dog
* img 标签的 cat
* img 标签的 dog。

再看下图中三张照片的加载顺序：

* 代码中最上面的 prefetch russian-girl 反而是最后被加载，并且在另外两张图片加载就绪之前始终处于 pending 状态，等另外两张图片加载完成后才加载，表明优先级最低，并且不会占用页面资源
* 处于最后的 img dog 反而是最先被加载的，然后 img cat 次之，因为 dog.png 通过 preload 做了预加载，表明 preload 的资源会优先被加载

**备注:** 为了观看加载效果，所以故意把网络调成了 fast 3G，所以图片加载时间比较长。

<img width="1536" alt="image" src="https://user-images.githubusercontent.com/26913352/232323297-43b87e61-d5b3-4d41-ac18-f8e388a5f2a0.png">

## 总结

prefetch 和 preload 是两种非常有用的资源预加载技术，可以显著提高网站性能并优化用户体验。使用 prefetch 可以帮助浏览器预取将来可能会被用户访问到的资源，而使用 preload 可以预加载即将被使用的资源。在使用这些技术时，我们需要注意谨慎使用，确保只预加载可能会被用户使用的资源，从而并避免过度预加载导致性能问题。

