---
title: "Learnings From Building a DB in Zig"
date: 2024-02-02T20:12:06+01:00
description: "I've built a database in zig and I've learnt a lot from it. This is a summary of what I've learnt."
---

# what I’ve learnt by writing an incredibly simple db in zig

For the last 3 days at Shopify we had a hackathon and in the spirit of learning something new I’ve decided to build a database in zig. Why? I’ve been wanting to learn zig for a while. How but like why the **** a db? It felt like a complex enough project that I wouldn’t get bored and I would need to learn enough of the language to do it.

## how did I start?

I can’t thank Phil Eaton and his blog post here: https://notes.eatonphil.com/zigrocks-sql.html enough. It was both a source of inspiration and also a good source material to base my db on. I’ve ended up almost copying his build setup, but everything else is pretty much original.

I’ve also taken quite a bit of inspiration from the first CockroachDB mapping schema (from sql to key value store semantics), which you can read about at: https://www.cockroachlabs.com/blog/sql-in-cockroachdb-mapping-table-data-to-key-value-storage/

Having read these 2 pieces of introductory information I decided that I just wanted to build something and that’s what I did. I started writing a parser for a simple CREATE query. Which is where I started getting myself introduced to zig. I’ve gone through https://zig.guide which taught me the basics and then I actually started writing code.

## first surprising moment

As I writing my parser I started having to handle or create errors, as either some allocation was failing or because the query my parser had to parse was bad and I wanted to return an error.

Zig built-in errors are different from what I expected. I’m used to the Result type from Rust, but zig errors are somehow similar but also different, in a surprising way. Zig errors are just a raw enum, they cannot have any associated data (like an error message or a cause or anything really). They can have descriptive names, but that’s it.

I also searched on the net and found that a possibility is to pass in an optional error reporter which can be populated with additional data when an error occur. Or instead of using the built-in errors you could recreate the rust result type and just use that.

It did seem a bit weird that there was no single way of doing errors (with data). I decided to just go with the built-in errors and not pass more information (such as where your query is wrong) because this was a hackathon and I also wanted to have the possibility to still use the try and catch syntax that zig has, which I think can only work with built-in errors.

## small tangent about parsers

As you may know by this point I had to write a parser for some simple subset of sql. The Phil Eaton blog post had some pretty thorough explanation about how he did it. He went for a pretty classic tokenizer + parser approach.

I didn’t want to do that, so I just rolled my own. I took inspiration from nom and parser combinators and built my own replica in zig (it’s not good I think, but it works, but it could potentially be made into something better and extracted out).

Anyway, this was cool and I much prefer this way of writing parsers. It feels simpler and you have more control over what’s happening and errors are somehow simpler to think about. Anyway, this is just a tangent!

## some pain points I didn’t expect

When I was about to write the “storage engine”, where to somehow store the data I decided to go the same way that Phil did and used rocksdb as my backend.

Now, rocksdb is a c++ project and if also offers a c api. I had heard that interfacing with c/cpp code from zig was somehow easy and I was expecting a way easier time with bundling some other native code and zig. That was not the case.

To be clear, one thing that the zig blog posts and documentation says about doing this is that you should build your c/cpp files using zig build and then it should be easy. I didn’t want to do that, because I didn’t want to read few thousand lines of Makefile code and understand how to build rocksdb to then replicate that in a build.zig file.

Anyway, I though I could just shell out to make from build.zig and just build it that way. Now you can do that, but there is no easy way to avoid shelling out to make if the static library is already compiled, which is annoying.

But the real pain point is that the makefile decides when it runs which compression library to use from your system libraries. These same libraries have to be added as a dynamic dependency to the final executable, but till make runs I have no way to know which ones it decides to use.

Where is the problem? build.zig tried to be a declarative dsl to declare how to build your project. So you first say I want to run this step (like shelling out) and then that step (like build my zig code) and then it actually happens. The trouble is that this means that it shells out to make after it has finalised describing how to build my zig code (which is where I would need to say that I need to link with some system library).

Unclear to fix this, so I just decided to run make in the rocksdb folder manually and just set which system libraries it uses on my machine. This is not portable to other machines, but it was a hackathon, I couldn’t spend 2 days just figuring out how to build my dependency.

This was the worst part of my zig journey, the build system has very little documentation and it also doesn’t allow a use case I would have thought it would cater for (using a separate build system easily to build some dependency).

I can see the point of it though, it’s a pretty cool way to build software, but it probably needs more polish and maybe what I was doing was just wrong.


## about memory management

It took a while to get used to manual memory management again, but it’s a considerable upgrade compared to C. defer and errdefer help a lot. It’s also really cool that allocators can just be swapped around and the language is really built around that. You want all your code to use an arena allocator? Easy peasy.

It’s also pretty cool that the testing allocator errors out when you do things that are bad, like for example leaking memory.

I did have one segfault on my third day, which caught me a bit by surprise. I still don’t fully understand why, changing a pointer to an ArrayList to just be an owned ArrayList fixed it. It would be neater if the testing allocator could save you from segfaults and print a nicer error.

It was also a bit weird that the standard library didn’t have support for utf8 strings, but maybe I’m just too rust-pilled on this topic.

## some good things about zig

I think, by this point, you may think I’ve kind of disliked my experience with zig, but actually I’ve enjoyed it. It’s a new-ish language and it isn’t stable yet, so I was warned about some parts needing polish. I did try to learn rust before 1.0 and I just gave up instead of writing something with it.

I really like the comptime stuff. I’m a huge meta programming nerd and comptime is just lovely. What is comptime you ask? It allows you to run zig code at compile time. This zig code can generate new types and do all sort of cool stuff. It can even allocate while it does its thing. The beauty of it is that it’s a small feature but it allows so many things, which I think is a great way to make a language powerful.

I mentioned that using c packages was a bit of a pain, but it also have to say that actually using the headers and coding with the c apis was a breeze (compared to figuring out how to link the 2 pieces together). zig can just import a header file and it generates zig code for everything inside of it. It’s incredible!

In general the language is pretty reasonable and it feels homogeneous, it uses the same patterns in most places. If you can read some zig code I think you can read most zig code, which is not always the case for languages that have a way larger set of features available.

## wait weren’t we talking about writing a db?

I guess I went into a pretty substantial tangent about zig, but I haven’t talked much about building a db on top of rocksdb. Now you could go read the 2 blog posts I linked and I think you would be good. But I’ll also try to condense what I’ve learnt/tried.

Note for the reader: what I have done may be (and probably is) bad. It was learning experience. I didn’t know much about zig nor database internals. But it was fun! You should try it!

Let’s start from where I started, the CREATE query. We have a parser for it. What do we do after that? I’ve decided that I would write to a rocksdb key called `/tables/<table_name>` some metadata about the table. This includes:

- its name (yes this is redundant)
- its columns, where each column has a name and a type (int or text for now)
- number of rows

The last one may not be necessary, but I wanted each row to have a unique id that is always increasing so having this in the table metadata helped with it. Could have I done something better? Maybe, but I didn’t have much time to be think about it so I went with this. But we're still missing a piece. We now to which key to write but what value do we write? rocksdb accepts plain bytes, so we need to serialize our table metadata. To do this, I decided, for no reason whatsover, to write my own simple binary format. strings are length prefixed, integers are just written as big endians, arrays are length prefixed and then followed by their elements, enums are just single byte integer values. With this it was pretty trivial to serialize my metadata and deserialize it.

Now we’re ready to support INSERT queries. We write a parser for them, using our nice parser combinators (a really cool thing is that once I wrote enough utilities for the CREATE query parsing function I had to do very little for all the others). Having parsed the query, we first fetch the table metadata. We also update how many rows the table has and write that to the db. Then we do some checks that we have the right number of columns, columns have the right names and the types are correct. If all of this is good, for each column we write its value to `/tables/<table_name>/<row_id>/<column_id>`. The `column_id` is just a number that is increasing for each column in the table. To this key we write the serialized value, which can be either a string or an integer. We serialize these by first writing a byte that indicates the type and then the value itself.

Now the db is really taking shape and we're ready to support SELECT queries, yuppy! As usual, we write a parser, no biggie. Having done that, we first fetch the table metadata, then we figure out which columns we want to select. Having done that, we iterate over all possible ids, as they are always increasing and we know from the metadata how many rows we have this is somehow trivial. For each row id we try to evaluate the where clause if there is one. I had to make where clause simple, so they can only be 1 binary comparison, like `age > 25`. You can't `AND` or `OR` them or use more complex syntax. I did not have the time to do this essentialy. Also no indexing is done, so this is pretty slow.

Anyway, how do we evaluate the where clause? Well, we evaluate the left hand side and the right hand side, which be either literals or column names. If they are column names we fetch the value from the db and deserialize it. Having done that we compare the 2 values and if the comparison is true we add the row id to the list of rows we want to return. Having done that we just fetch the values for the columns we want to return and we return them. Having evaluated the 2 operands we just use the right comparison function and we’re done. If the result is true then we fetch the wanted columns for the row and add them to a result set. If no where clause was given we just add all the rows to the result set. And that's it? We have a simple db!

## where's the code? plus some last word of wisdom (not really)

the code lives at: https://github.com/Maaarcocr/badb because as I said before, this is bad. But it was still fun and very educative to write it. I hope this inspires more people to try to do projects that may sound slightly out of their reach. I think it's important that as developers we try to make new cool things and sometimes this means doing hard things. I do not think I'm great at doing hard things, but at least I'm not afraid and I kind of enjoy the journey! I hope you do too!
