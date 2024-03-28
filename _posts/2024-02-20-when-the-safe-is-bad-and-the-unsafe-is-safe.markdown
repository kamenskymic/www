---
layout: post
title:  "When the \"safe\" is bad and the \"unsafe\" is safe."
subtitle: "Another day in product security"
date:   2024-02-20 09:19:21 +0300
categories: blog
hero_height: is-small
author: josh
summary: |
    Product Security is hard. There are a huge number of different things you think about at the same time, while still being able to identify the most serious and urgent issues.
image: /assets/img/2024-02-20-unsafe/dangerous.png
series: prisma_series
---

### Update 2024-03-28:

_It turns out that there are also situations where `$queryRaw` and `$executeRaw` are actually not safe 🤦‍♂️. You can read the full story in the [second post in this series](/blog/2024/03/28/when-the-safe-is-worse-than-you-thought.html)._

_I think the original point of this post still stands but I have added some clarifications marked with "**Update:**" below._

## Introduction

Product Security is hard. There are a huge number of different things you think about at the same time, while still being able to identify the most serious and urgent issues.

![image](/assets/img/2024-02-20-unsafe/dangerous.png){: .blog-image}

One of the biggest problems is that every application is different, with lots of different programming languages, and within each language are all sorts of different Frameworks and libraries which they might use. Each of these might have their own security quirks and issues.

In order to provide effective product security advice, you need to feel comfortable figuring out the correct guidance for the current situation.

## Framework specific guidance

I saw an example of this recently where a particular framework in a particular language was using syntax that I found super unintuitive, and to be honest a little misleading.

Having discussed this internally at Bounce, I set out to try and prove this through a [LinkedIn poll](https://www.linkedin.com/feed/update/urn:li:activity:7162720032936988673/).

### The LinkedIn poll

![image](/assets/img/2024-02-20-unsafe/LIpoll.png){: .blog-image}

Here is the formatted code as well in case it is easier to review.

``` ts
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

const users = await prisma.$queryRaw`
SELECT * FROM "User" WHERE email = ${untrustedInput};
`
```

The question is, is this code vulnerable to SQL injection 💉? 

### What do you think?

If you didn't see the poll, maybe take a minute to take a look and decide what your answer would be.

.

.

.

Just another minute to think about it

.

.

.

Ok? Ready? Let's move on.

## Responses to the poll

### Public responses

So here we can see the final responses to the poll.

![image](/assets/img/2024-02-20-unsafe/pollresults.png){: .blog-image}

Now as you can see, a significant majority of people who responded with an actual answer believe that this code is insecure.

### My response

To be honest, if I had been presented with this question and didn't have time to look it up, I probably would have also said that it's insecure. The reason for this is that as a product security person, I'm constantly expected to have familiarity with a variety of different languages without spending enough time to be an expert in any of them. That means that I'm always looking for the ways in which languages are similar and have similar syntax (and why Ruby is so challenging because it has completely different ideas about syntax compared to most other languages!). 

In this case, if I look at the code, not knowing the specific library and not being a JavaScript/Typescript expert, I see what looks like a clear case of string concatenation into an SQL query. It looks like standard string interpolation, like in the example below.

``` js
const org_name: string = "GeeksforGeeks";
const org_desc: string = "A Computer Science portal for all geeks."
console.log(`Organization name ${org_name}, 
Organization Description: ${org_desc}`);
```
[Source](https://www.geeksforgeeks.org/how-to-perform-string-interpolation-in-typescript/)

### The correct response

So in fact, the answer to this poll is that the code is secure!

Why? Well, [Tomer Zait](https://www.linkedin.com/in/realgam3/) (CTF player and builder extraordinaire and therefore an expert in edge cases) provided an explanation in the comments which is great because I myself found it quite hard to explain.

![image](/assets/img/2024-02-20-unsafe/tomersresponse.png){: .blog-image-tall .blog-image }

Here is Tomer's code so that you can try it yourself in the console:

``` js
function prepare(strings, ...values){
  console.log({query: strings.join("?"), values: values});
}
prepare`SELECT * FROM "users" WHERE username = ${'admin'}`
```

(Note that the Node.js security maven [Liran Tal](https://www.nodejs-security.com/) also provided a nice answer which I removed because I didn't want to spoil things 🤣)

The basic idea is that in JavaScript, if you provide a backticked string (which includes interpolated variables) immediately after the function (without even using brackets surrounding the parameters like in most function calls), the function actually receives a string with parameter markers plus a bunch of parameters. The functionality is called [tagged templates](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals#tagged_templates).

This is then easy to translate into a safe, parameterized SQL query.

The Prisma documentation [helpfully explains this as well](https://www.prisma.io/docs/orm/prisma-client/queries/raw-database-access/raw-queries#queryraw).

## So we're safe?

~~This is the only way of using that function, so that means that this function is secure, right? Well yes so it would seem.~~

_**Update:** As long as we are using the query like this, with the query text included as we call the function then yes this is safe. However, there are situations where you can make it unsafe as you can see in the [second post in this series](/blog/2024/03/28/when-the-safe-is-worse-than-you-thought.html)._

On the other hand, it still bothers me that the query *looks* unsafe. Also, because it hides why it is safe behind a specific JavaScript feature, I do worry that it is not clear *why* it is safe, and therefore it might not be clear why this sort of syntax would not be safe in other contexts.

Nevertheless, if you are looking for a simple way of writing dynamic but safe queries using Prisma, this is a great option (_**Update:** if you follow the guidance for using this securely in the [second post in this series](/blog/2024/03/28/when-the-safe-is-worse-than-you-thought.html)_). 

## Making the recommendation

So as product security people, we can now tell developers to only ever use that `$queryRaw` safe function, and carry on with our day right? Right...?

![image](/assets/img/2024-02-20-unsafe/right.jpg){: .blog-image }

Well actually no.

### Don't use raw queries

First of all, it's entirely possible that up to this point you've been screaming at the screen "why not build native, safe JSON queries instead of writing raw SQL because that's the whole point of using this library!"

So yes that's a first key takeaway here, that writing raw queries like this should never be the preferred option, assuming the use case allows it. Again, the Prisma documentation provides [lots of guidance on how to do this](https://www.prisma.io/docs/orm/prisma-client/queries).

### Right here, right now

But there's another problem. This function will only ever take a raw string. Nothing else. If you try and pass it a string variable it will parameterize it and your query will break. ~~In short, this function is only usable if your use case allows you to write your raw query in text, right there, as you use that function.~~  

If your use case requires you to build that query string, or even start building that query string somewhere else, ~~you cannot use this function~~ _**Update:** using this function gets a lot more complex and potentially dangerous_.

~~I repeat, you **cannot** use this function.~~

If you insist that a developer uses this function in that scenario, they will either dismiss you as not knowing what you're talking about, or they will waste time trying and failing to make this work, and *then* decide you don't know what you're talking about and be annoyed for having wasted time, (_**Update:** or they will come up with a crazy work around that ends up being insecure!_)

So let's say that in the developer's use case, they need to build query strings dynamically. You can tell them to use variables and parameter markers (like `?` or `$1`, `$2` depending on the database) instead of string concatenation, but they still won't be able to use this function, (_**Update:** in a straightforward and secure way._)

### So what are we going to recommend?

Luckily, there is another function in Prisma which they can use. You can pass the function a string variable containing the query with parameter markers, and then variables for each of the parameters and it will safely run the parameterized query.

Unluckily, this function is called `$queryRawUnsafe` which is guaranteed to immediately trigger any security person who gets a sniff of this (including me when I first saw this.) But it is called this for a good reason, it is very easy to use this function in an unsafe way that results in SQL injection.

Whilst the [warning in the documentation on this function](https://www.prisma.io/docs/orm/prisma-client/queries/raw-database-access/raw-queries#queryrawunsafe) is nuanced, it still pushes you to use `$queryRaw`.

![image](/assets/img/2024-02-20-unsafe/unsafewarning.png){: .blog-image }

And don't get me started on how SAST (code security) scanners will react.

So, for example. Look at the following code:

``` ts
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

const users = await prisma.$queryRawUnsafe(`
SELECT * FROM "User" WHERE email = ${untrustedInput};
`)
```

It looks almost identical to the original [code above](#the-linkedin-poll), right?

Except that **this code is 100% vulnerable and will lead to SQL injection**. If you want to use this function, like I said above you have to use parameter markers and then pass the parameters as one or more variables:

``` ts
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

const users = await prisma.$queryRawUnsafe(`
SELECT * FROM "User" WHERE email = $1;
`, untrustedInput)
```

This code is safe and not vulnerable. I also like the fact that the parameterization is a little more explicit.

The bottom line is that this function can be used safely but can also be used unsafely.

### Finally we have our recommendation

So having read all of this (_**Update:** and if you read the guidance in the [second post in this series](/blog/2024/03/28/when-the-safe-is-worse-than-you-thought.html)_), you are now in a position to make recommendations to developers in a particular usage scenario. As a software security person, you either need to be prepared to do this research or be able to coach and convince your developers to be able to do it for themselves. (_**Update:** You also need to be ready to discover that documentation is incomplete and that you have to come up with your own solution._) Obviously this is just one of the many questions which is likely to come up over the course of your day. Did I mention that software security is hard? 

## In conclusion

So hopefully this has been a useful thought exercise about the day to day considerations in software security.

A few key conclusions I think.

- Most developers have had a bad experience with security people, being able to bridge this divide means being able to speak their language and provide them with realistic solutions.
- **Update:** Hopefully this post and the [next post in this series](/blog/2024/03/28/when-the-safe-is-worse-than-you-thought.html) demonstrate why using the ORM's built-in query mechanism is a lot easier!
- Wherever possible, I'd like to provide a simple and ideal solution, like `$queryRaw` _**Update:** with queries being written there and then in the function call_.
- However, the ideal solution is not always possible and product security people will need to be ready to find alternatives, secure the alternatives, and to choose their battles.
- This also means that automated scanning which doesn't understand this context will probably provide the wrong answer (but more on this in a future post).

