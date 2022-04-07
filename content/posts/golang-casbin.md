---
title: "Golang 使用 Casbin 进行权限管理"
date: 2021-03-04T01:18:53+08:00
archives: 
    - 2021
tags:
  - Golang
  - 后端
image: /images/casbin_rabc_dom.jpeg
draft: false
---

先说需求，设计一个基于角色的权限控制系统，满足以下几个规则：
- 单独配置某个角色对某个资源的访问权限
- 一个用户可拥有多个角色
- 可对角色进行禁用或启用
- 各个角色之间的权限为并集，且只要有一个角色有权限，该用户就有权限操作
- 可对一个菜单下的某些重要功能做单独控制，当用户有该菜单的访问权限，没有功能访问权限时，依然能访问该菜单下的非功能接口

[Casbin](https://casbin.org/) 是一个强大的、高效的开源访问控制框架，其权限管理机制支持多种访问控制模型。

具体支持的模型官网有详细描述，回到本例应该使用的模型是RBAC (基于角色的访问控制)。

Casbin使用配置文件来设置访问控制模式。它有两个配置文件，`model.conf`和`policy.csv`。 其中，`model.conf`存储了访问模型，`policy.csv`存储了特定的用户权限配置。

## 模型
Casbin 将访问控制模型抽象为基于PERM(Policy, Effect, Request, Matcher)的一个文件，分别为：
- 策略
- 效果
- 请求
- 匹配器

基于角色的访问控制需要在此基础上加多一个`role_definition`进行角色的定义。

### 请求(request_definition)
该部分用于请求的定义，经典的三元组：访问实体 (Subject)，访问资源 (Object) 和访问方法 (Action)，也可以根据自己的需求进行增加或删除字段。后端常用的控制请求就是`uid, /api/res1, GET`。

```ini
[request_definition]
r = sub, obj, act
```

### 策略(policy_definition)
该部分定义控制策略的模板，哪个实体对哪个资源有怎样的权限。注意：这里的`sub, obj`不一定需要与请求里面的值一致，具体怎样匹配是匹配器定义的。例如这里的一条策略可以是`alice, res1, allow`。

```ini
[policy_definition]
p = sub, obj, eft
```

### 角色定义(role_definition)
该部分定义了角色系统，用户可以具有角色及其继承关系, 资源也可以具有角色及其继承关系。 这两个 RBAC 系统不会互相干扰。这里我使用了三个角色系统：`g`是用户和角色的从属关系；`g2`是资源的从属关系；`g3`是角色的开关。
```ini
[role_definition]
g = _, _
g2 = _, _, _
g3 = _, _
```
举个例子，`g`可以是`user_1, 1`，表示`uid`是 1 的用户拥有角色 1。`g2`可以理解为将多个接口组成一个资源组，角色拥有该资源组的权限则有这些接口的访问权限。
```
/api/order/info/*, resOrder, GET
/api/order/del, resOrder, POST
/api/task/add, resTask, *
```
可以看到上面有两个资源组，只要在策略配置对资源组的访问权限则可批量控制接口的权限了。
最后一个`g3`更易理解，`2, 1`可以表示角色 2 是启用的，反之，`2, 0`则角色 2 被禁用。

### 匹配器(matchers)
顾名思义，匹配器就是定义如何匹配规则的。我这里进行了三个部分的检查：
```ini
[matchers]
m = g(r.sub, p.sub) && g2(r.obj, p.obj, r.act) && g3(p.sub, "1")
```
第一部分`g(r.sub, p.sub)`是检查角色关系的，策略`p`里定义的实体是角色，请求`r`里定义是实体是用户，在角色系统`g`里检查用户角色从属关系。

第二部分`g2(r.obj, p.obj, r.act)`是检查资源关系的，请求`r`里的最后两个元素`r.obj`和`r.act`，在这个例子分别代表请求 url 和请求方法，`p.obj`则是策略里定义的资源组，在角色系统`g2`里检查资源从属关系。

第三部分`g3(p.sub, "1")`是检查角色是否启用，在角色系统`g3`里检查。

### 效果(policy_effect)
策略效果的定义。它确定如果多项策略规则与请求相符，是否应批准访问请求。例如，一项规则允许，另一项规则则加以拒绝。

现在我们考虑实现最后一个需求，可以把非功能接口都放在同一个资源组中，而每个功能用的接口都独立为一个资源组。例如查看属于非功能，编辑和删除属于功能：
```
# 非功能性
/api/order/info, order, GET
/api/order/list, order, GET
# 编辑功能
/api/order/edit, editOrder, POST
# 删除功能
/api/order/del, delOrder, POST
```
这样，当角色有菜单权限时，则赋予`order`资源权限；当有编辑功能权限时，则赋予`editOrder`资源权限。此时需要策略效果是，如果有任何匹配的策略规则允许, 最终效果是允许。
```ini
[policy_effect]
e = some(where (p.eft == allow))
```
而且这种策略效果和倒数第二点需求相呼应。

要说不足的地方，可能是每次添加接口，都需要把该接口添加到规则里面。当然你可以用通配符，但非功能性接口可能比较零散，很多接口一开始已经写好了，很难有`/api/order/public/*`这种匹配所有非功能性。

这里我们也可以换一个思路，先匹配该菜单下的所有接口，再检查功能性接口是否有权限。
```
# 菜单入口
/api/order/*, order, *
# 编辑功能
/api/order/edit, editOrder, POST
# 删除功能
/api/order/del, delOrder, POST
```
这时对于编辑功能，它既匹配菜单入口，也匹配编辑功能。当所有匹配的都允许，没有一个拒绝时，最终效果是允许。
```ini
[policy_effect]
e = some(where (p.eft == allow)) && !some(where (p.eft == deny))
```
这意味着至少有一个匹配的策略规则允许，并且没有匹配的否定的策略规则。

但这样做的话倒数第二点需求就无法满足了，当一个角色有权限，另一个无权限，按照这种效果判断是拒绝的。如果继续这种思路，我们需要自定义效果决策。

使用分组的思想，角色内的效果决策我们依然选择上面的决策，但角色之间的效果决策我们却使用第一种效果决策，有点像两种方法的结合。
```ini
[policy_effect]
e = some(where (p.eft == allow)) && !some(where (p.eft == deny)) / some(where (p.eft == allow))
```
注意这个分隔符`/`，这是我们自己自定义的，分隔符左边代表角色内的决策，分隔符右边代表角色之间的决策。然后我们需要自己实现这个分组决策器。
```go
package acl

import (
	"github.com/casbin/casbin/v2"
	"github.com/casbin/casbin/v2/effector"
	"log"
	"strings"
)

func Partition(s string, sep string) (string, string, string) {
	parts := strings.SplitN(s, sep, 2)
	if len(parts) == 1 {
		return parts[0], "", ""
	}
	return parts[0], sep, parts[1]
}

type GroupEffector struct {
	defaultEffector effector.DefaultEffector
	enforcer        *casbin.Enforcer
	innerExpr       string // 角色内策略
	interExpr       string // 角色间策略
}

func NewGroupEffector(e *casbin.Enforcer) *GroupEffector {
	obj := &GroupEffector{}
	obj.enforcer = e
	lExpr, sep, rExpr := Partition(e.GetModel()["e"]["e"].Value, "/")
	if sep == "" {
		log.Fatal("invalid effector expression")
		return nil
	}
	obj.innerExpr = strings.TrimSpace(lExpr)
	obj.interExpr = strings.TrimSpace(rExpr)
	return obj
}

func (g *GroupEffector) MergeEffects(expr string, effects []effector.Effect, matches []float64, policyIndex int, policyLength int) (effector.Effect, int, error) {
	if policyIndex < policyLength-1 {
		return effector.Indeterminate, -1, nil
	}
	policy := g.enforcer.GetModel()["p"]["p"].Policy
	// 对策略按角色分组
	groupResult := make(map[string][]int)
	for i := range effects {
		if matches[i] == 0 {
			continue
		}
		group := policy[i][0]
		if idx, ok := groupResult[group]; ok {
			groupResult[group] = append(idx, i)
		} else {
			groupResult[group] = []int{i}
		}
	}
	interEffects := make([]effector.Effect, 0, len(groupResult))
	interMatches := make([]float64, 0, len(groupResult))
	// 组内策略决策
	for _, idx := range groupResult {
		groupEffects := make([]effector.Effect, len(idx))
		groupMatches := make([]float64, len(idx))
		for i, j := range idx {
			groupEffects[i] = effects[j]
			groupMatches[i] = matches[j]
		}
		r, e, err := g.defaultEffector.MergeEffects(g.innerExpr, groupEffects, groupMatches, len(groupEffects)-1, len(groupEffects))
		if err != nil {
			return r, e, err
		}
		interEffects = append(interEffects, r)
		if r != effector.Indeterminate {
			interMatches = append(interMatches, 1)
		} else {
			interMatches = append(interMatches, 0)
		}
	}
	// 组间策略决策
	r, _, err := g.defaultEffector.MergeEffects(g.interExpr, interEffects, interMatches, len(interEffects)-1, len(interEffects))
	return r, -1, err
}
```

## 策略
简单来说，上面的模型像是定义了一种模板，这里的策略就像具体的数据了。这些策略可以存储在文件中，比如`csv`文件，也可以存储在数据库表中。对于后端开发最常用的就是数据库了。

表的结构类似这样：
|id	|ptype|	v0|	v1|	v2|	v3|	v4|	v5 |
|---|----|---|---|---|---|---|-----|
|1	|p	|1	|order|	allow	|		
|2	|p	|1	|editOrder|	deny	|		
|3	|g	|user_1	|1       |	   |
|4  |g2 |/api/order/*| order| *|
|5  |g2 |/api/order/edit| editOrder| POST|
|6	|g3	|1	|1       |	   |

把模型的数据都写到这张表中，Casbin有接口可以方便地操作这个策略表。

## 如何使用
具体流程比较简单，在程序启动时设定模型的定义信息，初始化策略存储的适配器，利用 Casbin 的接口创建`Enforcer`，后续就可以通过`Enforcer`进行权限管理了。

1. 创建策略存储适配器

在 Casbin 中，策略存储作为适配器实现，文档里有目前支持的[适配器列表](https://casbin.org/docs/zh-CN/adapters)。本例使用的是 Gorm Adapter。
```go
if adapter, err = gormadapter.NewAdapterByDB(yourDB); err != nil {
	log.Fatal(err)
	return
}
```
2. 创建模型

将上面定义的模型写到字符串中，使用 Casbin 的接口创建模型。
```go
const rbac = `[request_definition]
...
`

if acModel, err = model.NewModelFromString(rbac); err != nil {
	log.Fatal(err)
	return
}
```
3. 创建执行器

后面就可以通过调用这个执行器的方法操作策略了，比如添加/删除/更新角色和策略规则之类的。
```go
if Enforcer, err = casbin.NewEnforcer(); err != nil {
	log.Fatal(err)
	return
}

if err = Enforcer.InitWithModelAndAdapter(acModel, adapter); err != nil {
	log.Fatal(err)
	return
}
```

4. 初始化设置

对执行器执行一些初始化设置，比如加载策略，设定自定义的效果器等。
```go
Enforcer.AddNamedMatchingFunc("g2", "KeyMatch2", util.KeyMatch2)
Enforcer.AddNamedDomainMatchingFunc("g2", "KeyMatch2", util.KeyMatch2)

if err = Enforcer.BuildRoleLinks(); err != nil {
	log.Fatal(err)
	return
}

if err = Enforcer.LoadPolicy(); err != nil {
	log.Fatal(err)
	return
}

Enforcer.SetEffector(NewGroupEffector(Enforcer))
```
