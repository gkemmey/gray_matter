---
layout: post
title: "Dynamic Typing Flexibility: File#read"
published: true
---

Did you know, Ruby will take _literally_ anything when you're trying to open a file? It doesn't care about the type, it just cares if it can be treated as a string. Let's look, and assume `hello.txt` is a file with just the string "hello" in it:

```rb
File.read("hello.txt")
# => "hello"

File.read(Pathname.new("hello.txt"))
# => "hello"

class ATadContrivedSure
  def to_s
    "hello.txt"
  end
  alias to_str to_s
end

File.read(ATadContrivedSure.new)
# => "hello"
```

And if you give it something that can't be coerced into a string (doesn't define a `to_str` method), it'll blow chunks at runtime:

```rb
File.read(true)
# => TypeError (no implicit conversion of true into String)

# because we can, for now 👇
class TrueClass
  def to_str
    "hello.txt"
  end
end

File.read(true)
# => "hello"
```

Here's the types of designs I imagine Ruby + static typing will move us to (it obviously won't actually change `File#read`):

```rb
class File
  def self.typed_read(pathname)
    raise unless pathname.is_a?(Pathname) # faked type check
    read(pathname)
  end
end

# we're basically responsible for casting to Pathname
File.typed_read(Pathname.new("hello.txt"))
File.typed_read(Pathname.new(Nonsensical.new.to_s))
```

You know what this reminds me of? Java [^1].

```java
BufferedReader reader = new BufferedReader(new FileReader("./hello.txt"));
// 👆specifically this line 🤢

String         line = null;
StringBuilder  stringBuilder = new StringBuilder();

try {
  while ((line = reader.readLine()) != null) {
    stringBuilder.append(line);
    stringBuilder.append(System.getProperty("line.separator"););
  }

  return stringBuilder.toString();
}
finally {
  reader.close();
}
```

And I hear you asking, "Well, wouldn't you just let `typed_read` take an `Object`?". Yeah, and I'm sure that's how `File#read` will be type-annotated -- something like:

```
class File
  def read: (pathname: Object) -> String
end
```

And maybe we'll all write typed code that's equally as flexible as what we have today. But I'm skeptical. I worry types will enforce a certain rigidity in your design. After all, Java also has a top level `Object` class everything else inherits from, look what they came up with.

---

[^1]: Yes, this isn't the post-Java-7 way of doing things anymore, but it was when I was writing Java.
