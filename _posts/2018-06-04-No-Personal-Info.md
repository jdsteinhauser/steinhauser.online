---
title: Don't Put Personal Information in Code!
published: true
description: How I delivered a birthday card because of a code comment
tags: #security
cover_image: https://i.imgflip.com/20twio.jpg
---
_Originally posted on [The Practical Developer](https://dev.to/jdsteinhauser/dont-put-personal-information-in-code-3566)_

Normally, I wouldn't cringe during a code review when I see

```(C#)
var testValue = 716.1976;
```

*BUT* when I include the rest of the line...

```(C#)
var testValue = 716.1976;      // Carl's birthday
```

Well, then I start to question what in the world you were thinking! _(Note: some names and dates have been changed to protect the innocent)_

Most online tutorials involving DevOps or API keys will warn you to not include keys and passwords in your repo; those are the sort of things that are best relegated to environment variables (or some other secure method). But usually you don't come across tutorials warning to exclude Personally Identifiable Information (PII).

# What is PII?

In short, PII is anything that you can use to figure out who somebody is. You usually hear it in identity theft discussion. Several things are included as PII, such as:

- Birth date
- Home city
- Social Security Number (in the U.S.)
- High school
- Location of birth

You know all those security questions for password recovery? Yeah, all of that is PII.

# What can you do with that?

Well, there are several websites that, given a few tidbits of your personal info, can be used to find out all sorts of things. Current and past addresses. Names of relatives. Former/maiden names. All with a few simple keystrokes on [Pipl](https://www.pipl.com/) or [Intelius](https://www.intelius.com/) and you can find a lot of information.

Going back to the comment that started all of this, I didn't have a surname for "Carl," but I did have the developer's last name thanks to a comment block at the top of the file (not to mention, a commit history). A quick search got me his address, Facebook profile, occupation (a nurse!), and an age that corroborated his birthday in the code.

# Did you send "Carl" a birthday card?

I totally did... in my mind. But then, I played through how freaky that would be for Carl, and how mad he would be at "Lucy" despite no ill intentions. I didn't want to fracture any relationships solely to make a point... But "Lucy" is now well aware of the ramifications.