---
title: php filter
date: "2019-10-02 10:58:30"
sidebar: false
categories:
 - 技术
tags:
 - php
 - filter
publish: true
---

```php
<?php

namespace App\Filters;

use Illuminate\Support\Str;
use Illuminate\Database\Eloquent\Builder;
use Mainto\RpcServer\RpcServer\RpcContext;

/**
 * Class Filters
 * @package App\Filters
 */
abstract class Filter
{
    /**
     * @var bool
     */
    protected $usePrefix = false;

    /**
     * @var string
     */
    protected $prefix = '';

    /**
     * @var RpcContext
     */
    protected $request;

    /**
     * @var Builder
     */
    protected $builder;

    /**
     * @var array
     */
    protected $filters = [];

    /**
     * Filters constructor.
     * @param RpcContext $request
     */
    public function __construct(RpcContext $request)
    {
        $this->request = $request;
    }

    /**
     * @param Builder $builder
     * @return Builder
     *
     * @author 神符 <1025434218@qq.com>
     */
    public function apply(Builder $builder)
    {
        $this->builder = $builder;

        foreach ($this->getFilters() as $filter => $value) {
            $method = $this->usePrefix ? Str::camel(Str::after($filter, $this->prefix)) : Str::camel($filter);
            if (method_exists($this, $method)) {
                $this->{$method}($value);
            }
        }

        return $this->builder;
    }

    /**
     * @return array
     *
     * @author 神符 <1025434218@qq.com>
     */
    public function getFilters()
    {
        return array_filter($this->request->only($this->getKeys()), function ($item) {
            return $item !== '';
        });
    }

    /**
     * @return array
     *
     * @author 神符 <1025434218@qq.com>
     */
    private function getKeys(): array
    {
        if ($this->usePrefix) {
            $prefix = Str::endsWith($this->prefix, '_') ? $this->prefix : $this->prefix . '_';

            return array_map(function ($key) use ($prefix) {return $prefix . $key;}, $this->filters);
        }

        return $this->filters;
    }

    /**
     * @param array|mixed $fields
     * @return $this
     *
     * @author 神符 <1025434218@qq.com>
     */
    public function only($fields)
    {
        $fields = is_array($fields) ? $fields : func_get_args();

        $callback = function ($field) use ($fields) {
            return in_array($field, $fields);
        };

        $this->filters = array_filter($this->filters, $callback);

        return $this;
    }

    /**
     * @param string $prefix
     * @return $this
     *
     * @author 神符 <1025434218@qq.com>
     */
    public function withPrefix(string $prefix = '')
    {
        $this->prefix = $prefix ?: $this->prefix;
        $this->usePrefix = true;

        return $this;
    }
}

```