# 构建新浪微博WebApp简明教程
With Weibo Python SDK & Django Framework

## 啰嗦的话
其实这个东西是为了人工智能的数据挖掘作业，但是微博的Python SDK仅支持Web方式验证，不支持口令方式验证。而TA教的方法是手动copy授权获得的code，我觉得特别蛋疼，就干脆建一个WebApp算了。因为之前写了一篇Django Turorial（[传送门](http://lancy.applesysu.com/?p=26)），所以这一篇就不啰嗦Django了。

## 安装Weibo Python SDK
    $ sudo pip sinaweibopy
[See More](http://michaelliao.github.com/sinaweibopy/)
## 申请开发者和应用
[http://open.weibo.com/](http://open.weibo.com/)

按流程操作即可，应用地址填http://127.0.0.1:8000，方便测试

## 开始建站
* 接Django Tutorial那篇教程开始，我将这个站建在那个mysite下，起名叫amour
* Settings里面填好INSTALLED_APPS，再syncdb
* urls.py里面添加

        url(r'^amour/', include("amour.urls")),
        
## 修改amour.urls
* 我们仅仅需要两个页面，一个页面index有一个开始授权的连接，另一个页面results接受sina的回调，并进行处理
* Edit urls.py

        from django.conf.urls import patterns, url

        urlpatterns = patterns('',
            url(r'^$', 'amour.views.index'),
            url(r'results$', 'amour.views.results'),
        )
        
## 使用Python SDK
我们将在Views.py里面试用Python SDK

    from weibo import APIClient
    
    APP_KEY = '1234567' # app key
    APP_SECRET = 'abcdefghijklmn' # app secret
    CALLBACK_URL = 'http://127.0.0.1:8000/amour/results' # callback url
注意这里的CALLBACK_URL，为了使CALLBACK_URL生效，我们还必须回到注册应用的地方，将应用的高级信息里面的回调地址填成同样的地址。

继续修改

    def index(request):
        client = APIClient(app_key=APP_KEY, app_secret=APP_SECRET, redirect_uri=CALLBACK_URL)
        authorizeUrl = client.get_authorize_url()
        return render_to_response("amour/index.html", {"authorizeUrl": authorizeUrl})
    
    
    def results(request):
        code = request.GET['code']
        client = APIClient(app_key=APP_KEY, app_secret=APP_SECRET, redirect_uri=CALLBACK_URL)
        r = client.request_access_token(code)
        access_token = r.access_token
        expires_in = r.expires_in
        client.set_access_token(access_token, expires_in)
        accountUid = client.get.account__get_uid()
        usersShow = client.get.users__show(uid=accountUid["uid"])
        userTimeline = client.get.statuses__user_timeline(count=100)
        requestResults = {
            "userTimeline": userTimeline,
            "usersShow": usersShow,
        }
        return render_to_response("amour/results.html", requestResults)

再更改模板
index.html

    Welcome 
    <a href="{{ authorizeUrl }}">Begin Authorize</a>
    
results.html

    <h1>Welcome {{ usersShow.name }} </h1>
    <p><img src="{{ usersShow.avatar_large }}" alt="我的头像" /></p>
    <ul>
        <li>我的名字：{{ usersShow.name }}</li>
        <li>我的ID：{{ usersShow.id }}</li>
        <li>我的粉丝：{{ usersShow.followers_count }}</li>
        <li>我的关注：{{ usersShow.friends_count }}</li>
        <li>我的状态数：{{ usersShow.statuses_count }}</li>
        <li>我的签名：{{ usersShow.description }}</li>
    </ul>
    
    <h2>我的微博</h2>
    <ul>
    {% for statuse in userTimeline.statuses %}
        <p>
            <li>{{ statuse.created_at }}</li>
            <ul>
                <li>{{ statuse.text }} </li>
            </ul>
        </p>
    {% endfor %}
    </ul>
    
完工

## 关于Python SDK调用的详细规则
（引用自该SDK的Github Wiki）

首先查看新浪微博API文档，例如：

    API：statuses/user_timeline
    请求格式：GET    
    请求参数：
    source：string，采用OAuth授权方式不需要此参数，其他授权方式为必填参数，数值为应用的AppKey?。
    access_token：string，采用OAuth授权方式为必填参数，其他授权方式不需要此参数，OAuth授权后获得。
    uid：int64，需要查询的用户ID。
    screen_name：string，需要查询的用户昵称。

（其它可选参数略）

调用方法：将API的“/”变为“__”，并传入关键字参数，但不包括source和access_token参数：

    r = client.get.statuses__user_timeline(uid=123456)
    for st in r.statuses:
        print st.text
若为POST调用，则示例代码如下：

    r = client.post.statuses__update(status=u'测试OAuth 2.0发微博')
若为上传调用（Multipart Post），传入file-like object参数，示例代码如下：

    f = open('/Users/michael/test.png', 'rb')
    r = client.upload.statuses__upload(status=u'测试OAuth 2.0带图片发微博', pic=f)
    f.close() # APIClient不会自动关闭文件，需要手动关闭
请注意：上传的文件必须是file-like object，不能是str，因为无法区分一个str是文件还是字段。可以通过StringIO把一个str包装成file-like object。

考虑到大部分调用是GET操作，get可省略，例如，以下两种写法是等价的：

    r = client.statuses__user_timeline(uid=123456)
    r = client.get.statuses__user_timeline(uid=123456)

## 关于WebApp与本地应用的一些思考
我是写本地应用出生的，在刚开始实现这个Web应用的时候，我完全没有意识到Web应用和本地应用的设计逻辑是完全不同的逻辑，因而犯了不少迷糊。

本地应用和Web应用最大的区别就在于Web应用的代码运行在服务器上，其天生就是设计成给无数人同时使用的；而对于一个本地应用，可以说在一台设备上就只会有一个使用者。Web应用，页面和页面之间的耦合度是很低的，他们除了共用modles之外，相互之间只会通过表单进行通信。而本地应用，controller和controller之间，经常会有比较多的交互和通信，为了更好的实现这些需求，我们常常会在本地应用中，使用单例模式来单一的管理某些特定的业务（比如DataManager，HttpClient）。

也正是基于这样的设计思路，我在刚开始使用Python SDK的APIClient的时候，一度想把其设计成Singleton，因为我觉得，对于一个用户来说，有一个唯一的APIClient负责处理该用户的所有有关获取weibo信息的请求，是很安全的做法。后来才发现自己错了。

Web应用是不需要单例模式的，APIClient被设计出来的时候，完成一个用户的请求，根本就不需要保证其APIClient是同一个实例。再验证的时候，会获得一个acess_token，只要将这个token赋值给APIClient后，即使是不同的实例，他们也能起到相当于同一个实例的效果——为指定的一个用户服务。这个用于标示用户身份的token，也并不是像我原先想的那样，存在服务器上，而是简单的可以存在cookie里面，只要不过期，就可以轻易的继续使用。

相反的，如果强硬的把APIClient设计成单例，反而会造成一个巨大的麻烦，也就是当有一个用户该Client堵塞的时候，就会影响到所有的用户。就算是用folking或者多线程或者工厂模式，也会造成很多不必要的资源浪费。倒不如，每一个页面生成一个实例，处理完成后，又让其释放。

## 改进该WebApp，使其仅需要使用一个页面
如前面所说只需将token存在cookie里面，并将CALLBACK_URL改回即可，代码如下：

    def index(request):
        client = APIClient(app_key=APP_KEY, app_secret=APP_SECRET, redirect_uri=CALLBACK_URL)
        requestResults = dict()

    if "code" in request.GET:
        print("in request code, set_cookie")
        code = request.GET['code']
        r = client.request_access_token(code)
        access_token = r.access_token
        expires_in = r.expires_in
        client.set_access_token(access_token, expires_in)

    elif "access_token" in request.COOKIES:
        print("has access_token in cookies")
        access_token = request.COOKIES["access_token"]
        expires_in = request.COOKIES["expires_in"]
        client.set_access_token(access_token, expires_in)

    if not client.is_expires():
        print("client is not expires, get infomation")
        accountUid = client.get.account__get_uid()
        usersShow = client.get.users__show(uid=accountUid["uid"])
        userTimeline = client.get.statuses__user_timeline(count=100)
        requestResults = {
            "usersShow": usersShow,
            "userTimeline": userTimeline,
        }
    else:
        authorizeUrl = client.get_authorize_url()
        requestResults = {
            "authorizeUrl": authorizeUrl
        }
    response = render_to_response("amour/index.html", requestResults)
    if not client.is_expires():
        print("client is not expires, set cookie")
        response.set_cookie("access_token", access_token)
        response.set_cookie("expires_in", expires_in)
    return response
    
index.html

    {% if authorizeUrl %}
        <h1>Welcome</h1>
        <a href="{{ authorizeUrl }}">Begin Authorize</a>
    {% else %}
        <h1>Welcome {{ usersShow.name }} </h1>
        <p><img src="{{ usersShow.avatar_large }}" alt="我的头像" /></p>
        <ul>
            <li>我的名字：{{ usersShow.name }}</li>
            <li>我的ID：{{ usersShow.id }}</li>
            <li>我的粉丝：{{ usersShow.followers_count }}</li>
            <li>我的关注：{{ usersShow.friends_count }}</li>
            <li>我的状态数：{{ usersShow.statuses_count }}</li>
            <li>我的签名：{{ usersShow.description }}</li>
        </ul>
    
        <h2>我的微博</h2>
        <ul>
        {% for statuse in userTimeline.statuses %}
            <p>
                <li>{{ statuse.created_at }}</li>
                <ul>
                    <li>{{ statuse.text }} </li>
                </ul>
            </p>
        {% endfor %}
        </ul>
    {% endif %}