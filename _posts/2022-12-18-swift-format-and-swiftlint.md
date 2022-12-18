---
layout: post
title: swift-format and swiftlint
date: 2022-12-18
tags: [iOS, Swift, Notes]
excerpt_separator: <!--more-->
toc: true
---

How to start working with swift-format and swiftlint.

<!--more-->

### 1. swift-format

#### 1.1 Gets swift-format

swift-format is hosted on [apple/swift-format](https://github.com/apple/swift-format), build executable by following the [README](https://github.com/apple/swift-format#getting-swift-format)

```bash
VERSION=0.50700.0  # replace this with the version you need
git clone https://github.com/apple/swift-format.git
cd swift-format
git checkout "tags/$VERSION"
swift build -c release
```

the `swift-format` executable will be located at `.build/release/swift-format`.



#### 1.2 Place swift-format executable to global path

```bash
# Place swift-format executable to the global path
ln -s /path/to/your/binary/swift-format /usr/local/bin/swift-format
```



#### 1.3 Adds configuration file

Adds configuration file `.swift-format` to customize the rules from the project root dir. Refer to [Configuration](https://github.com/apple/swift-format/blob/main/Documentation/Configuration.md).

```
{
   "indentation" : {
      "spaces" : 2
   }
}
```



#### 1.4 Xcode Usage

Add a new `Run Script` to the `Build Phases`  section

```shell
if which swift-format >/dev/null; then
    swift-format format -i -r --configuration ${PROJECT_DIR}/.swift-format ${PROJECT_DIR}
else
    echo "warning: swift-format not installed"
fi
```



### 2. swiftlint

#### 2.1 Gets swiftlint

swiftlint is hosted on [realm/SwiftLint](https://github.com/realm/SwiftLint), install it by following the [README](https://github.com/realm/SwiftLint#installation)

```bash
# I prefer to use homebrew
brew install swiftlint
```



#### 2.2 Xcode Usage

Add a new `Run Script` to the `Build Phases`  section

```shell
if which swiftlint > /dev/null; then
  swiftlint
else
  echo "warning: SwiftLint not installed
fi
```



### 3. Troubleshotting

When using `swift-format` and `swiftlint` at the same time, some rules may conflict. So you can disable some `swiftlint` rules to avoid warnings.

Adds file `.swiftlint.yml` from the directory you'll run SwiftLint from. Then you can configure it by following this [guideline](https://github.com/realm/SwiftLint#configuration)

```yml
# By default, SwiftLint uses a set of sensible default rules you can adjust:
disabled_rules: # rule identifiers turned on by default to exclude from running
  - trailing_comma # against swift-format
  - opening_brace  # against swift-format
```

