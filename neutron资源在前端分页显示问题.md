## Python Generator及相关client的原生分页函数使用

honeypot api完全按照neutron api的标准定义，用来做项目中honeypot信息的增删改查和展示，最近由于返回数据量较大，研究一下openstack对分页功能的编写流程.

honeypot api调用的honeypot list函数如下：

```python
honeypots = honeypotclient(request).list_honeypots(**params).get('pods')
```

一个典型的从字典返回key为“pods“的值的方法，那么在honeypotclient中它是如何定义的呢？

```python
def list_honeypots(self, retrieve_all=True, **_params):
    return self.list('pods', self.honeypots_path, retrieve_all,
                     **_params)
```

这其中有一个预先定义好的list函数，函数如下：

```python
def list(self, collection, path, retrieve_all=True, **params):
    if retrieve_all:
        res = []
        for r in self._pagination(collection, path, **params):
            res.extend(r[collection])
        return {collection: res}
    else:
        return self._pagination(collection, path, **params)
```

 **其中retrieve_all参数是指是否需要分页，其值为True时，代表获得全部结果。**可以看出无论retrieve_all等于True或False，都要调用_pagination函数。

```python
    def _pagination(self, collection, path, **params):
        if params.get('page_reverse', False):
            linkrel = 'previous'
        else:
            linkrel = 'next'
        next = True
        while next:
            res = self.get(path, params=params)
            import hook
            hook.hook(res["data"])
            yield res["data"]
            next = False
            try:
                for link in res['%s_links' % collection]:
                    if link['rel'] == linkrel:
                        query_str = urlparse.urlparse(link['href']).query
                        params = urlparse.parse_qs(query_str)
                        next = True
                        break
            except KeyError:
                break
```

其中page_reverse控制访问数据的方向，**也就是所谓的“上一页”和“下一页”，默认为False（下一页）。_pagination通过get方法取数据，并且返回一个生成器给list函数。当retrieve_all=True时，list会用for循环一直用该生成器取数据，最终获得所有数据。反之，list返回一个生成器。**
在我想直接利用self.pagination来调用数据时，发现它是显然不行的，因为这个函数返回了一个generator，而我们看list_honeypots函数，它的需求是调用list返回一个key为pods，value为数据信息的字典，供honeypot api做get()使用。而generator显然是没有get方法的，它一般只使用next()方法（python2）。

报错信息如下：

> [Sun Jul 12 12:08:30.258444 2020] [:error] [pid 10972:tid 140333884905216]   File "/usr/share/openstack-dashboard/openstack_dashboard/wsgi/../../openstack_dashboard/api/honeypot.py", line 75, in _honeypot_list
> [Sun Jul 12 12:08:30.258452 2020] [:error] [pid 10972:tid 140333884905216]     honeypots = honeypotclient(request).list_honeypots(**params).get('pods')
> [Sun Jul 12 12:08:30.258459 2020] [:error] [pid 10972:tid 140333884905216] AttributeError: 'generator' object has no attribute 'get'

我应该做的是针对1100个pod的页面，以20个一页做限制，进行数据的显示。理想返回情况应该为前半段为正常获取的数据，后半段为接下来分页的信息，有个href超链接为分页需要使用的信息。

在这里po出我初步看的一些博客链接：

> https://blog.csdn.net/bc_vnetwork/article/details/51692386
>
> https://www.cnblogs.com/donghaiming/p/11007505.html
>
> https://blog.csdn.net/u011521019/article/details/52267495?locationNum=3&fps=1
>
> https://blog.csdn.net/zhong_jay/article/details/91799459

在进行真正的修改和测试之前，我仔细读了一下各个函数的输入输出，修改了keystone project(tenant)的界面demo进行尝试，并且先将generator和字典的用法好好地阅读和复习了一下：

#### 那么generator是什么？

在python的函数（function）定义中，只要出现了yield表达式（Yield expression），那么事实上定义的是一个*generator function*， 调用这个generator function返回值是一个*generator*。我们的_pagination函数符合一个generator函数的定义。遵循迭代器协议，实现next和iter接口；多次进入、多次返回，能够暂停函数体中代码的执行。

#### generator function产生的generator与普通的function有什么区别呢

function每次都是从第一行开始运行，而generator从上一次yield开始的地方运行；

function调用一次返回一个（一组）值，而generator可以多次返回；

function可以被无数次重复调用，而一个generator实例在yield最后一个值 或者return之后就不能继续调用了。

#### generator高级应用

1.Generator可用于产生数据流， generator并不立刻产生返回值，而是等到被需要的时候才会产生返回值，相当于一个主动拉取的过程(pull)，比如现在有一个日志文件，每行产生一条记录，对于每一条记录，不同部门的人可能处理方式不同，但是我们可以提供一个公用的、按需生成的数据流。**数据只有在需要的时候去拉取的，而不是提前准备好**。
2.一些编程场景中，一件事情可能需要执行一部分逻辑，然后等待一段时间、或者等待某个异步的结果、或者等待某个状态，然后继续执行另一部分逻辑。比如微服务架构中，服务A执行了一段逻辑之后，去服务B请求一些数据，然后在服务A上继续执行。或者在游戏编程中，一个技能分成分多段，先执行一部分动作（效果），然后等待一段时间，然后再继续。对于这种需要等待、而又不希望阻塞的情况，我们一般使用回调（callback）的方式。下面举一个简单的例子：

```python
def do(a):
    print ('do', a)
    CallBackMgr.callback(5, lambda a = a: post_do(a))
 
def post_do(a):
    print ('post_do', a)
```

这里的CallBackMgr注册了一个5s后的时间，5s之后再调用lambda函数，可见**一段逻辑被分裂到两个函数，而且还需要上下文的传递**（如这里的参数a）。我们用yield来修改一下这个例子，yield返回值代表等待的时间。

```python
@yield_dec
def do(a):
    print ('do', a)
    yield 5
    print ('post_do', a)
```

这里需要实现一个YieldManager， 通过yield_dec这个decrator将do这个generator注册到YieldManager，并在5s后调用next方法。Yield版本实现了和回调一样的功能，但是看起来要清晰许多。下面给出一个简单的实现以供参考：

```python
# -*- coding:utf-8 -*-
import sys
# import Timer
import types
import time
 
class YieldManager(object):
    def __init__(self, tick_delta = 0.01):
        self.generator_dict = {}
        # self._tick_timer = Timer.addRepeatTimer(tick_delta, lambda: self.tick())
 
    def tick(self):
        cur = time.time()
        for gene, t in self.generator_dict.items():
            if cur >= t:
                self._do_resume_genetator(gene,cur)
 
    def _do_resume_genetator(self,gene, cur ):
        try:
            self.on_generator_excute(gene, cur)
        except StopIteration,e:
            self.remove_generator(gene)
        except Exception, e:
            print 'unexcepet error', type(e)
            self.remove_generator(gene)
 
    def add_generator(self, gen, deadline):
        self.generator_dict[gen] = deadline
 
    def remove_generator(self, gene):
        del self.generator_dict[gene]
 
    def on_generator_excute(self, gen, cur_time = None):
        t = gen.next()
        cur_time = cur_time or time.time()
        self.add_generator(gen, t + cur_time)
 
g_yield_mgr = YieldManager()
 
def yield_dec(func):
    def _inner_func(*args, **kwargs):
        gen = func(*args, **kwargs)
        if type(gen) is types.GeneratorType:
            g_yield_mgr.on_generator_excute(gen)
 
        return gen
    return _inner_func
 
@yield_dec
def do(a):
    print 'do', a
    yield 2.5
    print 'post_do', a
    yield 3
    print 'post_do again', a
 
if __name__ == '__main__':
    do(1)
    for i in range(1, 10):
        print 'simulate a timer, %s seconds passed' % i
        time.sleep(1)
        g_yield_mgr.tick()
```

#### 针对keystone的project list进行了一些实验（即tenant列表）

keystone api写的要比普通的api好理解一些，嗯先从这里入手学习一下。

![image-20200713152954636](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200713152954636.png)

以2个数据为一页，输出的数据：

```
[Wed Jul 15 14:22:38.138745 2020] [:error] [pid 74566:tid 140019505780480] [<Tenant {u'enabled': True, u'description': u'Admin Tenant', u'name': u'admin', u'id': u'a5ab7b87bfc44ee0ac094750f6adec02'}>, <Tenant {u'enabled': True, u'description': u'Service Tenant', u'name': u'service', u'id': u'b65a6a6b59e2431e89e1ac2d6f8b1aa5'}>, <Tenant {u'enabled': True, u'description': u'', u'name': u'ckl', u'id': u'c2698e5a94a34ceba6441e299b560145'}>]
```

输出了第一页的两个和下一页的第一个，第二页（最后一页）只输出当前页的两个。

```
# openstack_dashboard/api/keystone.py

def tenant_list(request, paginate=False, marker=None, domain=None, user=None):
    manager = VERSIONS.get_project_manager(request, admin=True)
    page_size = utils.get_page_size(request) #这里的默认值是20

    # page_size = 2

​    limit = None
​    if paginate:  #分页标识符
​        limit = page_size + 1

​    has_more_data = False
​    if VERSIONS.active < 3:
​        tenants = manager.list(limit, marker)
​        if paginate and len(tenants) > page_size:
​            tenants.pop(-1)
​            has_more_data = True
​    else:
​        tenants = manager.list(domain=domain, user=user)
​    return (tenants, has_more_data)
```



```python
# openstack_dashboard/dashboards/admin/projects/views.py
class IndexView(tables.DataTableView):
    table_class = project_tables.TenantsTable
    template_name = 'admin/projects/index.html'

    # 实现后翻页的定义方法
    # 返回self._more值，该值根据返回的数据数量确定下一页是否有数据，如果有数据该值为True，即页面上显示“向前”，当然相对应的可以编写has_prev_data方法
    def has_more_data(self, table):
        return self._more

    def get_data(self):
        tenants = []
        # marker = self.request.GET.get("tenant_marker", None),在openstack_dashboard/dashboards/admin/projects/tables.py中的class Meta(object)
        # 该值指的是当前页面最后一条tenant的ID值
        marker = self.request.GET.get(
            project_tables.TenantsTable._meta.pagination_param, None)
        domain_context = self.request.session.get('domain_context', None)
        try:
            # paginate 参数用来设置是否采用分页
            tenants, self._more = api.keystone.tenant_list(
                self.request,
                domain=domain_context,
                paginate=True,
                marker=marker)
        except Exception:
            # 设置默认没有下一页
            self._more = False
            exceptions.handle(self.request,
                              _("Unable to retrieve project list."))
        return tenants
```

在原生的/horizon/tables/base.py(class DataTableOptions)和/horizon/tables/views.py(class MultiTableMixin)均定义了相关方法，所以有实现原生horizon模块前后翻页的客观条件，但需要动手写方法。