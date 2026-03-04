---
title: "코드 스니펫과 구문 강조"
date: 2026-03-03
draft: false
tags: ["code", "guide"]
translationKey: "code-snippets"
summary: "다양한 프로그래밍 언어의 코드 블록과 구문 강조 사용법을 소개합니다."
---

Hugo는 구문 강조에 [Chroma](https://github.com/alecthomas/chroma)를 사용하며, 200개 이상의 언어와 마크업 형식을 지원합니다. 포스트에서 코드 블록을 사용하는 방법을 알아보겠습니다.

## 기본 문법

코드를 트리플 백틱으로 감싸고 언어 식별자를 지정합니다:

````markdown
```go
fmt.Println("Hello, world!")
```
````

## 언어별 예시

### Go

```go
package main

import "fmt"

func main() {
	names := []string{"Alice", "Bob", "Charlie"}
	for i, name := range names {
		fmt.Printf("%d: %s\n", i, name)
	}
}
```

### Python

```python
from dataclasses import dataclass

@dataclass
class Post:
    title: str
    date: str
    tags: list[str]

    def summary(self) -> str:
        return f"{self.title} ({self.date})"

posts = [Post("Hello", "2026-03-01", ["blog"])]
for post in posts:
    print(post.summary())
```

### JavaScript / TypeScript

```javascript
const fetchPosts = async () => {
  const response = await fetch('/api/posts');
  const posts = await response.json();
  return posts.filter(post => !post.draft);
};
```

```typescript
interface Post {
  title: string;
  date: string;
  draft: boolean;
  tags: string[];
}

function formatDate(post: Post): string {
  return new Date(post.date).toLocaleDateString('ko-KR', {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
  });
}
```

### Rust

```rust
use std::collections::HashMap;

fn word_count(text: &str) -> HashMap<&str, usize> {
    let mut counts = HashMap::new();
    for word in text.split_whitespace() {
        *counts.entry(word).or_insert(0) += 1;
    }
    counts
}

fn main() {
    let counts = word_count("hello world hello rust");
    println!("{:?}", counts);
}
```

### Java

```java
import java.util.List;
import java.util.stream.Collectors;

public class Main {
    record Post(String title, boolean draft) {}

    public static void main(String[] args) {
        var posts = List.of(
            new Post("Hello World", false),
            new Post("Draft Post", true)
        );

        var published = posts.stream()
            .filter(p -> !p.draft())
            .map(Post::title)
            .collect(Collectors.toList());

        published.forEach(System.out::println);
    }
}
```

### SQL

```sql
SELECT p.title, p.date, GROUP_CONCAT(t.name) AS tags
FROM posts p
LEFT JOIN post_tags pt ON p.id = pt.post_id
LEFT JOIN tags t ON pt.tag_id = t.id
WHERE p.draft = false
GROUP BY p.id
ORDER BY p.date DESC
LIMIT 10;
```

### Shell / Bash

```bash
#!/bin/bash
echo "Building site..."
hugo --minify

if [ $? -eq 0 ]; then
  echo "Build successful!"
  hugo server --bind 0.0.0.0 --port 1313
else
  echo "Build failed." >&2
  exit 1
fi
```

### HTML / CSS

```html
<article class="post">
  <h1>{{ .Title }}</h1>
  <time datetime="{{ .Date.Format "2006-01-02" }}">
    {{ .Date | time.Format ":date_long" }}
  </time>
  <div class="content">
    {{ .Content }}
  </div>
</article>
```

```css
.post {
  max-width: 720px;
  margin: 0 auto;
  line-height: 1.8;
}

.post h1 {
  font-size: 2rem;
  margin-bottom: 0.5rem;
}

@media (prefers-color-scheme: dark) {
  .post {
    color: #e0e0e0;
  }
}
```

### YAML / TOML / JSON

```yaml
# hugo.yaml
baseURL: https://example.com/
title: "My Blog"
theme: gandanham
params:
  author: Hyeongjun
```

```toml
[params]
author = "Hyeongjun"
description = "A minimal blog"
```

```json
{
  "name": "my-blog",
  "version": "1.0.0",
  "scripts": {
    "dev": "hugo server",
    "build": "hugo --minify"
  }
}
```

### Kotlin

```kotlin
data class Post(val title: String, val tags: List<String>)

fun main() {
    val posts = listOf(
        Post("Hello World", listOf("blog")),
        Post("Code Guide", listOf("code", "guide"))
    )
    posts.filter { "blog" in it.tags }
         .forEach { println(it.title) }
}
```

### Docker / Dockerfile

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY . .
RUN go build -o server .

FROM alpine:3.19
COPY --from=builder /app/server /server
EXPOSE 8080
CMD ["/server"]
```

## 인라인 코드

인라인으로 코드를 언급할 때는 단일 백틱을 사용합니다: `hugo server --port 1313`.

## 지원 언어

Hugo의 Chroma 하이라이터는 200개 이상의 언어를 지원합니다: C, C++, C#, Dart, Elixir, Erlang, F#, Haskell, Julia, Lua, MATLAB, Objective-C, Perl, PHP, R, Ruby, Scala, Swift, Zig 등.

전체 목록은 [Chroma 문서](https://github.com/alecthomas/chroma#supported-languages)에서 확인하세요.
