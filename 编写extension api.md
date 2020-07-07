### 编写extension api接收k8s cni相关的pod信息：

关联neutron_server，所以需要tenant_id的处理，使用与networkbindings表的外键关系，建立父子表关联，解决级联删除问题和tenant_id的取值问题

#### 编写/neutron/extensions/cni_info.py

完成相关数据库表定义。全部允许post

#### 编写/neutron/db/cni_info_db.py

对pod_name这一字段定义外键关系，必将其设为主键

```
pod_name = sa.Column(sa.String(255),                    sa.ForeignKey('networkbindings.pod_name', ondelete="CASCADE"),                      nullable=False, primary_key=True)
    nb = orm.relationship(
        networkbindings_db.NetworkBinding, backref=orm.backref("cni_info",                    lazy='joined',uselist=False,                      cascade='delete')
    )
```

编写基本的增删改查逻辑；

#### 修改/neutron/plugins/ml2/plugin.py

继承CniInfoMixin类，即定义crud的方法

```
class Ml2Plugin(db_base_plugin.NeutronDbPlugin,
                AgentSchedulerDbMixin,
                External_net_db_mixin,
                NetworkBindingMixin,
                CniInfoMixin,
                ):
```

```
# add a new attribute
_supported_extension_aliases = ['external-net', 'port-bindings', 'agent', "network-bindings", "cni_infos"]
```

重新启动neutron-server

```
service neutron-server restart
```

进行post/get操作（已关闭keystone，若开启keystone，需要先token-get）
![image-20200707170609903](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200707170609903.png)
