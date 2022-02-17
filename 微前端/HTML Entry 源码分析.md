# HTML Entry 源码分析

**当学习成为了习惯，知识也就变成了常识。**感谢各位的 **点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

## 封面

![202202031927948](https://gitee.com/liyongning/typora-image-bed/raw/master/202202051816571.jpg)

## 简介

从 HTML Entry 的诞生原因 -> 原理简述 -> 实际应用 -> 源码分析，带你全方位刨析 HTML Entry 框架。

## 序言

`HTML Entry` 这个词大家可能比较陌生，毕竟在 `google` 上搜 `HTML Entry 是什么 ?` 都搜索不到正确的结果。但如果你了解微前端的话，可能就会有一些了解。

## 致读者

本着不浪费大家时间的原则，特此说明，如果你能读懂  **HTML Entry 是什么？？** 部分，则可继续往下阅读，如果看不懂建议阅读完推荐资料再回来阅读

## JS Entry 有什么问题

说到 `HTML Entry` 就不得不提另外一个词 `JS Entry`，因为 `HTML Entry` 就是来解决 `JS Entry` 所面临的问题的。

微前端领域最著名的两大框架分别是 `single-spa` 和 `qiankun`，后者是基于前者做了二次封装，并解决了前者的一些问题。

`single-spa` 就做了两件事情：

* 加载微应用（加载方法还得用户自己来实现）
* 管理微应用的状态（初始化、挂载、卸载）

而 `JS Entry` 的理念就在加载微应用的时候用到了，在使用 `single-spa` 加载微应用时，我们加载的不是微应用本身，而是微应用导出的 `JS` 文件，而在入口文件中会导出一个对象，这个对象上有 `bootstrap`、`mount`、`unmount` 这三个接入 `single-spa` 框架必须提供的生命周期方法，其中 `mount` 方法规定了微应用应该怎么挂载到主应用提供的容器节点上，当然你要接入一个微应用，就需要对微应用进行一系列的改造，然而 `JS Entry` 的问题就出在这儿，改造时对微应用的侵入行太强，而且和主应用的耦合性太强。

`single-spa` 采用 `JS Entry` 的方式接入微应用。微应用改造一般分为三步：

* 微应用路由改造，添加一个特定的前缀
* 微应用入口改造，挂载点变更和生命周期函数导出
* 打包工具配置更改

侵入型强其实说的就是第三点，更改打包工具的配置，使用 `single-spa` 接入微应用需要将微应用整个打包成一个 `JS` 文件，发布到静态资源服务器，然后在主应用中配置该 `JS` 文件的地址告诉 `single-spa` 去这个地址加载微应用。

不说其它的，就现在这个改动就存在很大的问题，将整个微应用打包成一个 `JS` 文件，常见的打包优化基本上都没了，比如：按需加载、首屏资源加载优化、css 独立打包等优化措施。

> **注意**：子应用也可以将包打成多个，然后利用 webpack 的 webpack-manifest-plugin 插件打包出 manifest.json 文件，生成一份资源清单，然后主应用的 loadApp 远程读取每个子应用的清单文件，依次加载文件里面的资源；不过该方案也没办法享受子应用的按需加载能力

项目发布以后出现了 `bug` ，修复之后需要更新上线，为了清除浏览器缓存带来的应用，一般文件名会带上 `chunkcontent`，微应用发布之后文件名都会发生变化，这时候还需要更新主应用中微应用配置，然后重新编译主应用然后发布，这套操作简直是不能忍受的，这也是 [微前端框架 之 single-spa 从入门到精通](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247484245&idx=1&sn=9ee91018578e6189f3b11a4d688228c5&chksm=9f696021a81ee937847c962e3135017fff9ba8fd0b61f782d7245df98582a1410aa000dc5fdc&token=165646905&lang=zh_CN#rd) 这篇文章中示例项目中微应用发布时的环境配置选择 `development` 的原因。

`qiankun` 框架为了解决 `JS Entry` 的问题，于是采用了 `HTML Entry` 的方式，让用户接入微应用就像使用 `iframe` 一样简单。

如果以上内容没有看懂，则说明这篇文章不太适合你阅读，建议阅读 [微前端框架 之 single-spa 从入门到精通](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247484245&idx=1&sn=9ee91018578e6189f3b11a4d688228c5&chksm=9f696021a81ee937847c962e3135017fff9ba8fd0b61f782d7245df98582a1410aa000dc5fdc&token=165646905&lang=zh_CN#rd)，这篇文章详细讲述了 `single-spa` 的基础使用和源码原理，阅读完以后再回来读这篇文章会有事半功倍的效果，请读者切勿强行阅读，否则可能出现头昏脑胀的现象。

## HTML Entry

`HTML Entry` 是由 `import-html-entry` 库实现的，通过 `http` 请求加载指定地址的首屏内容即 `html` 页面，然后解析这个 `html` 模版得到 `template`, ` scripts` , `entry`, `styles`

```javscript
{
  template: 经过处理的脚本，link、script 标签都被注释掉了,
  scripts: [脚本的http地址 或者 { async: true, src: xx } 或者 代码块],
  styles: [样式的http地址],
 	entry: 入口脚本的地址，要不是标有 entry 的 script 的 src，要不就是最后一个 script 标签的 src
}
```

然后远程加载 `styles` 中的样式内容，将 `template` 模版中注释掉的 `link` 标签替换为相应的 `style` 元素。

然后向外暴露一个 `Promise` 对象

```javascript
{
  // template 是 link 替换为 style 后的 template
	template: embedHTML,
	// 静态资源地址
	assetPublicPath,
	// 获取外部脚本，最终得到所有脚本的代码内容
	getExternalScripts: () => getExternalScripts(scripts, fetch),
	// 获取外部样式文件的内容
	getExternalStyleSheets: () => getExternalStyleSheets(styles, fetch),
	// 脚本执行器，让 JS 代码(scripts)在指定 上下文 中运行
	execScripts: (proxy, strictGlobal) => {
		if (!scripts.length) {
			return Promise.resolve();
		}
		return execScripts(entry, scripts, proxy, { fetch, strictGlobal });
	}
}
```

这就是 `HTML Entry` 的原理，更详细的内容可继续阅读下面的源码分析部分

## 实际应用

`qiankun` 框架为了解决 `JS Entry` 的问题，就采用了 `HTML Entry` 的方式，让用户接入微应用就像使用 `iframe` 一样简单。

通过上面的阅读知道了 `HTML Entry` 最终会返回一个 `Promise` 对象，`qiankun` 就用了这个对象中的 `template`、`assetPublicPath` 和 `execScripts` 三项，将 `template` 通过 `DOM` 操作添加到主应用中，执行 `execScripts` 方法得到微应用导出的生命周期方法，并且还顺便解决了 `JS` 全局污染的问题，因为执行 `execScripts` 方法的时候可以通过 `proxy` 参数指定 `JS` 的执行上下文。

更加具体的内容可阅读 [微前端框架 之 qiankun 从入门到源码分析](https://mp.weixin.qq.com/s?__biz=MzA3NTk4NjQ1OQ==&mid=2247484411&idx=1&sn=7e67d2843b8576fce01b18269f33f7e9&chksm=9f69608fa81ee99954b6b5a1e3eb40e194c05c1edb504baac27577a0217f61c78ff9d0bb7e23&token=165646905&lang=zh_CN#rd)

## HTML Entry 源码分析

### importEntry

```javascript
/**
 * 加载指定地址的首屏内容
 * @param {*} entry 可以是一个字符串格式的地址，比如 localhost:8080，也可以是一个配置对象，比如 { scripts, styles, html }
 * @param {*} opts
 * return importHTML 的执行结果
 */
export function importEntry(entry, opts = {}) {
	// 从 opt 参数中解析出 fetch 方法 和 getTemplate 方法，没有就用默认的
	const { fetch = defaultFetch, getTemplate = defaultGetTemplate } = opts;
	// 获取静态资源地址的一个方法
	const getPublicPath = opts.getPublicPath || opts.getDomain || defaultGetPublicPath;

	if (!entry) {
		throw new SyntaxError('entry should not be empty!');
	}

	// html entry，entry 是一个字符串格式的地址
	if (typeof entry === 'string') {
		return importHTML(entry, { fetch, getPublicPath, getTemplate });
	}

	// config entry，entry 是一个对象 = { scripts, styles, html }
	if (Array.isArray(entry.scripts) || Array.isArray(entry.styles)) {

		const { scripts = [], styles = [], html = '' } = entry;
		const setStylePlaceholder2HTML = tpl => styles.reduceRight((html, styleSrc) => `${genLinkReplaceSymbol(styleSrc)}${html}`, tpl);
		const setScriptPlaceholder2HTML = tpl => scripts.reduce((html, scriptSrc) => `${html}${genScriptReplaceSymbol(scriptSrc)}`, tpl);

		return getEmbedHTML(getTemplate(setScriptPlaceholder2HTML(setStylePlaceholder2HTML(html))), styles, { fetch }).then(embedHTML => ({
			template: embedHTML,
			assetPublicPath: getPublicPath(entry),
			getExternalScripts: () => getExternalScripts(scripts, fetch),
			getExternalStyleSheets: () => getExternalStyleSheets(styles, fetch),
			execScripts: (proxy, strictGlobal) => {
				if (!scripts.length) {
					return Promise.resolve();
				}
				return execScripts(scripts[scripts.length - 1], scripts, proxy, { fetch, strictGlobal });
			},
		}));

	} else {
		throw new SyntaxError('entry scripts or styles should be array!');
	}
}

```

### importHTML

```javascript
/**
 * 加载指定地址的首屏内容
 * @param {*} url 
 * @param {*} opts 
 * return Promise<{
  	// template 是 link 替换为 style 后的 template
		template: embedHTML,
		// 静态资源地址
		assetPublicPath,
		// 获取外部脚本，最终得到所有脚本的代码内容
		getExternalScripts: () => getExternalScripts(scripts, fetch),
		// 获取外部样式文件的内容
		getExternalStyleSheets: () => getExternalStyleSheets(styles, fetch),
		// 脚本执行器，让 JS 代码(scripts)在指定 上下文 中运行
		execScripts: (proxy, strictGlobal) => {
			if (!scripts.length) {
				return Promise.resolve();
			}
			return execScripts(entry, scripts, proxy, { fetch, strictGlobal });
		},
   }>
 */
export default function importHTML(url, opts = {}) {
	// 三个默认的方法
	let fetch = defaultFetch;
	let getPublicPath = defaultGetPublicPath;
	let getTemplate = defaultGetTemplate;

	if (typeof opts === 'function') {
		// if 分支，兼容遗留的 importHTML api，ops 可以直接是一个 fetch 方法
		fetch = opts;
	} else {
		// 用用户传递的参数（如果提供了的话）覆盖默认方法
		fetch = opts.fetch || defaultFetch;
		getPublicPath = opts.getPublicPath || opts.getDomain || defaultGetPublicPath;
		getTemplate = opts.getTemplate || defaultGetTemplate;
	}

	// 通过 fetch 方法请求 url，这也就是 qiankun 为什么要求你的微应用要支持跨域的原因
	return embedHTMLCache[url] || (embedHTMLCache[url] = fetch(url)
		// response.text() 是一个 html 模版
		.then(response => response.text())
		.then(html => {

			// 获取静态资源地址
			const assetPublicPath = getPublicPath(url);
			/**
 	     * 从 html 模版中解析出外部脚本的地址或者内联脚本的代码块 和 link 标签的地址
			 * {
 			 * 	template: 经过处理的脚本，link、script 标签都被注释掉了,
       * 	scripts: [脚本的http地址 或者 { async: true, src: xx } 或者 代码块],
       *  styles: [样式的http地址],
 	     * 	entry: 入口脚本的地址，要不是标有 entry 的 script 的 src，要不就是最后一个 script 标签的 src
 			 * }
			 */
			const { template, scripts, entry, styles } = processTpl(getTemplate(html), assetPublicPath);

			// getEmbedHTML 方法通过 fetch 远程加载所有的外部样式，然后将对应的 link 注释标签替换为 style，即外部样式替换为内联样式，然后返回 embedHTML，即处理过后的 HTML 模版
			return getEmbedHTML(template, styles, { fetch }).then(embedHTML => ({
				// template 是 link 替换为 style 后的 template
				template: embedHTML,
				// 静态资源地址
				assetPublicPath,
				// 获取外部脚本，最终得到所有脚本的代码内容
				getExternalScripts: () => getExternalScripts(scripts, fetch),
				// 获取外部样式文件的内容
				getExternalStyleSheets: () => getExternalStyleSheets(styles, fetch),
				// 脚本执行器，让 JS 代码(scripts)在指定 上下文 中运行
				execScripts: (proxy, strictGlobal) => {
					if (!scripts.length) {
						return Promise.resolve();
					}
					return execScripts(entry, scripts, proxy, { fetch, strictGlobal });
				},
			}));
		}));
}

```

### processTpl

```javascript
/**
 * 从 html 模版中解析出外部脚本的地址或者内联脚本的代码块 和 link 标签的地址
 * @param tpl html 模版
 * @param baseURI
 * @stripStyles whether to strip the css links
 * @returns {{template: void | string | *, scripts: *[], entry: *}}
 * return {
 * 	template: 经过处理的脚本，link、script 标签都被注释掉了,
 * 	scripts: [脚本的http地址 或者 { async: true, src: xx } 或者 代码块],
 *  styles: [样式的http地址],
 * 	entry: 入口脚本的地址，要不是标有 entry 的 script 的 src，要不就是最后一个 script 标签的 src
 * }
 */
export default function processTpl(tpl, baseURI) {

	let scripts = [];
	const styles = [];
	let entry = null;
	// 判断浏览器是否支持 es module，<script type = "module" />
	const moduleSupport = isModuleScriptSupported();

	const template = tpl

		// 移除 html 模版中的注释内容 <!-- xx -->
		.replace(HTML_COMMENT_REGEX, '')

		// 匹配 link 标签
		.replace(LINK_TAG_REGEX, match => {
			/**
			 * 将模版中的 link 标签变成注释，如果有存在 href 属性且非预加载的 link，则将地址存到 styles 数组，如果是预加载的 link 直接变成注释
			 */
			// <link rel = "stylesheet" />
			const styleType = !!match.match(STYLE_TYPE_REGEX);
			if (styleType) {

				// <link rel = "stylesheet" href = "xxx" />
				const styleHref = match.match(STYLE_HREF_REGEX);
				// <link rel = "stylesheet" ignore />
				const styleIgnore = match.match(LINK_IGNORE_REGEX);

				if (styleHref) {

					// 获取 href 属性值
					const href = styleHref && styleHref[2];
					let newHref = href;

					// 如果 href 没有协议说明给的是一个相对地址，拼接 baseURI 得到完整地址
					if (href && !hasProtocol(href)) {
						newHref = getEntirePath(href, baseURI);
					}
					// 将 <link rel = "stylesheet" ignore /> 变成 <!-- ignore asset ${url} replaced by import-html-entry -->
					if (styleIgnore) {
						return genIgnoreAssetReplaceSymbol(newHref);
					}

					// 将 href 属性值存入 styles 数组
					styles.push(newHref);
					// <link rel = "stylesheet" href = "xxx" /> 变成 <!-- link ${linkHref} replaced by import-html-entry -->
					return genLinkReplaceSymbol(newHref);
				}
			}

			// 匹配 <link rel = "preload or prefetch" href = "xxx" />，表示预加载资源
			const preloadOrPrefetchType = match.match(LINK_PRELOAD_OR_PREFETCH_REGEX) && match.match(LINK_HREF_REGEX) && !match.match(LINK_AS_FONT);
			if (preloadOrPrefetchType) {
				// 得到 href 地址
				const [, , linkHref] = match.match(LINK_HREF_REGEX);
				// 将标签变成 <!-- prefetch/preload link ${linkHref} replaced by import-html-entry -->
				return genLinkReplaceSymbol(linkHref, true);
			}

			return match;
		})
		// 匹配 <style></style>
		.replace(STYLE_TAG_REGEX, match => {
			if (STYLE_IGNORE_REGEX.test(match)) {
				// <style ignore></style> 变成 <!-- ignore asset style file replaced by import-html-entry -->
				return genIgnoreAssetReplaceSymbol('style file');
			}
			return match;
		})
		// 匹配 <script></script>
		.replace(ALL_SCRIPT_REGEX, (match, scriptTag) => {
			// 匹配 <script ignore></script>
			const scriptIgnore = scriptTag.match(SCRIPT_IGNORE_REGEX);
			// 匹配 <script nomodule></script> 或者 <script type = "module"></script>，都属于应该被忽略的脚本
			const moduleScriptIgnore =
				(moduleSupport && !!scriptTag.match(SCRIPT_NO_MODULE_REGEX)) ||
				(!moduleSupport && !!scriptTag.match(SCRIPT_MODULE_REGEX));
			// in order to keep the exec order of all javascripts

			// <script type = "xx" />
			const matchedScriptTypeMatch = scriptTag.match(SCRIPT_TYPE_REGEX);
			// 获取 type 属性值
			const matchedScriptType = matchedScriptTypeMatch && matchedScriptTypeMatch[2];
			// 验证 type 是否有效，type 为空 或者 'text/javascript', 'module', 'application/javascript', 'text/ecmascript', 'application/ecmascript'，都视为有效
			if (!isValidJavaScriptType(matchedScriptType)) {
				return match;
			}

			// if it is a external script，匹配非 <script type = "text/ng-template" src = "xxx"></script>
			if (SCRIPT_TAG_REGEX.test(match) && scriptTag.match(SCRIPT_SRC_REGEX)) {
				/*
				collect scripts and replace the ref
				*/

				// <script entry />
				const matchedScriptEntry = scriptTag.match(SCRIPT_ENTRY_REGEX);
				// <script src = "xx" />
				const matchedScriptSrcMatch = scriptTag.match(SCRIPT_SRC_REGEX);
				// 脚本地址
				let matchedScriptSrc = matchedScriptSrcMatch && matchedScriptSrcMatch[2];

				if (entry && matchedScriptEntry) {
					// 说明出现了两个入口地址，即两个 <script entry src = "xx" />
					throw new SyntaxError('You should not set multiply entry script!');
				} else {
					// 补全脚本地址，地址如果没有协议，说明是一个相对路径，添加 baseURI
					if (matchedScriptSrc && !hasProtocol(matchedScriptSrc)) {
						matchedScriptSrc = getEntirePath(matchedScriptSrc, baseURI);
					}

					// 脚本的入口地址
					entry = entry || matchedScriptEntry && matchedScriptSrc;
				}

				if (scriptIgnore) {
					// <script ignore></script> 替换为 <!-- ignore asset ${url || 'file'} replaced by import-html-entry -->
					return genIgnoreAssetReplaceSymbol(matchedScriptSrc || 'js file');
				}

				if (moduleScriptIgnore) {
					// <script nomodule></script> 或者 <script type = "module"></script> 替换为
					// <!-- nomodule script ${scriptSrc} ignored by import-html-entry --> 或
					// <!-- module script ${scriptSrc} ignored by import-html-entry -->
					return genModuleScriptReplaceSymbol(matchedScriptSrc || 'js file', moduleSupport);
				}

				if (matchedScriptSrc) {
					// 匹配 <script src = 'xx' async />，说明是异步加载的脚本
					const asyncScript = !!scriptTag.match(SCRIPT_ASYNC_REGEX);
					// 将脚本地址存入 scripts 数组，如果是异步加载，则存入一个对象 { async: true, src: xx }
					scripts.push(asyncScript ? { async: true, src: matchedScriptSrc } : matchedScriptSrc);
					// <script src = "xx" async /> 或者 <script src = "xx" /> 替换为 
					// <!-- async script ${scriptSrc} replaced by import-html-entry --> 或 
					// <!-- script ${scriptSrc} replaced by import-html-entry -->
					return genScriptReplaceSymbol(matchedScriptSrc, asyncScript);
				}

				return match;
			} else {
				// 说明是内部脚本，<script>xx</script>
				if (scriptIgnore) {
					// <script ignore /> 替换为 <!-- ignore asset js file replaced by import-html-entry -->
					return genIgnoreAssetReplaceSymbol('js file');
				}

				if (moduleScriptIgnore) {
					// <script nomodule></script> 或者 <script type = "module"></script> 替换为
					// <!-- nomodule script ${scriptSrc} ignored by import-html-entry --> 或 
					// <!-- module script ${scriptSrc} ignored by import-html-entry -->
					return genModuleScriptReplaceSymbol('js file', moduleSupport);
				}

				// if it is an inline script，<script>xx</script>，得到标签之间的代码 => xx
				const code = getInlineCode(match);

				// remove script blocks when all of these lines are comments. 判断代码块是否全是注释
				const isPureCommentBlock = code.split(/[\r\n]+/).every(line => !line.trim() || line.trim().startsWith('//'));

				if (!isPureCommentBlock) {
					// 不是注释，则将代码块存入 scripts 数组
					scripts.push(match);
				}

				// <script>xx</script> 替换为 <!-- inline scripts replaced by import-html-entry -->
				return inlineScriptReplaceSymbol;
			}
		});

	// filter empty script
	scripts = scripts.filter(function (script) {
		return !!script;
	});

	return {
		template,
		scripts,
		styles,
		// set the last script as entry if have not set
		entry: entry || scripts[scripts.length - 1],
	};
}

```

### getEmbedHTML

```javascript
/**
 * convert external css link to inline style for performance optimization，外部样式转换成内联样式
 * @param template，html 模版
 * @param styles link 样式链接
 * @param opts = { fetch }
 * @return embedHTML 处理过后的 html 模版
 */
function getEmbedHTML(template, styles, opts = {}) {
	const { fetch = defaultFetch } = opts;
	let embedHTML = template;

	return getExternalStyleSheets(styles, fetch)
		.then(styleSheets => {
			// 通过循环，将之前设置的 link 注释标签替换为 style 标签，即 <style>/* href地址 */ xx </style>
			embedHTML = styles.reduce((html, styleSrc, i) => {
				html = html.replace(genLinkReplaceSymbol(styleSrc), `<style>/* ${styleSrc} */${styleSheets[i]}</style>`);
				return html;
			}, embedHTML);
			return embedHTML;
		});
}

```

### getExternalScripts

```javascript
/**
 * 加载脚本，最终返回脚本的内容，Promise<Array>，每个元素都是一段 JS 代码
 * @param {*} scripts = [脚本http地址 or 内联脚本的脚本内容 or { async: true, src: xx }]
 * @param {*} fetch 
 * @param {*} errorCallback 
 */
export function getExternalScripts(scripts, fetch = defaultFetch, errorCallback = () => {
}) {

	// 定义一个可以加载远程指定 url 脚本的方法，当然里面也做了缓存，如果命中缓存直接从缓存中获取
	const fetchScript = scriptUrl => scriptCache[scriptUrl] ||
		(scriptCache[scriptUrl] = fetch(scriptUrl).then(response => {
			// usually browser treats 4xx and 5xx response of script loading as an error and will fire a script error event
			// https://stackoverflow.com/questions/5625420/what-http-headers-responses-trigger-the-onerror-handler-on-a-script-tag/5625603
			if (response.status >= 400) {
				errorCallback();
				throw new Error(`${scriptUrl} load failed with status ${response.status}`);
			}

			return response.text();
		}));

	return Promise.all(scripts.map(script => {

			if (typeof script === 'string') {
				// 字符串，要不是链接地址，要不是脚本内容（代码）
				if (isInlineCode(script)) {
					// if it is inline script
					return getInlineCode(script);
				} else {
					// external script，加载脚本
					return fetchScript(script);
				}
			} else {
				// use idle time to load async script
				// 异步脚本，通过 requestIdleCallback 方法加载
				const { src, async } = script;
				if (async) {
					return {
						src,
						async: true,
						content: new Promise((resolve, reject) => requestIdleCallback(() => fetchScript(src).then(resolve, reject))),
					};
				}

				return fetchScript(src);
			}
		},
	));
}

```

### getExternalStyleSheets

```javascript
/**
 * 通过 fetch 方法加载指定地址的样式文件
 * @param {*} styles = [ href ]
 * @param {*} fetch 
 * return Promise<Array>，每个元素都是一堆样式内容
 */
export function getExternalStyleSheets(styles, fetch = defaultFetch) {
	return Promise.all(styles.map(styleLink => {
			if (isInlineCode(styleLink)) {
				// if it is inline style
				return getInlineCode(styleLink);
			} else {
				// external styles，加载样式并缓存
				return styleCache[styleLink] ||
					(styleCache[styleLink] = fetch(styleLink).then(response => response.text()));
			}

		},
	));
}

```

### execScripts

```javascript
/**
 * FIXME to consistent with browser behavior, we should only provide callback way to invoke success and error event
 * 脚本执行器，让指定的脚本(scripts)在规定的上下文环境中执行
 * @param entry 入口地址
 * @param scripts = [脚本http地址 or 内联脚本的脚本内容 or { async: true, src: xx }] 
 * @param proxy 脚本执行上下文，全局对象，qiankun JS 沙箱生成 windowProxy 就是传递到了这个参数
 * @param opts
 * @returns {Promise<unknown>}
 */
export function execScripts(entry, scripts, proxy = window, opts = {}) {
	const {
		fetch = defaultFetch, strictGlobal = false, success, error = () => {
		}, beforeExec = () => {
		},
	} = opts;

	// 获取指定的所有外部脚本的内容，并设置每个脚本的执行上下文，然后通过 eval 函数运行
	return getExternalScripts(scripts, fetch, error)
		.then(scriptsText => {
			// scriptsText 为脚本内容数组 => 每个元素是一段 JS 代码
			const geval = (code) => {
				beforeExec();
				(0, eval)(code);
			};

			/**
			 * 
			 * @param {*} scriptSrc 脚本地址
			 * @param {*} inlineScript 脚本内容
			 * @param {*} resolve 
			 */
			function exec(scriptSrc, inlineScript, resolve) {

				// 性能度量
				const markName = `Evaluating script ${scriptSrc}`;
				const measureName = `Evaluating Time Consuming: ${scriptSrc}`;

				if (process.env.NODE_ENV === 'development' && supportsUserTiming) {
					performance.mark(markName);
				}

				if (scriptSrc === entry) {
					// 入口
					noteGlobalProps(strictGlobal ? proxy : window);

					try {
						// bind window.proxy to change `this` reference in script
						geval(getExecutableScript(scriptSrc, inlineScript, proxy, strictGlobal));
						const exports = proxy[getGlobalProp(strictGlobal ? proxy : window)] || {};
						resolve(exports);
					} catch (e) {
						// entry error must be thrown to make the promise settled
						console.error(`[import-html-entry]: error occurs while executing entry script ${scriptSrc}`);
						throw e;
					}
				} else {
					if (typeof inlineScript === 'string') {
						try {
							// bind window.proxy to change `this` reference in script，就是设置 JS 代码的执行上下文，然后通过 eval 函数运行运行代码
							geval(getExecutableScript(scriptSrc, inlineScript, proxy, strictGlobal));
						} catch (e) {
							// consistent with browser behavior, any independent script evaluation error should not block the others
							throwNonBlockingError(e, `[import-html-entry]: error occurs while executing normal script ${scriptSrc}`);
						}
					} else {
						// external script marked with async，异步加载的代码，下载完以后运行
						inlineScript.async && inlineScript?.content
							.then(downloadedScriptText => geval(getExecutableScript(inlineScript.src, downloadedScriptText, proxy, strictGlobal)))
							.catch(e => {
								throwNonBlockingError(e, `[import-html-entry]: error occurs while executing async script ${inlineScript.src}`);
							});
					}
				}

				// 性能度量
				if (process.env.NODE_ENV === 'development' && supportsUserTiming) {
					performance.measure(measureName, markName);
					performance.clearMarks(markName);
					performance.clearMeasures(measureName);
				}
			}

			/**
			 * 递归
			 * @param {*} i 表示第几个脚本
			 * @param {*} resolvePromise 成功回调 
			 */
			function schedule(i, resolvePromise) {

				if (i < scripts.length) {
					// 第 i 个脚本的地址
					const scriptSrc = scripts[i];
					// 第 i 个脚本的内容
					const inlineScript = scriptsText[i];

					exec(scriptSrc, inlineScript, resolvePromise);
					if (!entry && i === scripts.length - 1) {
						// resolve the promise while the last script executed and entry not provided
						resolvePromise();
					} else {
						// 递归调用下一个脚本
						schedule(i + 1, resolvePromise);
					}
				}
			}

			// 从第 0 个脚本开始调度
			return new Promise(resolve => schedule(0, success || resolve));
		});
}

```

## 结语

以上就是 `HTML Entry` 的全部内容，也是深入理解 `微前端`、`single-spa`、`qiankun` 不可或缺的一部分，源码在 [github](https://github.com/liyongning/import-html-entry)

阅读到这里如果你想继续深入理解 `微前端`、`single-spa`、`qiankun` 等，推荐阅读如下内容

* [微前端专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA3NTk4NjQ1OQ==&action=getalbum&album_id=2251416802327232513&scene=173&from_msgid=2247484411&from_itemidx=1&count=3&nolastread=1#wechat_redirect)

  * 微前端框架 之 single-spa 从入门到精通

  * 微前端框架 之 qiankun 从入门到源码分析

  * qiankun 2.x 运行时沙箱 源码分析


* [single-spa 官网](https://single-spa.js.org/docs/getting-started-overview)
* [qiankun 官网](https://qiankun.umijs.org/zh/guide)



感谢各位的：**点赞**、**收藏**和**评论**，我们下期见。

---

**当学习成为了习惯，知识也就变成了常识。**感谢各位的 **点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。