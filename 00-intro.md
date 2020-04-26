# Introduction to Vue Design Patterns

## What is this book all about?

Put simply, this book is a handy guide to solving some common requirements / recurring problems in web app developement, but within the scope of a Vue application.

We do this through some design patterns that we discuss in detail in the following chapters.

## What are design patterns?

When you develop an app, you find that some problems (or patterns) keep appearing over and over again. For example, most components that you create will have a loading state, an error state and a state of showing data (including no data).

Over time, as a collective, we've come to identify recurrent problems (or requirements) and find common/generic solutions for such situations.

These become design patterns.

As we progress as developers, one key ingredient to becoming good developers seems to be the knack of finding a general pattern out of a specific problem and then trying to find a solution for the general pattern/problem. This way of working through abstraction helps us solve a class of problems in one-go instead of cranking out a solution for each specific problem individually. This is how foundations for frameworks and libraries are laid.

## How can design patterns help?

Take the example of a component that needs to have a loading, a success (data) and an error state. This is somewhat common for most components that you create.

At your work (or in your projects), you will find yourself writing a similar logic to render the component based on what internal state it has (loading, success or error) over and over again. With some minor differences, you will pretty much be writing the same logic for every component you create.

There are many ways you can reduce the grunt-work of writing the same logic for each component: each solution is a kind of a design pattern that you can abstract away and re-use in each component you create (that needs to show those three states). 

What you'll end up doing is save time once your design pattern is ready. This is one of the most important benefits of using design patterns.

But there's something to be said about design patterns (or abstractions) beyond saving time or avoiding grunt-work. Trying to be aware of repetitive (similar or same) logic across your code, figuring out the common parts of the logic (the common patterns) and then extracting them out into a generic solution – all of this is what makes writing code fun. (Sure there are times when this is insanely frustrating, but I think that's part of the process).

## Who is this book for?

This is not a beginner's book. If you are new to Vue.js or if you have not spent some time writing an application or two in Vue.js, you might not find this book to be very useful. 

That said, you will find this useful if:
- you understand basic ideas of Vue.js
- you feel like you have plateaued when it comes to Vue and need some new ideas to play with and put into practice at work / projects
- you realize you are creating applications with very similar peripheral logic in most components you create
- you want to just find out what's in the book because the table of contents piqued your interest

## A Note on Code samples

WIP 

### Lazy loading components

WIP