---
title: "Code Snippets & Syntax Highlighting"
date: 2026-03-03
draft: false
tags: ["code", "guide"]
translationKey: "code-snippets"
summary: "How to use fenced code blocks with syntax highlighting for various programming languages."
---

Hugo uses [Chroma](https://github.com/alecthomas/chroma) for syntax highlighting, which supports over 200 languages and markup formats. Here's how to use code blocks in your posts.

## Basic Syntax

Wrap your code in triple backticks with the language identifier:

````markdown
```go
fmt.Println("Hello, world!")
```
````

## Language Examples

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
  return new Date(post.date).toLocaleDateString('en-US', {
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

## Inline Code

For inline mentions, use single backticks: `hugo server --port 1313`.

## Supported Languages

Hugo's Chroma highlighter supports 200+ languages including: C, C++, C#, Dart, Elixir, Erlang, F#, Haskell, Julia, Lua, MATLAB, Objective-C, Perl, PHP, R, Ruby, Scala, Swift, Zig, and many more.

For the full list, see the [Chroma documentation](https://github.com/alecthomas/chroma#supported-languages).
