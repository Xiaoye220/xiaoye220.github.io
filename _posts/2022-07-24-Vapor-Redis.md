---
layout: post
title: Vapor - Redis
date: 2022-07-24
tags: [Vapor]
excerpt_separator: <!--more-->
toc: true
---

Additions to the  [Vapor Doc - Redis](https://docs.vapor.codes/redis/overview/) , refer to [doc of redis.io](https://redis.io/docs/getting-started/)

<!--more-->



### 1. Installation

```bash
brew install redis
```



### 2. Start

```
brew services start redis
```



### 3. Check info

```
brew services info redis
```



### 4. Stop

```
brew services stop redis
```



### 5. Connect

```
redis-cli
```



### 6. Exploring Redis with the CLI

```
$ redis-cli
redis 127.0.0.1:6379> ping
PONG
redis 127.0.0.1:6379> set mykey somevalue
OK
redis 127.0.0.1:6379> get mykey
"somevalue"
```

