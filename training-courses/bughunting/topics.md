---
title: "Accurate and Scalable: Web Application Bug Hunting"
subtitle: Automating your research while keeping it customized to better uncover vulnerabilities
layout: service
hero_height: is-small
backlink: /training
backname: Training
menubar: bughunting_menu
toc: false
---

### Course Content

The interesting, important, and hard to find bugs are not generic. They often stem from the unique business logic of the product, so they require familiarity with it.

While analyzing a web application for vulnerabilities, you probably find yourself performing lots of manual work. Whether it’s looking for interesting issues in the codebase or suspicious behavior in the running application, it probably feels repetitive, tedious and time consuming. All this means that you might miss certain areas of a large-scale app which you simply don’t have time to investigate deeply enough.

Sure, generic scanning tools and fuzzers can help, but often they won't be smart enough to find exactly what you are looking for.

### Summary of topics covered

In this course, you’ll learn how to use customizable scanning tools in order to discover those sneaky vulnerabilities, at scale.
With topics including:

- Optimizing your work through an assisted manual search, that checks all the relevant places in the app in a matter of seconds.
- Finding generic but hard to discover vulnerabilities that other tools miss, thanks to your understanding of the application and its flows.
- Crafting custom security rules specifically designed for the application's business logic, such as flows involving payment or booking.
- Looking for vulnerabilities in dependencies (e.g., CVEs), answering questions such as “does it affect this app?” and “if it does, is it exploitable?” .
- Finding the places where custom, application specific, security mechanisms are not implemented correctly .
- Addressing false positives created by generic tools.
- Learning how to make the above processes repeatable, by building continuous verification and regression testing mechanisms.

The course is focused on a hands-on approach, with multiple examples and exercises. We’ll work with 2 free, open-source tools:

- Semgrep for static analysis
- Nuclei for dynamic analysis

**[Back to top ↑](#top)**

### Two day training breakdown

The first day is focused on static analysis. You will learn how you can find interesting patterns while keeping it specific and omitting false positives, using customizable tools. For example, how to check whether a CVE in a dependency is exploitable in the application, how to track the flow of tainted data and find the root cause and how to do all that on the scale of a whole app when you approach a project.

The second day is focused on dynamic analysis. You will learn how you can dynamically test an issue everywhere in the app at once, using customizable, dynamically flexible tools. For example, checking for unique issues that stem from the business logic of the application, looking for known vulnerabilities, following complex flows that go through multiple protocols and how to approach a project of dynamically testing an application using custom automation.

**[Back to top ↑](#top)**

### Key outcome

You’ll leave the course with a clear understanding of how to automate your work while keeping it customized, so you’ll uncover vulnerabilities efficiently and continuously.

**[Back to top ↑](#top)**