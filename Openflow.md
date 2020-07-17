### FlowMod

Controller向Switch下达规则的讯息。依靠他。我们的规则得以下发至switch

```
# OpenFlow 1.3
mod = datapath.ofproto_parser.OFPFlowMod(
			datapath=datapath, cookie=0, cookie_mask=0, table_id=table,
			command=datapath.ofproto.OFPFC_ADD, idle_timeout=0, hard_timeout=0,
			priority=priority, buffer_id=buffer_id,
			out_port=datapath.ofproto.OFPP_ANY, out_group=datapath.ofproto.OFPG_ANY,
			flags=0, match=match, instructions=inst)

datapath.send_msg(mod)
```

生成一个FlowMod，需要大量的参数，其中包含了这条规则的Match和Action

### PacketIn

当switch 遇到不知道怎么处理的包，就转送controller。实际运用上，可以利用此事件，进行未知封包的学习

### PacketOut

可以直接进行包的转送。以此方式送出的包，会直接进行设定的Action。实际运用上，遇到未知包时，最后处理的方式，可能会通过它来进行封包的转送，例如flooding或是直接将包导向指定的port上

> https://www.cnblogs.com/goldsunshine/p/7262484.html?utm_source=tuicool&utm_medium=referral

![img](https://images2017.cnblogs.com/blog/1060878/201707/1060878-20170731172427458-1079756420.png)

### 使用 OpenFlow 的 Switch

在能运行Openflow的Switch中,有两个主要单元:

Switch Agent, Data Plane

#### Switch Agent

负责Switch和Controller的沟通,并解析OpenFlow协定,进而将解析出的规则下发至Data Plane中执行

#### Data Plane

在 Data Plane 部分，由以下项目完成整体的运行：

Ports，Flow Tables（存放规则），Flows（Match+Action），Classifiers（对比Flow并进行相应动作，无符合转送Controller）

> ```
> Ports(in) -> Flow Tables -> Classifiers -> (Match) -> Flows -> (Action) -> Ports(out)
> ```

#### 封包在 Data Plane 中的 Lifecycle

封包在到达 Switch 后，将会建立出一个代表此封包的 **Key**，并转送至 Table 中，搜寻对应的 Flow ，并执行其中的 Action。也因为特定 Action 可以将封包转往其它的 Table 进行下一步的规则对应，所以封包有可能在 Action 后离开 Switch，也有可能是转往下一個个Table 进行下一步的转送规划。

#### Lifecycle

> ```
> 封包到达 -> 取出特定封包资讯产生 Key -> 进入 Flow Table -> 搜寻对应规则 -> 执行 Action 离开 Switch，或转往其他 Flow Table
> ```

### 利用Controller控制规则

例：switch *1, host *3

需求主要是对规则的增删以及EventOFPPortStateChange事件

#### 新增

要对 Switch 新增规则，是需要由 Controller（Ryu）通过传送`OFPFlowMod`给 Switch，Switch 收到此讯息后，则将对应的规则加入。

```python
def add_flow(self, dp, match=None, inst=[], table=0, priority=32768):
	ofp = dp.ofproto # dp:指定的switch
	ofp_parser = dp.ofproto_parser

	buffer_id = ofp.OFP_NO_BUFFER

    # inst:Match规则后将执行的动作
	mod = ofp_parser.OFPFlowMod(
		datapath=dp, cookie=0, cookie_mask=0, table_id=table,
		command=ofp.OFPFC_ADD, idle_timeout=0, hard_timeout=0,
		priority=priority, buffer_id=buffer_id,
		out_port=ofp.OFPP_ANY, out_group=ofp.OFPG_ANY,
		flags=0, match=match, instructions=inst)

	dp.send_msg(mod)
```

#### 删除

和新增方式的差别在于command参数的给定。在删除中，给定的是ofp.OFPFC_DELETE，新增则是ofp.OFPFC_ADD。可以通过指定的Match条件及所在的table进行删除。

```python
def del_flow(self, dp, match, table):
	ofp = dp.ofproto
	ofp_parser = dp.ofproto_parser

	mod = ofp_parser.OFPFlowMod(datapath=dp,
		command=ofp.OFPFC_DELETE,
		out_port=ofp.OFPP_ANY,
		out_group=ofp.OFPG_ANY,
		match=match,
		table_id=table)

	dp.send_msg(mod)

```

### 最短路径规划

> https://github.com/YanHaoChen/Learning-SDN/tree/master/Controller/Ryu/ShortestPath