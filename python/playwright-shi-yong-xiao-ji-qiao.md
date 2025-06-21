# playwright使用小技巧

## playwright简介

playwright是微软开源的一款自动化测试框架，其以卓越的性能和强大的功能，在自动化测试和爬虫领域越来越流行，逐步开始替代selenium。本文记录在使用playwright时用到的一些小技巧

## playwright 简单实用

### 安装

```bash
pip install playwright # 安装playwright 库
playwright install # 此时会自动化安装浏览器，非必须
playwright install-deps # 安装一些依赖，非必须
```

### 使用

```python
import json
import time
import asyncio


chrome_path = 'chrome 路径'

with sync_playwright() as p:
    # 使用系统安装的 Chromium
    browser = p.chromium.launch(
        executable_path=chrome_path,
        headless=False
        args=[
            "--no-sandbox",
            "--disable-gpu",
            "--disable-setuid-sandbox",
            "--disable-dev-shm-usage",
            "--disable-extensions",
            "--disable-blink-features=AutomationControlled",
            "--start-maximized"
        ]
    )
    context = browser.new_context()
    page = context.new_page()
    page.goto('https://www.baidu.com')
    await asyncio.sleep(300)

```

执行上面的代码，会打开chrome浏览器，然后打开百度页面

## playwright小技巧

### 1. 开启debug模式

在启动，增加参数headless=False，运行代码时，会看到浏览器被打开，可以很方面的看到代码所执行的操作

```python
browser = p.chromium.launch(
        executable_path=chrome_path,
        headless=False
    )
```

### 2. 隐藏爬虫指纹

此处主要是将js对象navigator.webdriver 隐藏掉，同时设置使用的系统平台信息

```python
init_script = """
    Object.defineProperty(navigator, 'webdriver', { get: () => undefined });
    Object.defineProperty(navigator, 'platform', { get: () => 'MacIntel' });
    window.chrome = {
        runtime: {},
        // 添加其他 Chrome 对象属性
    }
"""

page = context.new_page(java_script_enabled=True)
page.add_init_script(script=init_script)
from playwright.sync_api import sync_playwright
stealth_sync(page)
```

### 3. 上下文信息传递和保存

大多数网站都要求登录，我们在使用playwright时经常需要让用户处于登录态才能进行下一步操作，此时我们可以通过上下文，在初始化时，让用户处于登录态，同时操作完成是，将上下文信息持久化存储

#### 初始化上下文

```python

    with open('state.json') as f:
        state = json.load(f)
    context = browser.new_context(storage_state=state)
```

在初始化context时，可以通过storage\_state参数将上下文信息传递进去，storage\_state支持两种格式，一种是文件路径，另一种是python dict形式；因此我可以将上下文保存到文件中，通过文件的形式传递，也可以从缓存或者数据库中读取，构造成dict形式传递

保存上下文

```python
    state = context.storage_state()
    with open('state.json', 'w') as f:
        json.dump(state, f)
```

通过context.storage\_state()可以拿到当先上下文信息，然后可以保存到文件，或者持久化到缓存或者数据库中

### 4. 打印控制台信息

在遇到异常时，尤其是js执行异常时，排查非常困难，因此可以通过将控制台信息打印出来，帮助我们排查问题

```python
    def log_console(msg):
        print("Browser Console:", msg.text)
        print("Browser Console:", msg)
        location = msg.location
        if location:
            url = location.get("url")
            if url:
                print("Resource URL:", url)
            else:
                print("Resource URL not available")
        else:
            print("Location information not available")


    page.on('console', log_console)
```

通过定义方法，然后通过page.on 注册

### 5. 打印网络请求和响应信息

有些站点可能针对爬虫做了特殊处理，比如我知道的有些站点检测到爬虫时会返回400或者202，针对这种情况在排查时非常困难且耗时，因此可以将网络请求信息全部打印，协助排查

```python
    page.on("request", lambda request: print(
        f"请求: {request.method} {request.url} {request.headers}"))
    # 监听响应事件
    page.on("response",
            lambda response: print(
                f"响应: {response.status} {response.url} {response.headers}"))
```

也是通过page.on注册

### 6. 篡改请求或者响应

在某些时候，我们可能希望拦截请求，然后针对请求或者响应做一些特殊处理来模拟一些情况，此时如果可以篡改请求，将会非常有用

这里主要通过page.route方法实现

#### 6.1 篡改请求

```python
   def route_interceptor(route):
        headers = dict(route.request.headers)
        # headers['cookie'] = ''
        print(headers)
        if 'cookie' in headers:
            print('移除cookie')
            del headers['cookie']
        print(
            f"拦截请求: {type(route)} {route.request.method} {route.request.url} {headers}")
        route.continue_(headers=headers)

    page.route('**/*', route_interceptor)
```

6.2 篡改响应

<pre class="language-python"><code class="lang-python">def handle_route(route):
    logger.info("handle_route call")
    request = route.request
    url = request.url

    # 跳过非静态资源
    if not is_static_resource(url):
        route.continue_()
        return

    logger.info("handle_route 开始缓存")
    # 生成基于 URL 的唯一缓存文件名
    url_hash = hashlib.sha256(url.encode()).hexdigest()
    cache_file = Path(os.path.join(GLOBAL_CACHE_DIR, url_hash))

    # 如果缓存存在，使用缓存响应
    if cache_file.exists():
        logger.info(f"使用全局缓存: {url}")
        content_type = get_content_type(url)
        route.fulfill(
            status=200,
            headers={"Content-Type": content_type},
            body=cache_file.read_bytes()
        )
        return

    # 否则请求并缓存
    logger.info(f"请求并缓存: {url}")
    route.continue_()
    response = route.request.response()

    # 只缓存成功的静态资源
    if response.status == 200:
        try:
            body = response.body()
            cache_file.write_bytes(body)
            route.fulfill(
                status=200,
                headers=dict(response.all_headers()),
                body=body
            )
            return
        except Exception as e:
            logger.info(f"缓存失败 {url}: {e}")

    route.fulfill(
        status=response.status,
        headers=dict(response.all_headers()),
        body=response.body()
    )

<strong>    page.route('**/*', handle_route)
</strong></code></pre>

笔者使用此方法实现了静态文件缓存

### 7. 屏幕截图

如果开启无头模式，将没有浏览器界面，此时发生了什么我们并不清楚，因此可以通过屏幕截图的方式来保存现场

```python
page.screenshot('图片路径')
```
