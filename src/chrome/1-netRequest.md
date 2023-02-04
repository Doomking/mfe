# 背景
最近在开发过程中，总是面临各种环境问题：比如需要添加额外的请求头，才能请求到特定环境的接口；比如接口异常，出现跨域问题等；极大影响了开发调试效率，虽然能处理网络请求/响应头，解决跨域的chrome扩展很多，但是因为我们的场景有一些特殊，所以这些扩展要么不是不好用，就是不起作用。恰好我们自己开发过，用来进行接口Mock的chrome插件，一直用着很方便，那何不在此插件的基础上，再增加可以自定义请求头及跨域的能力呢？

网上一搜，chrome.webRequest API轻松解决此问题，看来很快就能如丝般顺滑的开发调试代码了。
说干就干，代码都快要写好了，结果，什么！webRequest API在MV3中已经废弃了，废弃了！

那怎么办，怎么办？没事，咱MV3不是有declarativeNetRequest API呢，分分钟也能搞定。。。

然而declarativeNetRequest API的教程真是太少了，官方文档太简单，太不清晰了，其他文章只有简单的介绍，但是很少有实战的，这下真是麻烦大了。

真是一坑又一坑，落入了持续的艰难的避坑斗争！

**webRequest vs declarativeNetRequest**

话说 webRequest API用着好好的，既简单方便，又成熟稳定，为什么在MV3中要废弃呢？这不是自找麻烦么？

Chrome官方的解释是，由于WebRequest API的机制是当网络请求发起进，就会拦截，然后就可以进行各种分析处理，但是很容易被开发者滥用，导致性能受影响，并且给开发者的权限太大，很可能导致安全问题。因此在MV3中，chrome扩展提供了声明式的API，即是chrome.declarativeNetRequest，即在manifest.json中配置一套规则，通过规则匹配，就可以指定要拦截的请求或者类型，这种方式更加安全，性能也更好，也是官方主推的方式。

二者的核心区别就是chrome.webRequest API在实现网络请求修改时，是通过实时拦截，直接修改，可以很方便的在任一个请求中进行拦截，分析和修改，非常简单和方便；但是chrome.declarativeNetRequest API是声明式的，预先通过配置好一套规则，然后当网络请求时，如果匹配上了规则，就会根据规则对网络请求修改，否则不处理。这样网络请求和规则设定就变成了2个非常独立的过程，根本无法监控网络请求及做出相应的处理，整个实现思路和chrome.webRequest API的方法就完全不一样了；

# declarativeNetRequest API介绍

官方文档：<https://developer.chrome.com/docs/extensions/reference/declarativeNetRequest/>

chrome.declarativeNetRequest在官方文档上有非常多的说明和介绍，比较细致，但内容多且是英文，不仅不容易理解，而且缺少案例和Demo，读起来云里雾里，非常吃力；

这个API的核心就是进行规则配置和匹配，我们在开发中主要的工作就是规则配置，declarativeNetRequest主要提供了2种规则配置方式：静态规则和动态规则，具体使用如下：

## 1.  权限设定

在manifest.json的permissions中添加以下权限：

```json
{
	"permissions": ["declarativeNetRequest", "declarativeNetRequestWithHostAccess", "declarativeNetRequestFeedback"],
	"host_permissions": ["<all_urls>"]
}
```
-   若申请的是declarativeNetRequest权限，则阻止或升级请求不需要申请"host_permissions"，但是如果需要重定向请求，或者修改请求头，则要申请相应的"host_permissions"
-   若申请的是declarativeNetRequestWithHostAccess权限，则任何功能都需要申请相应的"host_permissions"。

-   若申请的是declarativeNetRequestFeedback，则可以使用getMatchedRules()方法。并且能够监听匹配成功事件，但是该监听仅允许用于测试环境使用。

## 2.  添加规则集

使用declarativeNetRequest对网络请求做处理，实质上是定义一些json格式的规则，这些规则会匹配所有的网络请求，并遵从这些规则做相应处理。匹配规则有静态规则和动态规则两种。

### 静态规则配置

1.  需要创建一个配置规则集的json文件，通过json文件存储一个数组，里面可以放置多个规则。
1.  需要在manifest.json中添加一个declarative_net_request的配置信息，通过path指向配置规则集的json文件。

manifest.json

```json
{
	"name": "declarativeNetRequest extension",
	"version": "1",
	"declarative_net_request": {
		"rule_resources": [{
			"id": "ruleset_1",
			"enabled": true,
			"path": "rules.json"
		}]
	},
	"permissions": ["*://*.google.com/*", "*://*.abcd.com/*", "*://*.example.com/*", "https://*.xyz.com/*", "*://*.headers.com/*", "declarativeNetRequest"],
	"manifest_version": 3
}
```

**rule_resources有三个参数:**

-   id：每个规则集需要对应一个id

-   enabled：规则集是否启用

-   path：配置规则的json文件地址

rules.json

```json
[{
		"id": 1,
		"priority": 1,
		"action": {
			"type": "block"
		},
		"condition": {
			"urlFilter": "google.com",
			"resourceTypes": ["main_frame"]
		}
	},
	{
		"id": 2,
		"priority": 1,
		"action": {
			"type": "allow"
		},
		"condition": {
			"urlFilter": "google.com/123",
			"resourceTypes": ["main_frame"]
		}
	},
	{
		"id": 3,
		"priority": 2,
		"action": {
			"type": "block"
		},
		"condition": {
			"urlFilter": "google.com/12345",
			"resourceTypes": ["main_frame"]
		}
	},
	{
		"id": 4,
		"priority": 1,
		"action": {
			"type": "redirect",
			"redirect": {
				"url": "https://example.com"
			}
		},
		"condition": {
			"urlFilter": "google.com",
			"resourceTypes": ["main_frame"]
		}
	},
	{
		"id": 5,
		"priority": 1,
		"action": {
			"type": "redirect",
			"redirect": {
				"extensionPath": "/a.jpg"
			}
		},
		"condition": {
			"urlFilter": "abcd.com",
			"resourceTypes": ["main_frame"]
		}
	},
	{
		"id": 6,
		"priority": 1,
		"action": {
			"type": "redirect",
			"redirect": {
				"transform": {
					"scheme": "https",
					"host": "new.example.com"
				}
			}
		},
		"condition": {
			"urlFilter": "||example.com",
			"resourceTypes": ["main_frame"]
		}
	},
	{
		"id": 7,
		"priority": 1,
		"action": {
			"type": "redirect",
			"redirect": {
				"regexSubstitution": "https://\\1.xyz.com/"
			}
		},
		"condition": {
			"regexFilter": "^https://www\\.(abc|def)\\.xyz\\.com/",
			"resourceTypes": [
				"main_frame"
			]
		}
	},
	{
		"id": 8,
		"priority": 2,
		"action": {
			"type": "allowAllRequests"
		},
		"condition": {
			"urlFilter": "||b.com/path",
			"resourceTypes": ["sub_frame"]
		}
	},
	{
		"id": 9,
		"priority": 1,
		"action": {
			"type": "block"
		},
		"condition": {
			"urlFilter": "script.js",
			"resourceTypes": ["script"]
		}
	},
	{
		"id": 10,
		"priority": 2,
		"action": {
			"type": "modifyHeaders",
			"responseHeaders": [{
					"header": "h1",
					"operation": "remove"
				},
				{
					"header": "h2",
					"operation": "set",
					"value": "v2"
				},
				{
					"header": "h3",
					"operation": "append",
					"value": "v3"
				}
			]
		},
		"condition": {
			"urlFilter": "headers.com/123",
			"resourceTypes": ["main_frame"]
		}
	},
	{
		"id": 11,
		"priority": 1,
		"action": {
			"type": "modifyHeaders",
			"responseHeaders": [{
					"header": "h1",
					"operation": "set",
					"value": "v4"
				},
				{
					"header": "h2",
					"operation": "append",
					"value": "v5"
				},
				{
					"header": "h3",
					"operation": "append",
					"value": "v6"
				}
			]
		},
		"condition": {
			"urlFilter": "headers.com/12345",
			"resourceTypes": ["main_frame"]
		}
	}
]
```


### 动态规则集配置

动态规则集可以通过代码，调用chrome.declarativeNetRequest.updateDynamicRules方法，实现动态的规则更新；

```js
chrome.declarativeNetRequest.updateDynamicRules({
	addRules: [{
			"id": 1,
			"priority": 1,
			"action": {
				"type": "block"
			},
			"condition": {
				"urlFilter": "abc",
				"domains": ["foo.com"],
				"resourceTypes": ["script"]
			}
		},
		{
			"id": 2,
			"priority": 1,
			"action": {
				"type": "block"
			},
			"condition": {
				"urlFilter": "abc",
				"domains": ["foo.com"],
				"resourceTypes": ["script"]
			}
		}, {
			"id": 3,
			"priority": 1,
			"action": {
				"type": "modifyHeaders",
				"responseHeaders": [{
						"header": "h1",
						"operation": "set",
						"value": "v4"
					},
					{
						"header": "h2",
						"operation": "append",
						"value": "v5"
					},
					{
						"header": "h3",
						"operation": "append",
						"value": "v6"
					}
				]
			},
			"condition": {
				"urlFilter": "headers.com/12345",
				"resourceTypes": ["main_frame"]
			}
		},
	],
	removeRuleIds: [11, 12, 13],
}, () => {})
```

## 3.  规则

一个规则以对象的格式定义，其中有四个属性：id、priority、action、condition

- id:每个规则需要对应一个id，不同的规则不能设置相同的id
- priority：优先级，填一个数字，数字越大，优先级越高
- action：规则匹配时，采取的操作。包括以下四个参数：
    1. type: 必填参数，操作的类型，参数为："block"（拦截请求）, "redirect"（重定向请求）, "allow"（允许请求）, "upgradeScheme"（升级请求）, "modifyHeaders"（修改请求头）, 或"allowAllRequests"（允许所有请求）
    2. redirect：选填，只有type为redirect时才可填，配置如何执行重定向
-   extensionPath:相对于扩展目录的路径。应该以'/'开头
-   regexSubstitution:指定正则表达式过滤器的规则的替换模式
-   url：需要重定向到的网址，不可以重定向到javascript url
-   transform：url转换
-   requestHeaders：选填，只有操作类型为modifyHeaders时才可填，用于修改请求头
-   responseHeaders：选填，只有操作类型为modifyHeaders时才可填，用于修改响应头

- condition：触发此规则的条件，有以下参数，皆是选填参数，常用参数（域名匹配、请求方法匹配、资源类型匹配、url匹配）

    1. domainType：指定网络请求是发起它的域的"firstParty"（第一方）还是"thirdParty"（第三方）。如果省略，则接受所有请求。
    2. domains：匹配域名列表，如省略则匹配所有域名的请求
    3. excludedDomains：和domains相反，指定一个排除匹配规则的域名列表，优先级高于domains
    4. requestMethods：http匹配请求方法（"get"、"post"等）列表，如省略则匹配所有请求方法。
    5. excludedRequestMethods：与requestMethods相反，排除一个请求方法列表，优先级高于requestMethods
    6. resourceTypes：匹配资源类型（"main_frame"、""xmlhttprequest"等）列表，如省略则匹配所有请求类型
    7. excludedResourceTypes：跟resourceTypes相反
    8. tabIds：匹配tabId列表，如省略则匹配所有tabs页
    9. excludedTabIds：和tabIds相反
    10. isUrlFilterCaseSensitive：是否区分大小写，默认是true，区分大小写
    11. regexFilter：匹配网络请求的正则表达式
    12. urlFilter：匹配网络请求的url

# 修改网络请求的MV3实现

我们的核心诉求就是修改请求头和跨域设置，具体功能就是在Action面板上配置请求头信息和跨域信息，然后保存到chrome.storage里面，发生请求时读取chrome.storage存储的配置信息，然后修改请求信息；

-   如果是用chrome.webRequest来实现，过程和实现非常简单，基本如下：


![MV2流程.drawio.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/38234c32b073480fb6fd3f0d4d2a7c77~tplv-k3u1fbpfcp-watermark.image?)

-   但是现在在MV3中，就只能使用chrome.declarativeNetRequest API来实现了，整个过程就变得不一样了，如下图：


![规则更新流程.drawio.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a79f439ee93d40e5b22f4c44bbfc43ab~tplv-k3u1fbpfcp-watermark.image?)

其中Action面板和chrome.storage的实现就比较容易了，这里就不具体说明了，核心主要是如何实现动态的更新规则，这里面就涉及到什么时机更新规则，怎么更新规则，在哪里更新规则，用哪些方法等，具体如下：

-   在background的service work 中执行初始化及storage中规则数据变更时，执行规则更新代码；
-   主要用到chrome.declarativeNetRequest.updateDynamicRules方法，首先获取当前已存在的动态规则集，然后获取 已存在规则集的id数组，再通过storage中的配置生成规则，调用updateDynamicRules方法，传入addRules和removeIds2个参数，用于添加新的规则集和删除已有的规则集；

样例代码如下：


```js
const createRules = (list) => {
	const headers = list.map((item) => {
		const {
			name,
			value
		} = item;
		const header = {
			header: name,
			value,
			ooperation: 'set',
		};
		return header;
	});

	const rule = {
		id,
		priority: 1,
		action: {
			type: 'modifyHeaders',
			requestHeaders: headers,
		},
		condition: {
			urlFilter: '*',
			resourceTypes: ['main_frame', 'sub_frame', 'xmlhttprequest'],
		},
	};
	return rule;
};

const getNewRules = async () => {
	try {
		const list = await getHeaderList(); // 从storage内获取规则数据，由于不是网络请求修改的核心代码，故省略函数实现
		const rule = createRules(list);
		return [rule];
	} catch (err) {
		return null;
	}
};

const getCurrentRules = async () => {
	const rules = await chrome.declarativeNetRequest.getDynamicRules();
	return rules;
};

const setRules = async () => {
	try {
		const newRules = await getNewRules();
		const currentRules = await getCurrentRules();
		const removeIds = currentRules?.map((rule) => rule.id);
		const option = {
			addRules: newRules,
			removeRuleIds: removeIds,
		};
		chrome.declarativeNetRequest.updateDynamicRules(option, () => {
			console.log(chrome.runtime.lastError);
		});
	} catch (err) {
		console.log(err);
	}
};

const initRules = async () => {
	setRules();
};

const updateRules = () => {
	chrome.storage.onChanged.addListener(async (changes, area) => {
		if (changes && changes.updateRule && area === 'local') {
			setRules();
		}
	});
};

const setRequestheader = () => {
	initRules();
	updateRules();
};

setRequestheader();
```

# 问题

declarativeNetRequest，对于官方来说，确实解决了性能和安全等问题，但是对于开发者来说，由于declarativeNetRequest目前的教程和实践比较少，现成的代码及实现也很少，因此很难找到合适的代码及相应的解决方案和问题处理，实现起来会遇到各种各样的问题，导致很简单的一段代码，调试了很久才成功；

其中“Rule with id xxx specifies an invalid request header to be appended. Only standard HTTP request headers that can specify multiple values for a single entry are supported”问题，特别困扰，隔了一个春节，才最终解决；

当前网上根本无法找到这个问题的相关内容及说明，只能苦苦调试、思索了；问题的大概意思就是同一个请求头，设置了多值，因为这个请求头不是标准的HTTP请求头，所以就报错了；通过不懈的调试，调试，调试。。。终于终于问题解决了！！！

问题的核心应该包括2个部分：

1、background脚本启动后，会执行2个部分：

-   initRules函数（执行规则初始设定，获取storage内规则信息，然后执行动态规则更新，提供新的规则集，同时将要删除的规则id数据设置为空）
-   updateRules函数（在storage触发数据变更时执行，并且也是获取storage内规则信息，然后执行动态规则更新，提供新的规则集，但是将要删除的规则id数据设置为当前的规则集id数组，确保规则集数据不重复）

但是这2个部分，因为代码里面存在多个异步过程，最终是initRules先执行规则更新，还是updateRules先执行规则更新，是不确定的；如果是updateRules先执行了，然后initRules后执行更新，那么因为initRules执行的时候，removeRuleIds为[]，这样没有将之前的规则集删除，结果就导致请求头重复了，然后报错了；

2、生成规则的时候，当规则的action的type为modifyHeaders时，请求头的数据结构包括3个字段：header、value和operation，其中operation的取值包括3个值：append、set、remove；我理解remvoe是表示移除头，append应该是新增，set应该是修改；所以在代码实现上用的是append结果，结果就一直报错、报错，然后改成set就成了！

然后就最终成功了！！！