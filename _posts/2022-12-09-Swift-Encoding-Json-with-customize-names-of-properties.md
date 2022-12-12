---
layout: post
title: Swift - Encoding Json with customize names of properties
date: 2022-12-09
tags: [iOS, Swift, Notes]
excerpt_separator: <!--more-->
toc: true
---

Swift can simply encode json to object with JSONDecoder

<!--more-->

```swift
struct Family: Codable {
  var persons: [Person]
}

struct Person: Codable {
  var name: String
  var age: Int
}

let json = """
{
  "persons": [
    { "name": "Lilei", "age": 8 },
    { "name": "Hanmeimei", "age": 10 }
  ]
}
"""

let jsonData = json.data(using: .utf8)!
let jsonDecoder = JSONDecoder()
let persons = try! jsonDecoder.decode(Family.self, from: jsonData)
```



But consider the case that json contains invalid property like this

```json
{
  "persons": [
    { "person.name": "Lilei", "person.age": 8 },
    { "person.name": "Hanmeimei", "person.age": 10 }
  ]
}
```

We can't define `person.name` and `person.age` as properties because it againsts the swift property naming (contains `.`).

Now we need to use custom names of propertied for this case.

```swift
struct Person: Codable {
  var name: String
  var age: Int
  
  enum CodingKeys: String, CodingKey {
    case name = "person.name"
    case age = "person.age"
  }
}

let json = """
{
  "persons": [
    { "person.name": "Lilei", "person.age": 8 },
    { "person.name": "Hanmeimei", "person.age": 10 }
  ]
}
"""
```

