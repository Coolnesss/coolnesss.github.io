---
layout: post
title:  "Writing Ruby extensions with Rice"
date:   2016-12-13 00:53:41 +0200
categories: ruby C++
author: Chang
---

Writing Ruby is meant to be fast and intuitive, saving time for the programmer and getting things done fast. This often comes with a cost on actual software speed. A lot of times this is not something one needs to worry about, but sometimes you just feel that function you wrote would be so much faster when written in C or C++. Maybe if you're a seasoned C veteran, you might even feel like it would be easier to implement in the language you know so well.

Luckily Ruby allows us to rather seamlessly blend Ruby code with C/C++ code, to the point where one can mix functions written in both languages and call them from Ruby like they were actual Ruby code. There's a lot of tutorials on the web for writing C extensions in Ruby, but not very many for actual [C++](https://www.scaler.com/topics/cpp/), and even fewer with a "no bullshit"-approach. I'll reference a couple of good ones, but even when following these tutorials i felt like the web was missing a straightforward tutorial on writing specifically C++. With that said, [here](https://www.amberbit.com/blog/2014/6/12/calling-c-cpp-from-ruby/) is a great tutorial that lists different approaches but doesn't really go into detail in any of them. Another good guide is [this Rubygems guide](http://guides.rubygems.org/gems-with-extensions/) which focuses on releasing a gem with a C extension.

This guide will focus on [Rice](https://github.com/jasonroelofs/rice) with an emphasis on C++. We're gonna be making a object in Ruby that will have methods which are written in Ruby and others that are written in C++. I'll assume you have Ruby installed, and you've installed the Rice gem and a C++ compiler of your choice.

Let's start by writing some C++ code. Our idea is to count the occurrences of a substring in a string. We'll start by creating a C++ class that wraps this functionality.

```c++
// File frequency.cpp
#include <string.h>
#include "rice/Class.hpp"
#include "rice/Constructor.hpp"

using namespace Rice;
using namespace std;

class Frequency {
    private:
        string s;

    public:
        Frequency(string s) :s(s) {};

        int count_word(const string& w) {
          int count = 0;
          int pos = s.find(w, 0);
          while (pos != string::npos) {
              count++;
              pos = s.find(w, pos + 1);
          }
          return count;
        }

};

extern "C"

void Init_frequency() {
    Data_Type<Frequency> rb_cfrequency =
    define_class<Frequency>("Frequency")
      .define_constructor(Constructor<Frequency, string>())
      .define_method("count_word", &Frequency::count_word);
}
```

There's a lot going on here so let's break this down. We're including the Rice header libraries and creating a class called Frequency that takes a string in the constructor and allows us to check if that string contains a certain substring. What's interesting is at the bottom, where we define the Ruby class that our C++ gets wrapped into.

We use the `Data_Type` class to create an object and define the methods that our Ruby class should have. Here we also define the constructor that our class has. Note that Rice automatically performs type conversions for us, so we don't have to worry about how an `std::string` gets converted into a Ruby string. If you want to use something more complicated, like an `std::vector`, you'll have to do some more work though.

Now all our C++ coding is done. This is pretty good because with Rice you never have to alter existing code, simply add the code that defines the Ruby interface and you're all set.

Next we'll have to create an `extconf.rb` file that will create a Makefile for our C++ code. Mine looks like this:

```ruby
require 'mkmf-rice'

$CXXFLAGS += " -std=c++11 "
$CXXFLAGS += " -Ofast "
create_makefile('frequency/frequency')
```

Here we're also adding some compiler flags, clearly we want our C++ code to be as fast as possible so we use `-Ofast`, and we'd like to use C++11 features so we set that flag as well.

Now simply running this file should yield

```bash
$ ruby extconf.rb
creating Makefile
```

Great, our Makefile is done. If you open it you'll notice it has a lot of stuff in it, but we'll just call `make` to compile our code.

```bash
$ make
compiling frequency.cpp
(...)
linking shared-object frequency/frequency.so
```

Alright, and we're pretty much done already! Amazingly, we can now require the `.so` file that was just built straight from Ruby, and use the code. Let's try it.

```ruby
[1] pry(main)> require './frequency'
=> true
[2] pry(main)> f = Frequency.new "really cool stuff this is"
=> #<Frequency:0x000000012f28c0>
[3] pry(main)> f.count_word "stuff"
=> 1
```



So this is pretty neat. With a few short steps we were able to wrap our C++ class into a full-fledged Ruby class. But let's go even further. Wouldn't it be cool if our `word_count` method was available on any string in Ruby? After all, why make a separate class for something that should clearly be a job of the `String` class.

Here's where the power of Ruby really shines. We can simply monkey-patch the `String` class to include a method that uses our `Frequency` implementation. While you probably [actually should not do it this way](http://www.justinweiss.com/articles/3-ways-to-monkey-patch-without-making-a-mess/), we'll just write some simple code for illustration.

```ruby
# frequency.rb
require './frequency.so'

class String
  def count_word(word)
    f = Frequency.new self
    f.count_word word
  end
end
```

Note that we have to be explicit about requiring the actual `.so` object this time around, since we have two files in our folder that are both called `frequency`. Let's test our code out.

```ruby
[1] pry(main)> require './frequency.rb'
=> true
[2] pry(main)> "cool stuff guys".count_word "cool"
=> 1
```

Wow, how cool is that? We're calling C++ from a core Ruby string literal.

But we can actually go even further. Maybe we decide our `Frequency` class requires some more functionality, but it makes more sense to write it in Ruby this time. Let's go ahead and do just that:

```ruby
require './frequency.so'

class Frequency
  def contains?(word)
     count_word(word) >= 1
  end
end
```

And

```ruby
[2] pry(main)> f = Frequency.new "great examples, really"
=> #<Frequency:0x000000022265f8>
[3] pry(main)> f.contains? "great"
=> true
```

To summarize, it's really easy to write C++ extensions into Ruby, and they can blend in really well with your existing code. Next time a Ruby implementation isn't fast enough for you, just write your C++ implementation and wrap it over to Ruby.
