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
    const { scripts = [], styles = [], html = '' } = entry;
    // 拼接 html 和 styles
    const setStylePlaceholder2HTML = tpl => styles.reduceRight((html, styleSrc) => `${genLinkReplaceSymbol(styleSrc)}${html}`, tpl);
    // 再拼接 scripts
    const setScriptPlaceholder2HTML = tpl => scripts.reduce((html, scriptSrc) => `${html}${genScriptReplaceSymbol(scriptSrc)}`, tpl);
    // 通过 getEmbedHTML 生成拼接后的模版字符串，返回一个对应，涵盖模版内容和下面的方法及参数
    return getEmbedHTML(getTemplate(setScriptPlaceholder2HTML(setStylePlaceholder2HTML(html))), styles, { fetch }).then(embedHTML => ({
        template: embedHTML,
        assetPublicPath: getPublicPath(entry),
        getExternalScripts: () => getExternalScripts(scripts, fetch),
        getExternalStyleSheets: () => getExternalStyleSheets(styles, fetch),
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
}
```
