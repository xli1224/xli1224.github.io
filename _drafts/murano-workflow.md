
# 创建component

获取session：
1. 如果当前http session里面存在session id，则用此id去后台查询。查询到的session是个valid session就返回（valid的定义就是存在，如果是OPEN状态的话没有其他session在部署，如果not valid，会抛错给调用方）。前端会去新建一个session。如果查询到的session状态是ready或者deploy_failure，就删掉再建一个新的。


murano dashboard获取session，进行对session的修改（添加/删除组件），每次添加/删除都会直接调用api接口进行更新（parameters里面是添加的组件的属性）。

```python
api.muranoclient(request).services.post(environment_id,
                                                   path='/',
                                                   data=parameters,
                                                   session_id=session_id)

api.muranoclient(request).services.delete(environment_id,
                                                     '/' + service_id,
                                                     session_id)
```

# API session管理
### 创建session
当前env状态不在deploying或者deleting时，创建一个新session，状态为open，拷贝env的内容。

### add service

### delete service

### 
当前session状态
