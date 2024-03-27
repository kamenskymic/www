---
layout: post
title:  "When the \"safe\" is worse than you thought"
subtitle: "Yet another day in Product Security, this one more painful"
date:   2024-03-02 09:19:21 +0300
categories: blog
hero_height: is-small
author: josh
summary: |
    Product Security continues to be hard. Sometimes even when you think you have the solution, reality bites back. In this post I will take you through how I had to eat humble pie after my previous blog post.
image: /assets/img/2024-04-02-prisma-2/pie.jpg
series: prisma_series
---

## Introduction

[My previous post](/blog/2024/02/20/when-the-safe-is-bad-and-the-unsafe-is-safe.html) was about better understanding the situation when you make software security recommendations and how that can often be quite tricky. I illustrated this with the way that the Prisma ORM handles SQL injection.

![Picture of pie representing me eating humble pie](/assets/img/2024-04-02-prisma-2/pie.jpg){: .blog-image}

As it turns out, I proved myself correct almost immediately by discovering that I had overestimated the strict accuracy of the documentation and underestimated the ability of developers to get the job done in whatever way possible.

In this post, I'll explain what happened and the revised guidance around Prisma ORM.

## Suggesting the "safe" method

![image](/assets/img/2024-04-02-prisma-2/inconceivable.png){: .blog-image}

### Original documentation

So my previous post was very much in line with how I understood Prisma's documentation of the `$queryRaw` function, (or at least how their documentation used to look). You can see an extract here:

![image](/assets/img/2024-04-02-prisma-2/docs-extract.png){: .blog-image}

(See original documentation [at this archive link](https://web.archive.org/web/20240229151956/https://www.prisma.io/docs/orm/prisma-client/queries/raw-database-access/raw-queries#raw-queries-with-relational-databases).)

My feeling from the documentation was that the `$queryRaw` and `$executeRaw` functions were safe but if you wanted to generate a query dynamically into a variable somewhere other than directly in these functions, it would not be possible.

I was currently looking at a use case which required this type of dynamic query generation so I was expecting that developers would therefore not be able to use `$queryRaw` and that instead they would need to use `$queryRawUnsafe`.

### Inconceivable...or is it?

But, as I said above, I had underestimated developers! They came up with something like this stunning piece of code which got me scratching my head as to why on earth it worked.

```js
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

const untrustedInput = 'lomo@prisma.io\' OR \'1\'=\'1'
const sql = `
  SELECT * FROM "User" WHERE email = '${untrustedInput}';
`

const templateString: any = [sql];
templateString.raw = sql;

const users = await prisma.$queryRaw(templateString)
console.log(users)
```

Headscratcher or not, this works and in the example above leads to an SQL inection vulnerability, despite using the "safe" method.

> **Note:** You can try the examples in this section in the [Prisma Playground](https://playground.prisma.io/examples/advanced/raw-queries/with-argument) although be aware that:
>
> - Prisma Playground sometimes errors out so you might need a few attempts before it works...
>   - Alternatively, you can try [this offline version](https://github.com/BounceSecurity/prisma-playground-simulator) I created.
> - Prisma Playground runs JavaScript so you may need to take that into account. For example, in the version above you will need to remove the `: any` from `const templateString: any = [sql];` in order for the code to run.

### So, why does this work?

The short reason is that we are talking about JavaScript where pretty much anything goes 😂. (Well actually, this was Typescript, but it turns out you can persuade it to act like JavaScript without too much trouble 🤦‍♂️.)

The long reason for why this works is that the `$queryRaw` function accepts two possible object types, the [TemplateStringsArray](https://microsoft.github.io/PowerBI-JavaScript/interfaces/_node_modules_typedoc_node_modules_typescript_lib_lib_es5_d_.templatestringsarray.html) type (which Typescript uses as the object type for Tagged Templates that I described in the previous post) or an [Sql](https://github.com/blakeembrey/sql-template-tag) object which itself is based on Tagged Templates.

A Tagged Template has a property called raw and through the use of the any keyword, the developer has created an object that looked similar enough to a Tagged Template object to be accepted by the `$queryRaw` function in both JavaScript and Typescript.

### A simpler bypass

In fact, it turns out that you don't even need to go to this much effort. Prisma supplies a helper method called raw which pretty much does this for you but for the Sql type so now making this function unsafe is as simple as the following code.

```ts
import { Prisma, PrismaClient } from '@prisma/client'

const prisma = new PrismaClient();

const untrustedInput = 'lomo@prisma.io\' OR \'1\'=\'1'

const users = await prisma.$queryRaw`
 SELECT * FROM "User" WHERE email = '${Prisma.raw(untrustedInput)}';
`
console.log(users)
```

I did another LinkedIn poll to see if people would pick up on this but there were still people who thought that this was not vulnerable.

![image](/assets/img/2024-04-02-prisma-2/linkedinpoll.png){: .blog-image-tall .blog-image }
<https://www.linkedin.com/feed/update/urn:li:activity:7175089981831405568/>

If you dig deep enough, you can see warnings about this particular function and it is also a lot easier to discover if it is being used than the previous method. But more on that later.

## So now what?

### The best option for being safe

Having seen all this, I decided that I need to find a way of making the unsafe usage of the safe function safe again.

The best way of doing this would be to build a safe `Sql` object which includes parameter markers and parameters and then pass that to the `$queryRaw` function.

Unfortunately, this is still not a usable solution if you want to dynamically build the query bit by bit as you need to have all the text strings surrounding the parameters in separate variables which is pretty fiddly.

```ts
// Example is safe if the text query below is completely trusted content
const query1 = `SELECT id, name FROM "User" WHERE name = ` // The first parameter would be inserted after this string
const query2 = ` OR name = ` // The second parameter would be inserted after this string

const inputString1 = "Fred"
const inputString2 = `'Sarah' UNION SELECT id, title FROM "Post"`

const query = Prisma.sql([query1, query2, ""], inputString1, inputString2)
const result = await prisma.$queryRaw(query);
console.log(result);
```

### Playing the Uno reverse code

![image](/assets/img/2024-04-02-prisma-2/unoreverse.png){: .blog-image}

In the end, I was inspired by the original bypass code to generate my own `Sql` object but this time create it safely with parameters. You technically shouldn't be able to generate the `Sql` object in this way because the `values` property is readonly and it won't support multiple databases but since this is Javascript we can get away with it 🙃.

This code allows the dynamic generation of a query, using parameterization, which could then be used with `$queryRaw` or `$executeRaw`.

```ts
// Version for Typescript
const query: any

// Version for Javascript
const query

// Safe if the text query below is completely trusted content
query = Prisma.sql`SELECT id, name FROM "User" WHERE name = $1`

// inputString can be untrusted input
const inputString = `'Sarah' UNION SELECT id, title FROM "Post"`
query.values = [inputString]

const result = await prisma.$queryRaw(query)
console.log(result)
```

### But is it safe?

So where does this leave us? This updated code is safe at the moment but it could still be made unsafe.

This type of dynamic query building will always need to be done with caution, with careful attention being paid to how the queries are being built. If the dynamic queries are being built programmatically, hopefully you can be more confident that this treatment is being correctly applied. If developers are still manually building these dynamic queries one by one, you may have more reason for concern

In the meantime, I submitted some quite extensive updates to the Prisma documentation to hopefully make these considerations clearer. You can see the updated documentation [at this link](https://www.prisma.io/docs/orm/prisma-client/queries/raw-database-access/raw-queries#sql-injection-prevention).

(Thanks to the team at Prisma for being open to my suggestions 😀)

## In conclusion

Well at a meta-level, this is another great illustration of the day to day challenges in software security. Sometimes you can't even rely on the documentation...

I think the key conclusions here are:

- Raw queries will always be dangerous and require extra attention because they are so easy to get wrong.
- This is one of the reasons why SQL injection is still so prevalant and we certainly haven't eradicated it yet.
- Being able to detect these sort of edge cases or changes in the way things are done is super important. In a future post, we'll try and demonstrate how...





