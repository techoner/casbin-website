---
id: syntax-for-models
title: Syntax for Models
---

- A model CONF should have at least four sections: ``[request_definition], [policy_definition], [policy_effect], [matchers]``.

- If a model uses RBAC, it should also add the ``[role_definition]`` section.

- A model CONF can contain comments. The comments start with ``#``, and ``#`` will comment the rest of the line.

## Request definition

``[request_definition]`` is the definition for the access request. It defines the arguments in ``e.Enforce(...)`` function.

```ini
[request_definition]
r = sub, obj, act
```

``sub, obj, act`` represents the classic triple: accessing entity (Subject), accessed resource (Object) and the access method (Action). However, you can customize your own request form, like ``sub, act`` if you don't need to specify an particular resource, or ``sub, sub2, obj, act`` if you somehow have two accessing entities.

## Policy definition

``[policy_definition]`` is the definition for the policy. It defines the meaning of the policy. For example, we have the following model:

```ini
[policy_definition]
p = sub, obj, act
p2 = sub, act
```

And we have the following policy (if in a policy file)

```
p, alice, data1, read
p2, bob, write-all-objects
```

Each line in a policy is called a policy rule. Each policy rule starts with a ``policy type``, e.g., `p`, `p2`. It is used to match the policy definition if there are multiple definitions. The above policy shows the following binding. The binding can be used in the matcher.

```
(alice, data1, read) -> (p.sub, p.obj, p.act)
(bob, write-all-objects) -> (p2.sub, p2.act)
```

**Note**: Currently only single policy definition ``p`` is supported. ``p2`` is yet not supported. Because for common cases, our user doesn't have the need to use multiple policy definitions. If your requirement has to use additional policy definitions, please send an issue about it.

**Note 2**: The elements in a policy rule are always regarded as``string``. If you have any question about this, please see the discussion at: https://github.com/casbin/casbin/issues/113

## Policy effect

``[policy_effect]`` is the definition for the policy effect. It defines whether the access request should be approved if multiple policy rules match the request. For example, one rule permits and the other denies.
    
```ini
[policy_effect]
e = some(where (p.eft == allow))
```

The above policy effect means if there's any matched policy rule of ``allow``, the final effect is ``allow`` (aka allow-override). ``p.eft`` is the effect for a policy, it can be ``allow`` or ``deny``. It's optional and the default value is ``allow``. So as we didn't specify it above, it uses the default value.

Another example for policy effect is:

```ini
[policy_effect]
e = !some(where (p.eft == deny))
```

It means if there's no matched policy rules of``deny``, the final effect is ``allow`` (aka deny-override). ``some`` means: if there exists one matched policy rule. ``any`` means: all matched policy rules (not used here). The policy effect can even be connected with logic expressions:

```ini
[policy_effect]
e = some(where (p.eft == allow)) && !some(where (p.eft == deny))
```

It means at least one matched policy rule of``allow``, and there is no matched policy rule of``deny``. So in this way, both the allow and deny authorizations are supported, and the deny overrides.

## Matchers

``[matchers]`` is the definition for policy matchers. The matchers are expressions. It defines how the policy rules are evaluated against the request.

```ini
[matchers]
m = r.sub == p.sub && r.obj == p.obj && r.act == p.act
```

The above matcher is the simplest, it means that the subject, object and action in a request should match the ones in a policy rule.

You can use arithmetic like ``+, -, *, /`` and logical operators like ``&&, ||, !`` in matchers.

**Note**: Although it seems like there will be multiple matchers such as ``m1``, ``m2`` like other primitives, currently, we only support one matcher ``m``. You can always use the above logical operators to implement complicated logic judgment in one matcher. So we believe there is no need to support multiple matchers for now. Let me know if you have other opinions.

### Functions in matchers

You can even specify functions in a matcher. You can use the built-in functions or specify your own function. The supported built-in functions are:

Function | Meaning | Example
----|------|----
keyMatch(arg1, arg2) | arg1 is a URL path like ``/alice_data/resource1``. arg2 can be a URL path or a ``*`` pattern like ``/alice_data/*``. It returns whether arg1 matches arg2 | [keymatch_model.conf](https://github.com/casbin/casbin/blob/master/examples/keymatch_model.conf)/[keymatch_policy.csv](https://github.com/casbin/casbin/blob/master/examples/keymatch_policy.csv)
keyMatch2(arg1, arg2) | arg1 is a URL path like ``/alice_data/resource1``. arg2 can be a URL path or a ``:`` pattern like ``/alice_data/:resource``. It returns whether arg1 matches arg2. | [keymatch2_model.conf](https://github.com/casbin/casbin/blob/master/examples/keymatch2_model.conf)/[keymatch2_policy.csv](https://github.com/casbin/casbin/blob/master/examples/keymatch2_policy.csv)
regexMatch(arg1, arg2) | arg1 can be any string. arg2 is a regular expression. It returns whether arg1 matches arg2. | [keymatch_model.conf](https://github.com/casbin/casbin/blob/master/examples/keymatch_model.conf)/[keymatch_policy.csv](https://github.com/casbin/casbin/blob/master/examples/keymatch_policy.csv)
ipMatch(arg1, arg2) | arg1 is an IP address like ``192.168.2.123``. arg2 can be an IP address or a CIDR like ``192.168.2.0/24``. It returns whether arg1 matches arg2. | [ipmatch_model.conf](https://github.com/casbin/casbin/blob/master/examples/ipmatch_model.conf)/[ipmatch_policy.csv](https://github.com/casbin/casbin/blob/master/examples/ipmatch_policy.csv)

### How to add a customized function

First prepare your function. It takes several parameters and return a bool:

```go
func KeyMatch(key1 string, key2 string) bool {
	i := strings.Index(key2, "*")
	if i == -1 {
		return key1 == key2
	}

	if len(key1) > i {
		return key1[:i] == key2[:i]
	}
	return key1 == key2[:i]
}
```

Then wrap it with ``interface{}`` types:

```go
func KeyMatchFunc(args ...interface{}) (interface{}, error) {
	name1 := args[0].(string)
	name2 := args[1].(string)

	return (bool)(KeyMatch(name1, name2)), nil
}
```

At last, register the function to the Casbin enforcer:

```go
e.AddFunction("my_func", KeyMatchFunc)
```

Now, you can use the function in your model CONF like this:

```ini
[matchers]
m = r.sub == p.sub && my_func(r.obj, p.obj) && r.act == p.act
```
