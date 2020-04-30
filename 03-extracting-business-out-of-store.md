# Design Pattern #3: Extracting API Response Handlers Out of Store Actions

## Introduction

In idiomatic Vue, all API/backend interactions are typically relegated to store actions. This is a very good way to keep the point where we interact with the backend data in one comfortable place from where we can make modifications as and when things change in the backend responses.

As part of this, developers write a lot of `fetch`-like calls in applications that use a RESTful API. And as you can see where we're going with this, that's a lot of repetition.

Think about the things a `fetch` (or a `fetch`-like call) has to do/handle:

- it has to call an endpoint/URL with specific parameters for the HTTP method, headers, request data, query params etc.
- once the backend responds, it has to handle the response. If successful response, it has to process the data and hand it over to the function or expression that called it. If error, it has to process the error and perhaps add some more information (like status) etc.

All of these things can be generalized for most API calls. And let's look at how we can make that possible.

Note: these things, in a strict sense, fall outside Vue's ecosystem of design patterns. However, they fit neatly into the idea of using store actions to run API calls.

## Extracting Fetch Away to Handle Errors Gracefully
- data structure of response
- a library for fetch/axios
- usage in actions

Here's an example of a store actions object that calls two different endpoints:

```js
actions: {

}
````

## Extending The Library With Definitions
- url and methods appear frequently: can we extract them? 
- replacing that in the library above

## Using Transformations In Definitions
- highlight transformations
- good use cases: 1) API in flux, 2) component state names decoupled from API keys
