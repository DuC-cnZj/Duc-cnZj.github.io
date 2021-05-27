---
title: laravel 源码解析之 validater
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - laravel
tags:
 - 源码解析
 - laravel
 - validater
publish: true
---


`make` 之后的

```php
$validator = $this->resolve(
    $data, $rules, $messages, $customAttributes
);
```

初始化了`validator`

```php
return new Validator($this->translator, $data, $rules, $messages, $customAttributes);
```

其中

```php
$this->setRules($rules);
public function addRules($rules)
{
  // The primary purpose of this parser is to expand any "*" rules to the all
  // of the explicit rules needed for the given data. For example the rule
  // names.* would get expanded to names.0, names.1, etc. for this data.
‼️  很关键，设置了类的 implicitAttributes 属性
  explode 方法解析了以下的格式
  "data" => [
      0 => [
         "name" => "duc"
         "age" => 1
      ],

      1 => [
         "name" => "abc"
         "age" => 1
      ]
  ]
  解析后是
    "data.0.name" => 'duc',
    "data.0.age" => 1,
    "data.0.name" => 'abc',
    "data.0.age" => 1,
  最终设置了 implicitAttributes 属性
$implicitAttributes = [
  'data.*.name' => [
    "data.0.name",
    "data.1.name",
  ]  
]
 也设置了 rules 属性
$rules = [
	'data.0.name' => 'required',
	'data.1.name' => 'required',
]
    
rules 属性用于 passes() 方法的循环，
这里还用了 Arr::dot() 用 . 压平数组
  
  
  
  $response = (new ValidationRuleParser($this->data))
    ->explode($rules);

  $this->rules = array_merge_recursive(
    $this->rules, $response->rules
  );

  $this->implicitAttributes = array_merge(
    $this->implicitAttributes, $response->implicitAttributes
  );
}
```






```php
public function passes()
{
  $this->messages = new MessageBag;

  [$this->distinctValues, $this->failedRules] = [[], []];

  foreach ($this->rules as $attribute => $rules) {
    // name\.data 转换成 name->data
    $attribute = str_replace('\.', '->', $attribute);

    foreach ($rules as $rule) {
      $this->validateAttribute($attribute, $rule);

      // 是否带有 Bail 规则，或者文件上传失败
      if ($this->shouldStopValidating($attribute)) {
        break;
      }
    }
  }

  // 调用回调函数 第一个参数是 Validator 实例
  foreach ($this->after as $after) {
    $after();
  }

  return $this->messages->isEmpty();
}
```



```php
protected function validateAttribute($attribute, $rule)
    {
        $this->currentRule = $rule;

  // 把 required 转换成 Required ，因为后面有 "validate{$method}" => validateRequired()方法
        [$rule, $parameters] = ValidationRuleParser::parse($rule);

        if ($rule == '') {
            return;
        }

  				// 首先，如果字段嵌套在其中，我们将获得给定属性的正确键数组。 
         // 然后，我们确定给定的规则是否接受其他字段名称作为参数。
         //如果是这样，我们将使用正确的键替换在参数中找到的所有星号。
  			// 看 $implicitAttributes 属性👇
        if (($keys = $this->getExplicitKeys($attribute)) &&
            $this->dependsOnOtherFields($rule)) {
            $parameters = $this->replaceAsterisksInParameters($parameters, $keys);
        }

        $value = $this->getValue($attribute);

				 //如果属性是文件，我们将验证文件上传是否成功
         //如果不是，我们将为该属性添加一个失败。 文件可能无法成功
         //根据PHP的设置，如果它们太大，请上传，因此在这种情况下，我们将保释。
        if ($value instanceof UploadedFile && ! $value->isValid() &&
            $this->hasRule($attribute, array_merge($this->fileRules, $this->implicitRules))
        ) {
            return $this->addFailure($attribute, 'uploaded', []);
        }

				 //如果到目前为止，我们将确保该属性是有效的，如果是
         //我们将使用属性调用验证方法。 如果方法返回false，则
         //属性无效，我们将为此失败的属性添加一条失败消息。
        $validatable = $this->isValidatable($rule, $attribute, $value);

        if ($rule instanceof RuleContract) {
            return $validatable
                    ? $this->validateUsingCustomRule($attribute, $value, $rule)
                    : null;
        }

        $method = "validate{$rule}";

        if ($validatable && ! $this->$method($attribute, $value, $parameters, $this)) {
            $this->addFailure($attribute, $rule, $parameters);
        }
    }
```

## `replacers` 属性

```php
replacers 属性 会替换 对应规则的 message
比如
$v->addReplacer('string', 'dddd');

string 规则失败的时候就会返回 dddd
```

## `extensions` 属性

> 当没有验证规则时触发，提供了可扩展的能力

```php
// $parameters => [$attribute, $value, $parameters, $this] 分别是 你上传的 (key value 规则类似于 in:a,b,c 的 [a,b,c], $validator)
public function __call($method, $parameters)
{
  $rule = Str::snake(substr($method, 8));

  if (isset($this->extensions[$rule])) {
    return $this->callExtension($rule, $parameters);
  }

  throw new BadMethodCallException(sprintf(
    'Method %s::%s does not exist.', static::class, $method
  ));
}
```

## `$implicitAttributes` 属性

> 存下通配符

```php
// 该属性会存符合 name.* 的所有key
'name.*' => 'RequiredWith:*,a.*|duc|required|string',

$v = validator(
  [
    'name' => [
      "data" => 'dada',
      "q"    => 'qqq',
    ],
  ],
  [
    'name.*' => 'RequiredWith:*,a.*|duc|required|string',
  ],
  [
    'name.required' => 'name必填',
  ]
);

// name.data => 依赖 data, a.data
```



## `distinctValues` 在 `distinct`规则时用到


> Distinct



## `dependentRules` 属性

> 用来做依赖验证用的属性

```php
// 用在这里，来替换带了*的属性, 比如我自定义规则 duc:a.*,b.*, $keys = ["data"], 那么这里就会进入判断，把$parameters属性变成 [a.data, b.data]
if (($keys = $this->getExplicitKeys($attribute)) &&
    $this->dependsOnOtherFields($rule)) {
  $parameters = $this->replaceAsterisksInParameters($parameters, $keys);
}
```



## 验证后钩子

```php
$v->after(function (Validator $v) {
    dump(1, $v->messages()->messages());
});
```



## 提供了自定义翻译器的能力 `translator`



## 测试用的代码🧐

```php
/** @var Validator $v */
$v = validator(
  [
    'name' => [
      "data" => 'dada',
      "q"    => 'qqq',
    ],
  ],
  [
    'name.*' => 'duc:*,a.*|duc|required|string',
  ],
  [
    'name.required' => 'name必填',
  ]
);

$v->addReplacer('string', function () {
  return "dadada";
});

$v->addExtension('duc_check', function () {
  return true;
});

$v->addDependentExtension('duc', function ($attribute, $value, $parameters, $v) {
  dd([$attribute, $value, $parameters,$v]);
  // 做你的依赖验证
  return true;
});

$v->after(function (Validator $v) {
  dump(1, $v->messages()->messages());
});

$res = $v->validate();

if ($v->fails()) {
  dd($v->failed(), $v->getMessageBag());
}
```


