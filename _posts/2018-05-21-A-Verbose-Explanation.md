---
title: A verbose explanation of compact code
published: true
description: My attempt explaining code I am proud of to my wife
tags: #showdev #clojure #productivity
cover_image: https://thepracticaldev.s3.amazonaws.com/i/iawy8ahxe44s8o326p2j.jpg
---
_Originally posted on [The Practical Developer](https://dev.to/jdsteinhauser/a-verbose-explanation-of-compact-code-546g)_

My wife is a genius. She's straight up brilliant. She's an astrophysicist with a passion for process and product improvement. However, for the life of me, I have not found an effective way to talk to her about details of code that I've written. Granted, most of the time we have to discuss work happenings is at the dinner with 2 or 3 kids (depending on extracurriculars) being loud, so I have to be succinct. I can answer "how many SLOC" and "how can you reuse this elsewhere" and "what does it do" and "why did you pick that language" fairly easily. But when she wants more technical detail... well, I haven't found a way to be succinct and still convey understanding. So this post is dedicated to the conversation I was trying to have with my wife a few weeks ago about a task I finished at work that day.

# Scope of the work

I have a customer that absolutely loves to automate as much as possible so that his people (re: me and the other folks in the lab) can maximize our time spent in code. He's been a really great customer so far, and I volunteered to write something: an automated task to pull together status on all the findings that we found and/or worked on this week, generate a PDF, and ship it off to his boss. I mean, how hard could it be?

# Tech Choices

My customer has a love of keeping things as simple as possible, and wanted to build our reports in Markdown, with our findings reported each week in JIRA placed in a table. This was going to require flattening out a response from the [JIRA REST API](https://developer.atlassian.com/server/jira/platform/rest-apis/) and printing out the applicable data, with a "TBD" where there were nulls. Oh, and he wanted to be able to reuse this for multiple reports, where the different table columns could change.

At this point, I'd had a few things chosen for me:
- JIRA input
- Markdown output

Now, all I had to do was choose a language. I finally saw a place to use my pet project language (Clojure) in production! It wasn't just a whimsical choice; I evaluated a few different languages before finally settling in on Clojure.

# Breaking apart the JIRA response

This project didn't really take very long; most of my time was spent in Chrome Dev Tools figuring the IDs for the custom fields so that I could flatten the data structure into a simple HashMap. Since all I had to deal with were the `issues` portion of the JIRA response, I chose to treat each issue as its own "object". This provided a good mapping from response to table because each JIRA issue was going to be its own row in the table. I created a function similar to the following:

```(Clojure)
(defn issues->useable
  [issue]
  { :finding-id (str "[" (:key issue) "](" (:self issue) ")")
    :title (get-in issue [:fields :summary])
    :status (get-in issue [:fields :status :name])
    :priority (get-in issue [:fields :priority :name])
    :date-created (get-in issue [:fields :created])
    :components (->> issue :fields :components (map :name) (str/join ","))
    :due-date (get-in issue [:fields :duedate])
    :labels (->> issue :fields :labels (str/join ","))
    :risk-consequence (get-in issue [:fields :customfield_22007 :value])
    :risk-probability (get-in issue [:fields :customfield_22006 :value])
  })
```

Yes, I was a little cheeky in my function naming. And yes, I know there's a lot of code to digest here. I'll step through it.

## Output

The output of this function isn't readily apparent to those who haven't looked at Clojure before. Clojure typically represents data structures as HashMaps. The tokens with a leading ":" indicate that they are a `keyword`, mostly like a sigil in Elixir or a symbol in Ruby. I'll elaborate on why I say "mostly" later, as it was critical to choosing Clojure. In Clojure, a map is surrounded in curly braces with keys and values grouped together. For example,

```(Clojure)
{:a 1, "foo" "bar"}

```

is a map with the key/value pairs `:a => 1` and `"foo" => "bar"`. Yes, Clojure can have non-keywords as keys, but it's not something that is commonly done in practice. The comma in between the pairs is actually optional; Clojure treats all commas as whitespace. So the output of my `issues->useable` function is a map with keyword/string pairs.

The `get-in` function is also rather nice. Given a nested hashmap and a sequence of keys, it will parse its way down through the nested hashmap and return the value stored there. If there is no value associated with that ending key, `nil` is returned.

A few other Clojure concepts come to life in this section of code as well. If we look at how the `components` field is generated in my output map, you'll see a the `->>` macro. `->>` is an end-threading macro that applies the output of the previous function evaluation as the last argument of next function evaluation. So for

```(Clojure)
    :components (->> issue :fields :components (map :name) (str/join ","))
```

First, the `issue` is passed to the *function* `:fields`.

### I thought :fields was a keyword!

Here is the big reason I chose Clojure. Keywords in Clojure also implement the `IFn` interface, meaning that they are considered to be functions in Clojure. So the expression `(:a {:a 5 :b 6})` will evaluate to `5`. It is returning the value associated with the `:a` keyword.

Sure, I could've written that part of the function as `(:fields issue)`, but then when I wanted to get further in depth, I'd have to write the whole function as

```(Clojure)
(str/join "," (map :name (:components (:fields issue))))

```

And that's nowhere near as clean as with using the `->>` macro. By using the front-threading and rear-threading macros in Clojure, you are able to see the data pipeline much more cleanly, similar to how you would use the `|>` operator (and its variants) in OCaml, F#, or Elixir.

### Creating table rows

So now I could transform an issue into a flattened map, but I need to have it as a markdown row. Markdown separates its table columns with pipes (`|`), with pipes on the outsides to box in the whole row. I wrote the following function to do this:

```(Clojure)
(defn create-row
  [values]
  (->> values
       (map (fnil name "TBD"))
       (into '(nil))
       (cons nil)
       reverse
       (interpose "|")
       (apply str)))
```

So again, I start with the rear-threading macro and *just the values* from my key/value pairs. Then I map `(fnil name "TBD")` over each of the values. `fnil` returns a higher-order function that takes `nil` parameters passed to it, replaces them with a different specified value (in our case "TBD"), and calls the function listed as the first argument. I used `name` to return a string-ified version of the value in the map (or "TBD" in the case of a nil). Sure, I could've used the `identity` function instead of `name`, but that's 4 whole characters more ;-) I love `fnil` solely because it reduces the footprint of NULL/nil checking logic.
The next line may look a little bizarre to you then, based on my previous line:
`(into '(nil))` puts the values that have just be string-/"TBD"-ified and puts them into a list that contains only the value `nil`. I'll explain that in a sec, along with the `(cons nil)` line. The call to `reverse` ought to be fairly clear: reverse the order of items in a sequence.

The reason for using `nil`s above was for use in the `interpose` function. `interpose` takes a sequence of items and inserts the specified value in between them. For instance, `(interpose 1 [3 4 5])` will yield the sequence `(3 1 4 1 5)`. However, I need pipes at the beginning and ending of my sequence as well, for the "outer walls" of the table. So I use `(into '(nil))` to pass the values from my sequence into a list that has only a `nil` in it. Since `into` performs a `conj` operation on each element, the elements in the sequence that we've just created will be added in reverse order. For example `(into '(nil) [1 5])` will yield `(5 1 nil)`. If a `cons` a `nil` onto the front of that collection, then I'll have a sequence of `(nil 5 1 nil)`. Interposing pipes into that will then give me `(nil "|" 5 "|" 1 "|" nil)`.

I then `apply` the `str` function to the collection that we have just created. The `str` function creates a string by concatenating the args passed to it together, with `nil` yielding an empty string. `apply`ing that to our now reversed collection `(nil "|" 1 "|" 5 "|" nil)` will yield the string "|1|5|". Perfect! This is exactly what we need to create a table row in Markdown.

### Creating the whole table

I now have a function to create my rows, but rows aren't terribly useful in a table that has no labels. So we need to have a function that creates the whole table, complete with which "column" of data is which. I was able to generate that in 6 very dense lines of Clojure:

```(Clojure)
(defn create-table
  [ordered-keys issues]
  (let [header        (create-row (map ->PascalCase ordered-keys))
        separator-row (create-row (repeat (count ordered-keys) "---"))
        rows (map (comp create-row (apply juxt ordered-keys)) issues)]
    (apply vector header separator-row rows)))
```

Let's walk through this. The function `create-table` takes a list of ordered keys, as well as the flattened issues (or, basically, any flat hashmap) and generates a Markdown table. I'm also assigning values to three local variables: `header`, `separator-row`, and `rows`. Let's look at the expression where we use those variables: `(apply vector header separator-row rows)`. The `vector` function takes a list of arguments and creates a vector (i.e., `[1 2 3]` and not a list like `(1 2 3)`) from those arguments. In this case, I'm creating a vector of table rows, starting with the header, and a Markdown-required separator row between the headings and values. The `header` is merely a row created from converting the ordered-keys from keywords to PascalCase using the indispensable [camel-snake-kebab](https://github.com/qerub/camel-snake-kebab) library. This way, `:finding-id` will be printed as the more management-palatable `FindingId`. Secondly, the separator-row is just three dashes repeated as many times as we have columns. Now, onto my favorite line of Clojure in this project, just due to the dense power of it:

```(Clojure)
rows (map (comp create-row (apply juxt ordered-keys)) issues)
```

Clearly, I'm `map`ping a function across all the rows passed in. That function is fairly complex. `juxt` is one of my absolute favorite functions in all of Clojure. It takes a sequence of functions and returns a function that passes the same argument to all functions in that sequence and returns a vector of their results. For instance, `((juxt + *) 3 4)` yields `[7 12]`. When I was doing primarily .NET development, I ported this function and multiple arities of it so that I could calculate multiple, independent metrics on terabytes of data in one pass. Using `apply juxt` to our list of ordered keys (which are all keywords), I just created a function that, when evaluating our data, will generate a vector of the values associated with those keywords in my data, in the order specified. For instance, `((juxt :a :id) {:a 5 :b 60 :id "blue"})` will yield `[5 "blue"]`. I then compose that (the `comp` function) with the create-row function, because

```(Clojure)
(map create-row
  (map (apply juxt ordered-keys)) issues)
```

just didn't look as clean.

# Conclusion

So that's basically it! It's a lot of dense code that is hard to understand at one pass, but is extremely flexible once you understand its use. I can now create multiple tables from any set of data, not tied to any particular type of data. I can pass my JIRA issues (in this case) into my `create-table` function with different sets of ordered keys to create different reports - for example, one table useful for middle management to report to upper management, and one for developers realizing the visibility of the issues they are working. I'm also not tied to JSON or any other format of data. So long as I can get the data into a map, I can reuse this to generate Markdown tables.

I know this was a rather long read, so congrats if you made it through! If you've got some particularly tricky yet useful code, I'd love to see you write about it as well. And if you have yet to explore Clojure, I welcome you to give it a try and see these and some other exciting language features :-)