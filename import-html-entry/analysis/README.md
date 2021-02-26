# import-html-entry

## 背景

之所以会阅读这个源码，完全是因为笔者最近在进行微前端实践，所以研究了 [single-spa](https://github.com/single-spa/single-spa) 和 [qiankun](https://github.com/umijs/qiankun) 这两个微前端框架。

`single-spa` 就是采用 `JS Entry` 的方式来加载微应用，也就是说需要将每个应用打包成一个 `JS` 文件发布后，然后在主应用中配置改文件地址来加载应用。这种方式的痛点十分明显：
- 整体打包成一个 `JS` ，那按需加载、切片等优化方案基本都不可用
- 为了避免缓存，现在一般文件都会生成 `hashcontent`，这就意味着每次应用更新，都需要重新更新主应用配置并且编译发布，这更新成本太高了

`qiankun` 则使用 `HTML Entry` 来解决这个问题，也就是通过解析 `HTML` 的内容，来动态加载所需的相关资源（样式和脚本）。所以，本文要分析的源码也就是这个解析的过程。

## 源码分析

我的源码阅读习惯是从测试用例入口去看，为了更好的阅读体验，我会把参数校验等非核心实现的逻辑删去。然后整个分析过程会以先大体逻辑，再细看各方法实现。

### test-config-entry.js

先看来第一个测试文件，这个里面有两个 `case`，看输入和预期可以知道，这个是验证 `importEntry` 方法的，该方法的作用返回字符串模版。

#### importEntry

先来看大体逻辑，然后再细看方法。

```javascript
export function importEntry(entry, opts = {}) {
	// 获取配置相关
	const { fetch = defaultFetch, getTemplate = defaultGetTemplate } = opts;
	const getPublicPath = opts.getPublicPath || opts.getDomain || defaultGetPublicPath;

	if (!entry) {
		throw new SyntaxError('entry should not be empty!');
	}
	// 如果 enrty 是字符串，通过 importHTML 获取输出结果
	// html entry
	if (typeof entry === 'string') {
		return importHTML(entry, {
			fetch,
			getPublicPath,
			getTemplate,
		});
	}
	// 如果指定了要加载的脚本或样式资源
	// config entry
	if (Array.isArray(entry.scripts) || Array.isArray(entry.styles)) {

		const { scripts = [], styles = [], html = '' } = entry;
		// 拼接 html 和 styles
		const setStylePlaceholder2HTML = tpl => styles.reduceRight((html, styleSrc) => `${genLinkReplaceSymbol(styleSrc)}${html}`, tpl);
		// 再拼接 scripts
		const setScriptPlaceholder2HTML = tpl => scripts.reduce((html, scriptSrc) => `${html}${genScriptReplaceSymbol(scriptSrc)}`, tpl);
		// 通过 getEmbedHTML 生成拼接后的模版字符串，返回一个对应，涵盖模版内容和下面的方法及参数
		return getEmbedHTML(getTemplate(setScriptPlaceholder2HTML(setStylePlaceholder2HTML(html))), styles, { fetch }).then(embedHTML => ({
			// 模版字符串
			template: embedHTML,
			// 获取公共路径
			assetPublicPath: getPublicPath(entry),
			// 获取外部脚本
			getExternalScripts: () => getExternalScripts(scripts, fetch),
			// 获取外部样式表
			getExternalStyleSheets: () => getExternalStyleSheets(styles, fetch),
			// 执行脚本
			execScripts: (proxy, strictGlobal, execScriptsHooks = {}) => {
				if (!scripts.length) {
					return Promise.resolve();
				}
				return execScripts(scripts[scripts.length - 1], scripts, proxy, {
					fetch,
					strictGlobal,
					beforeExec: execScriptsHooks.beforeExec,
					afterExec: execScriptsHooks.afterExec,
				});
			},
		}));

	} else {
		throw new SyntaxError('entry scripts or styles should be array!');
	}
}
```

### test-custome-fetch.js

看下一个测试文件，这里面就一个 `case`，看输入和预期知道这是一个自定义 `fetch` 的验证。

### importHTML

```javascript
export default function importHTML(url, opts = {}) {
	let fetch = defaultFetch;
	let autoDecodeResponse = false;
	let getPublicPath = defaultGetPublicPath;
	let getTemplate = defaultGetTemplate;

	// compatible with the legacy importHTML api
	// 如果opts是函数，就当做fetch函数处理
	if (typeof opts === 'function') {
		fetch = opts;
	} else {
		// fetch option is availble
		if (opts.fetch) {
			// 如果是函数，就当做fetch函数处理
			// fetch is a funciton
			if (typeof opts.fetch === 'function') {
				fetch = opts.fetch;
			} else { // configuration
				// fn 是函数当做fetch处理
				fetch = opts.fetch.fn || defaultFetch;
				autoDecodeResponse = !!opts.fetch.autoDecodeResponse;
			}
		}
		getPublicPath = opts.getPublicPath || opts.getDomain || defaultGetPublicPath;
		getTemplate = opts.getTemplate || defaultGetTemplate;
	}
	// 如果有缓存，则直接返回
	return embedHTMLCache[url] || (embedHTMLCache[url] = fetch(url)
	// 将响应转成字符串
		.then(response => readResAsString(response, autoDecodeResponse))
		.then(html => {

			const assetPublicPath = getPublicPath(url);
			// 通过 processTpl 处理获取模版字符串、脚本、入口、样式等
			const { template, scripts, entry, styles } = processTpl(getTemplate(html), assetPublicPath);

			return getEmbedHTML(template, styles, { fetch }).then(embedHTML => ({
				template: embedHTML,
				assetPublicPath,
				getExternalScripts: () => getExternalScripts(scripts, fetch),
				getExternalStyleSheets: () => getExternalStyleSheets(styles, fetch),
				execScripts: (proxy, strictGlobal, execScriptsHooks = {}) => {
					if (!scripts.length) {
						return Promise.resolve();
					}
					return execScripts(entry, scripts, proxy, {
						fetch,
						strictGlobal,
						beforeExec: execScriptsHooks.beforeExec,
						afterExec: execScriptsHooks.afterExec,
					});
				},
			}));
		}));
}
```

### test-exec-scripts.js

显然，这个是用来验证加载脚本、脚本钩子、远端脚本、页面内脚本等各个配置下的脚本相关

#### execScripts

```javascript
/**
 * FIXME to consistent with browser behavior, we should only provide callback way to invoke success and error event
 * @param entry
 * @param scripts
 * @param proxy
 * @param opts
 * @returns {Promise<unknown>}
 */
export function execScripts(entry, scripts, proxy = window, opts = {}) {
	// 默认配置
	const {
		fetch = defaultFetch, strictGlobal = false, success, error = () => {
		}, beforeExec = () => {
		}, afterExec = () => {
		},
	} = opts;

	return getExternalScripts(scripts, fetch, error)
		// 获取脚本的内容
		.then(scriptsText => {

			const geval = (scriptSrc, inlineScript) => {
				// 源码
				const rawCode = beforeExec(inlineScript, scriptSrc) || inlineScript;
				// 获取可执行脚本代码
				const code = getExecutableScript(scriptSrc, rawCode, proxy, strictGlobal);
				// 执行脚本代码
				(0, eval)(code);
				// 钩子
				afterExec(inlineScript, scriptSrc);
			};

			function exec(scriptSrc, inlineScript, resolve) {

				const markName = `Evaluating script ${scriptSrc}`;
				const measureName = `Evaluating Time Consuming: ${scriptSrc}`;

				// 性能
				if (process.env.NODE_ENV === 'development' && supportsUserTiming) {
					performance.mark(markName);
				}

				if (scriptSrc === entry) {
					noteGlobalProps(strictGlobal ? proxy : window);

					try {
						// bind window.proxy to change `this` reference in script
						geval(scriptSrc, inlineScript);
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
							// bind window.proxy to change `this` reference in script
							// 直接执行脚本
							geval(scriptSrc, inlineScript);
						} catch (e) {
							// consistent with browser behavior, any independent script evaluation error should not block the others
							throwNonBlockingError(e, `[import-html-entry]: error occurs while executing normal script ${scriptSrc}`);
						}
					} else {
						// external script marked with async
						inlineScript.async && inlineScript?.content
							// 下载完后执行
							.then(downloadedScriptText => geval(inlineScript.src, downloadedScriptText))
							.catch(e => {
								throwNonBlockingError(e, `[import-html-entry]: error occurs while executing async script ${inlineScript.src}`);
							});
					}
				}

				if (process.env.NODE_ENV === 'development' && supportsUserTiming) {
					performance.measure(measureName, markName);
					performance.clearMarks(markName);
					performance.clearMeasures(measureName);
				}
			}

			// 递归
			function schedule(i, resolvePromise) {

				if (i < scripts.length) {
					const scriptSrc = scripts[i];
					const inlineScript = scriptsText[i];

					exec(scriptSrc, inlineScript, resolvePromise);
					// resolve the promise while the last script executed and entry not provided
					if (!entry && i === scripts.length - 1) {
						// 全部执行完后成功回调
						resolvePromise();
					} else {
						schedule(i + 1, resolvePromise);
					}
				}
			}

			return new Promise(resolve => schedule(0, success || resolve));
		});
}
```

### test-process-tpl.js

这个就是处理html内容，处理成对应的脚本、样式、入口等

#### processTpl

```javascript
/**
 * parse the script link from the template
 * 1. collect stylesheets
 * 2. use global eval to evaluate the inline scripts
 *    see https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function#Difference_between_Function_constructor_and_function_declaration
 *    see https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval#Do_not_ever_use_eval!
 * @param tpl
 * @param baseURI
 * @stripStyles whether to strip the css links
 * @returns {{template: void | string | *, scripts: *[], entry: *}}
 */
export default function processTpl(tpl, baseURI) {

	let scripts = [];
	const styles = [];
	let entry = null;
	const moduleSupport = isModuleScriptSupported();
	// 字符串按规则替换
	const template = tpl

		/*
		remove html comment first
		*/
		// 删除注释
		.replace(HTML_COMMENT_REGEX, '')
		// 匹配闭合标签
		.replace(LINK_TAG_REGEX, match => {
			/*
			change the css link
			*/
			const styleType = !!match.match(STYLE_TYPE_REGEX);
			if (styleType) {
				// 样式格式
				const styleHref = match.match(STYLE_HREF_REGEX);
				const styleIgnore = match.match(LINK_IGNORE_REGEX);
				// 如果是远端样式
				if (styleHref) {

					const href = styleHref && styleHref[2];
					let newHref = href;

					if (href && !hasProtocol(href)) {
						newHref = getEntirePath(href, baseURI);
					}
					if (styleIgnore) {
						return genIgnoreAssetReplaceSymbol(newHref);
					}
					// 需要获取的样式列表
					styles.push(newHref);
					return genLinkReplaceSymbol(newHref);
				}
			}

			const preloadOrPrefetchType = match.match(LINK_PRELOAD_OR_PREFETCH_REGEX) && match.match(LINK_HREF_REGEX) && !match.match(LINK_AS_FONT);
			if (preloadOrPrefetchType) {
				const [, , linkHref] = match.match(LINK_HREF_REGEX);
				return genLinkReplaceSymbol(linkHref, true);
			}

			return match;
		})
		.replace(STYLE_TAG_REGEX, match => {
			if (STYLE_IGNORE_REGEX.test(match)) {
				return genIgnoreAssetReplaceSymbol('style file');
			}
			return match;
		})
		// 处理脚本
		.replace(ALL_SCRIPT_REGEX, (match, scriptTag) => {
			const scriptIgnore = scriptTag.match(SCRIPT_IGNORE_REGEX);
			const moduleScriptIgnore =
				(moduleSupport && !!scriptTag.match(SCRIPT_NO_MODULE_REGEX)) ||
				(!moduleSupport && !!scriptTag.match(SCRIPT_MODULE_REGEX));
			// in order to keep the exec order of all javascripts

			const matchedScriptTypeMatch = scriptTag.match(SCRIPT_TYPE_REGEX);
			const matchedScriptType = matchedScriptTypeMatch && matchedScriptTypeMatch[2];
			if (!isValidJavaScriptType(matchedScriptType)) {
				return match;
			}

			// if it is a external script
			if (SCRIPT_TAG_REGEX.test(match) && scriptTag.match(SCRIPT_SRC_REGEX)) {
				/*
				collect scripts and replace the ref
				*/

				const matchedScriptEntry = scriptTag.match(SCRIPT_ENTRY_REGEX);
				const matchedScriptSrcMatch = scriptTag.match(SCRIPT_SRC_REGEX);
				let matchedScriptSrc = matchedScriptSrcMatch && matchedScriptSrcMatch[2];

				if (entry && matchedScriptEntry) {
					throw new SyntaxError('You should not set multiply entry script!');
				} else {

					// append the domain while the script not have an protocol prefix
					if (matchedScriptSrc && !hasProtocol(matchedScriptSrc)) {
						matchedScriptSrc = getEntirePath(matchedScriptSrc, baseURI);
					}

					entry = entry || matchedScriptEntry && matchedScriptSrc;
				}

				if (scriptIgnore) {
					return genIgnoreAssetReplaceSymbol(matchedScriptSrc || 'js file');
				}

				if (moduleScriptIgnore) {
					return genModuleScriptReplaceSymbol(matchedScriptSrc || 'js file', moduleSupport);
				}

				if (matchedScriptSrc) {
					const asyncScript = !!scriptTag.match(SCRIPT_ASYNC_REGEX);
					scripts.push(asyncScript ? { async: true, src: matchedScriptSrc } : matchedScriptSrc);
					return genScriptReplaceSymbol(matchedScriptSrc, asyncScript);
				}

				return match;
			} else {
				if (scriptIgnore) {
					return genIgnoreAssetReplaceSymbol('js file');
				}

				if (moduleScriptIgnore) {
					return genModuleScriptReplaceSymbol('js file', moduleSupport);
				}

				// if it is an inline script
				const code = getInlineCode(match);

				// remove script blocks when all of these lines are comments.
				const isPureCommentBlock = code.split(/[\r\n]+/).every(line => !line.trim() || line.trim().startsWith('//'));

				if (!isPureCommentBlock) {
					scripts.push(match);
				}

				return inlineScriptReplaceSymbol;
			}
		});

	scripts = scripts.filter(function (script) {
		// filter empty script
		return !!script;
	});

	// 返回模版、脚本列表、样式列表、入口
	return {
		template,
		scripts,
		styles,
		// set the last script as entry if have not set
		entry: entry || scripts[scripts.length - 1],
	};
}
```

### test-utils

这个是来验证辅助工具等

#### defaultGetPublicPath

```javascript
export function defaultGetPublicPath(entry) {
	// 如果配置入口是个对象，直接返回 /
	if (typeof entry === 'object') {
		return '/';
	}
	try {
		// URL 构造函数不支持使用 // 前缀的 url
		// 如果使用的//补齐协议
		const { origin, pathname } = new URL(entry.startsWith('//') ? `${location.protocol}${entry}` : entry, location.href);
		const paths = pathname.split('/');
		// 移除最后一个元素
		paths.pop();
		return `${origin}${paths.join('/')}/`;
	} catch (e) {
		console.warn(e);
		return '';
	}
}
```

#### readResAsString

```javascript
export function readResAsString(response, autoDetectCharset) {
	// 没有下述任何逻辑，直接返回文本
	// 未启用自动检测
	if (!autoDetectCharset) {
		return response.text();
	}

	// 如果没headers，发生在test环境下的mock数据，为兼容原有测试用例
	if (!response.headers) {
		return response.text();
	}

	// 如果没返回content-type，走默认逻辑
	const contentType = response.headers.get('Content-Type');
	if (!contentType) {
		return response.text();
	}

	// 解析content-type内的charset
	// Content-Type: text/html; charset=utf-8
	// Content-Type: multipart/form-data; boundary=something
	// GET请求下不会出现第二种content-type
	let charset = 'utf-8';
	const parts = contentType.split(';');
	if (parts.length === 2) {
		const [, value] = parts[1].split('=');
		// 处理编码格式
		const encoding = value && value.trim();
		if (encoding) {
			charset = encoding;
		}
	}

	// 如果还是utf-8，那么走默认，兼容原有逻辑，这段代码删除也应该工作
	if (charset.toUpperCase() === 'UTF-8') {
		return response.text();
	}

	// 走流读取，编码可能是gbk，gb2312等，比如sofa 3默认是gbk编码
	return response.blob()
		.then(file => new Promise((resolve, reject) => {
			const reader = new window.FileReader();
			reader.onload = () => {
				resolve(reader.result);
			};
			reader.onerror = reject;
			reader.readAsText(file, charset);
		}));
}

```

OK，到这里跟测试用例相关的代码我们已经阅读完了，我们已经对该项目的功能及流程和结构大致有个数了，接下来如果感兴趣就可以再整体来阅读一遍。
