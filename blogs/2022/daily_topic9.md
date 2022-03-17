---
title: k8s 每日一问 - helm -f 是 merge 默认values.yaml值 而不是 replace ?
date: '2022-03-17 18:21:31'
sidebar: false
categories:
 - 技术
tags:
 - 每日一问
publish: true
---


## **helm -f** 是**merge** 默认values.yaml值 而不是 **replace ?**

**是 merge**

helm 优先级

`-f/--values` < `--set` < `--set-string` < `--set-file`

1. `RunWithContext`   
2. `valuesToRender, err := chartutil.ToRenderValues(chrt, vals, options, caps)`
3. `vals, err := CoalesceValues(chrt, chrtVals)`
4. `coalesceValues(printf, ch, dest, prefix)`
5. `c.Values` 就是 charts 的 默认 `values.yaml`

```go
func coalesceValues(printf printFn, c *chart.Chart, v map[string]interface{}, prefix string) {
	subPrefix := concatPrefix(prefix, c.Metadata.Name)
	for key, val := range c.Values {
		if value, ok := v[key]; ok {
			if value == nil {
				delete(v, key)
			} else if dest, ok := value.(map[string]interface{}); ok {
				src, ok := val.(map[string]interface{})
				if !ok {
					if val != nil {
						printf("warning: skipped value for %s.%s: Not a table.", subPrefix, key)
					}
				} else {
					coalesceTablesFullKey(printf, dest, src, concatPrefix(subPrefix, key))
				}
			}
		} else {
			v[key] = val
		}
	}
}
```


