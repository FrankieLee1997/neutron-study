在honeystack的对应egg目录下

/usr/local/lib/python2.7/dist-packages/honeystack_neutron-1.0-py2.7.egg/plugins/ml2/drivers

有多种可用type driver可供加载

在/usr/local/lib/python2.7/dist-packages/honeystack_neutron-1.0-py2.7.egg/EGG-INFO中

entry_points.txt文件定义了ml2的type driver，

[neutron.ml2.type_drivers]
flat = neutron.plugins.ml2.drivers.type_flat:FlatTypeDriver
vxlan = neutron.plugins.ml2.drivers.type_vxlan:VxlanTypeDriver

