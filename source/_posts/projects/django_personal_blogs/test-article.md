---
title: 基于Django的简单个人博客
date: 2023-11-23 23:48:06
tags:
---

## 前言

2021年春考研结束后，看着自己的csdn账号，看着前两年发过的一篇小博客，决定还是要多少写一点博客之类的东西，毕竟可以拿来面试用~

思前想后，决定还是自己搞一个“博客”项目（即本站https://zhengfei.xin）出来，然后再往上存东西，毕竟买来的服务器不能光放着。

其实我对Python的Web开发了解并不多，但是Python相对用的多一点，也就决定用Python来写，毕竟语言是次要的，主要是想借这个东西练练手。不过也有计划后续再把这个博客改程Java + Vue的。

## 技术选型

主要语言自然是Python，然后在Django和Flask这两个我略有使用的框架中选择了Django，主要还是我比较懒，选Django就直接把轮子都装个差不多了，省心！

前端部分直接使用了Django的模板，然后从万能的互联网上去搜了几个静态的网页模板，就这样就准备好了这个网站的原材料。

除开语言层面，我又掏出了MySQL和Redis，MySQL是必须的，Redis不是，主要是想把Redis塞进来，然后我就拥有了一个使用了Redis的项目√

## 具体实现

> 关于Django和其他组件的基础使用此处就不再赘述

### 接口父类

在我第一次写的时候，其实并没有搞一个父类出来，写了一大半之后，发现很多接口的代码重复度太高了，此时突然想起了OOP之类的，所以抽象出了几个类，作为所有接口的父类。

首先是父类中的父类，所有接口都继承自这个父类,具体功能参照注释。

```python
class AbstractApiView(APIView):
    CODE = 500
    # 当访问接口为非json模式时，所要加载的模板文件
    TEMPLATE = "manage/404.html"

    # post、get、delete方法均调用新定义的request_handle方法，统一处理数据
    def post(self, requests):
        return self.request_handle(self.post_solution, requests.POST, requests)

    def get(self, requests):
        return self.request_handle(self.get_solution, requests.GET, requests)

    def delete(self, requests):
        return self.request_handle(self.delete_solution, requests.POST, requests)

    # 统一处理数据
    def request_handle(self, method, methodType, request):
        try:
            # 尝试通过相应的方法获取接口返回的数据
            result = method(request)
        except Exception as e:
            self.CODE = 500
            result = {}

        # 判断访问接口的类型，如果是json则返回json字符串，否则返回html页面
        self.reqType = methodType.get("type")
        responseData = {
            "code": self.CODE,
            "message": STATUS_INFO[self.CODE],
            "data": result["data"] if result not in [None, {}] else {},
            "config": SITE_CONFIG
        }
        if self.reqType == "json":
            return JsonResponse(self.data_wrap(responseData))
        else:
            return HttpResponse(
                loader.get_template(self.TEMPLATE).render(self.data_wrap(responseData), request)
            )

    # 数据包装函数，通过在子类中重写此方法，来对数据进行一些特有的处理和封装
    def data_wrap(self, responseData):
        return responseData

    # post、get、delete各自的处理方法
    def post_solution(self, requests):
        pass

    def get_solution(self, requests):
        pass

    def delete_solution(self, requests):
        pass
```

接下来就是对不同访问情况的代码进行重写，例如：

1. 在简历页面，统一获取相关基本信息
2. 在后台管理页面，判断登录状态是否存在，若登录状态失效，则弹出登录页面

再之后就是具体的各个接口的处理，有了以上两层封装，对于实际接口一般只需要定义好使用的数据库、要显示的页面模板即可，整体是降低了不少代码量的。

### 引入Markdown编辑与解析

除以上访问接口的封装外，还添加了两个相对重要的组件。其一就是Markdown编辑与解析功能。

对于Markdown的编辑，直接使用了mdeditor组件，Django有其定制化的Module。在`INSTALLED_APPS`中导入即可使用。

在`settings.py`中可以对此编辑器进行配置，代码如下:

```python
MDEDITOR_CONFIGS = {
    'default':{
        'width': '100%',  # 自定义编辑框宽度
        'heigth': '100%',   # 自定义编辑框高度
        'toolbar': ["undo", "redo", "|",
                    "bold", "del", "italic", "quote", "|",
                    "list-ul", "list-ol", "hr", "|",
                    "link", "reference-link", "image", "code", "preformatted-text", "table", "datetime",
                    "emoji", "html-entities", "|",
                    "help", "info",
                    "||", "preview", "watch", "fullscreen"],  # 自定义编辑框工具栏
        'upload_image_formats': ["jpg", "jpeg", "gif", "png", "bmp", "webp"],  # 图片上传格式类型
        'image_folder': 'mdImage',  # 图片保存文件夹名称
        'theme': 'default',  # 编辑框主题 ，dark / default
        'preview_theme': 'default',  # 预览区域主题， dark / default
        'editor_theme': 'default',  # edit区域主题，pastel-on-dark / default
        'toolbar_autofixed': True,  # 工具栏是否吸顶
        'search_replace': True,  # 是否开启查找替换
        'emoji': True,  # 是否开启表情功能
        'tex': True,  # 是否开启 tex 图表功能
        'flow_chart': True,  # 是否开启流程图功能
        'sequence': True,  # 是否开启序列图功能
        'watch': True,  # 实时预览
        'lineWrapping': True,  # 自动换行
        'lineNumbers': True  # 行号
    }
}
```

编辑好Markdown后，又需要一个解析的功能，对于解析则使用Python的Markdown模块，此模块功能已足够强大，支持数学公式，并且同样可以进行定制化。对此功能进行简单封装，如下：

```python
import markdown

def parse_md(text):
    return markdown.markdown(text.replace("\r\n", ' \n'), extensions=[
        'markdown.extensions.extra',
        'markdown.extensions.codehilite',
        'markdown.extensions.toc',
        'mdx_math',
    ])
```

解析好后，接下来就是对前端显示的样式调整，这个过程并不算难，只是做起来非常需要耐心。

### 基于Haystack的全文搜索引擎

另外，就是需要一个全文搜索引擎，可以对文章标题、介绍、正文等内容进行匹配，然后显示出搜索的结果。同样是贪图方便，暂时先选用了Haystack + Whoosh的方案。

对于此模块的使用不再赘述，在使用过程中，由于不太满意搜索排序的结果，遂重写了相关代码：

```python
    def build_page(self):
        """
        Paginates the results appropriately.

        In case someone does not want to use Django's built-in pagination, it
        should be a simple matter to override this method to do what they would
        like.
        """
        try:
            page_no = int(self.request.GET.get("p", 1))
        except (TypeError, ValueError):
            raise Http404("Not a valid number for page.")

        if page_no < 1:
            raise Http404("Pages should be 1 or greater.")

        start_offset = (page_no - 1) * self.results_per_page
        self.results = self.results.order_by("-id")
        self.results[start_offset : start_offset + self.results_per_page]

        paginator = Paginator(self.results, self.results_per_page)

        try:
            page = paginator.page(page_no)
        except InvalidPage:
            raise Http404("No such page!")

        return (paginator, page)
```

另，通过如下代码，不直接通过接口访问搜索类，间接获得了搜索的结果

```python
Search(results_per_page=10).__call__(requests).content
```
