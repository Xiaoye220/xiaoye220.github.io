---
layout: post
title: iOS Package Manager
date: 2022-01-19
Author: Xiaoye
tags: [Notes, Learning day]
excerpt_separator: <!--more-->
toc: true
---

Some notes about iOS package manager tools, CocoaPods, Swift Package Manager, Carthage

<!--more-->

### 1. CocoaPods

#### 1.1 [Guidance of CocoaPods](https://cocoapods.org/)

#### 1.2 Install 

```ruby
sudo gem install cocoapods
```

#### 1.3 Import dependencies

Create a text file named `Podfile` in your Xcode project directory

```ruby
# Podfile
platform :ios, '8.0'
use_frameworks!

target 'MyApp' do
  pod 'AFNetworking', '~> 2.6'
end
```

Then run

```ruby
pod install
```

#### 1.4 Upload library to CocoaPods specs

##### 1.4.1 Register trunk

```ruby
()# Register trunk
pod trunk register xxxx@microsoft.com 'Xiaoye__220' --verbose

# You can list your sessions by running 
pod trunk me
```

##### 1.4.2 Create podspec file under the project root folder

```ruby
# Replace 'MyLib' with your lib name
pod spec create [MyLib]
```

##### 1.4.3 Edit podspec file

Refer to [Specs and the Specs Repo](https://guides.cocoapods.org/making/specs-and-specs-repo.html) for more configuration. 

Note: Make sure you had add the `git tag` with same `spec.version` to your `spec.source`

##### 1.4.4 Validate podspec file

```ruby
pod spec lint [NAME.podspec]
```

##### 1.4.5 Upload library

```ruby
pod trunk push [NAME.podspec]
```

##### 1.4.6 Update library

Push your changes and add new tag like 

```ruby
git add tag '1.0.1'
git push -tags
```

Then push again

```ruby
pod trunk push [NAME.podspec]
```

##### 1.4.7 Remove library

```ruby
# pod trunk delete [NAME] [VERSION]
pod trunk delete [MyLib] [1.0.0]

# pod trunk deprecate [NAME]
pod trunk deprecate [MyLib]
```

#### 1.4 Upload library to private specs repo

##### 1.4.1 Create repo to store your podspec file

Create a repro like `https://github.com/Xiaoye220/Specs` 

##### 1.4.2 Add your specs repo to CocoaPods repo

```ruby
# pod repo add [Spec repo name] [Spec repo source]
pod repo add [Specs] [https://github.com/Xiaoye220/Specs]
```

Then you open `~/.cocoapods/repos` you can see the `Specs` had been added to there

```ruby
open ~/.cocoapods/repos
```

##### 1.4.3 Push library to private specs repo

```ruby
# pod repo push [Spec repo name] --verbose
pod repo push Specs --verbose --use-libraries --allow-warnings
```

##### 1.4.4 Use private repo

Clarify the private spec repo source in `Profile` file

```ruby
source 'https://github.com/Xiaoye220/Specs.git';
source 'https://github.com/CocoaPods/Specs.git';

target 'Main' do
pod 'MyLib'
end
```

#### 1.5 Create lib project 

```ruby
# pod lib create [Project]
pod lib create MyProject
```



### 2. Swift Package Manager

#### 2.1 [Guidance of Swift Package](https://developer.apple.com/documentation/swift_packages)

#### 2.2 [Adding Package Dependencies to Your App](https://developer.apple.com/documentation/swift_packages/adding_package_dependencies_to_your_app)

#### 2.3 Create package

##### 2.3.1 Add Package.swift manifest file

```ruby
swift package init --type=executable
```

##### 2.3.2 Edit Package.swift manifest file

Here is the [guidance](https://developer.apple.com/documentation/xcode/creating_a_standalone_swift_package_with_xcode) about how to configue `Package.swift`

##### 2.3.3 Push Package.swift 

Push `Package.swift` to your remote repo so that others can add your package by adding dependency to your remote repo.



### 3. Carthage

#### 3.1 [Guidance of Carthage](https://github.com/Carthage/Carthage#quick-start)

#### 3.2 Install

```ruby
brew install carthage
```

#### 3.3 Add dependency via Carthage

##### 3.3.1 Create Cartfile file under the root folder

##### 3.3.2 Edit Cartfile with denpendency

[Detailed configuration](https://github.com/Carthage/Carthage/blob/master/Documentation/Artifacts.md)

Example

```ruby
# Require version 2.3.1 or later
# Clone repo from https://github.com/ReactiveCocoa/ReactiveCocoa
github "ReactiveCocoa/ReactiveCocoa" >= 2.3.1

# Require version 1.x
github "Mantle/Mantle" ~> 1.0    # (1.0 or later, but less than 2.0)

# Require exactly version 0.4.1
github "jspahrsummers/libextobjc" == 0.4.1

# Use the latest version
github "jspahrsummers/xcconfigs"

# Use the branch
github "jspahrsummers/xcconfigs" "branch"

# Use a project from GitHub Enterprise
github "https://enterprise.local/ghe/desktop/git-error-translations"

# Use a project from any arbitrary server, on the "development" branch
git "https://enterprise.local/desktop/git-error-translations2.git" "development"

# Use a local project
git "file:///directory/to/project" "branch"

# A binary only framework
binary "https://my.domain.com/release/MyFramework.json" ~> 2.3

# A binary only framework via file: url
binary "file:///some/local/path/MyFramework.json" ~> 2.3

# A binary only framework via local relative path from Current Working Directory to binary project specification
binary "relative/path/MyFramework.json" ~> 2.3

# A binary only framework via absolute path to binary project specification
binary "/absolute/path/MyFramework.json" ~> 2.3
```

##### 3.3.3 Update Carthage frameworks

```ruby
carthage update --use-xcframeworks --platform iOS
```

So you can find the frameworks under the folder `./Carthage/Build`

##### 3.3.4 Add framework to Xcode

On your application Targets -> General -> Frameworks, Libraries, and Embedded Content, drag and drop each **XCFramework** you want to use from the [Carthage/Build](https://github.com/Carthage/Carthage/blob/master/Documentation/Artifacts.md#carthagebuild) folder on disk

Then you can use the framework by `import XXXX` 

#### 3.4 Supporting Carthage for your framework

##### 3.4.1 Add new framework target

Xcode->File->Target->Framework, Add new framwork target 

##### 3.4.2 Mark new scheme about framework target as shared

[**To share an Xcode project’s scheme**](https://developer.apple.com/library/archive/documentation/IDEs/Conceptual/xcode_guide-continuous_integration/ConfigureBots.html#//apple_ref/doc/uid/TP40013292-CH9-SW3) so that Carthage can find it

##### 3.4.3 Verify if framework can be built correctly

```ruby
# Default scheme
carthage build --no-skip-current --use-xcframeworks
```



