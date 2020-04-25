# Contents

- [Introduction To Vue Design Patterns](01-intro.md)
    - [What is this book all about?](01-intro.md#what-is-this-book-all-about)
    - [What are design patterns?](01-intro.md#what-are-design-patterns)
    - [How can design patterns help?](01-intro.md#how-can-design-patterns-help)
    - [Who is this book for?](01-intro.md#who-is-this-book-for)
    - [A note on code samples](01-intro.md#a-note-on-code-samples)

- [Design Pattern #1: Creating Components Made Easy](02-creating-components-made-easy.md)
    - [Introduction](02-creating-components-made-easy.md#introduction)
    - [Use Case / Example](02-creating-components-made-easy.md#use-case--example)
    - [Renderless Components and Scoped Slots](02-creating-components-made-easy.md#renderless-components-and-scoped-slots)
    - [Renderless to Render Component: Custom Slots and Slot Props](02-creating-components-made-easy.md#renderless-to-render-component-named-slots-with-props)
    - [Going Deeper with Global Source (via Store)](02-creating-components-made-easy.md#going-deeper-with-global-source-via-store)

- Design Pattern #2: Extracting API Response Handlers Out of Store Actions
    - intro = API request data & responses cannot be used as-is. they are tweaked. often when backend and frontend dev happens asynchronously (or very independently), or when API responses are in flux, this pattern comes handy
    - skeleton idea implementation = a handler helper function that "intercepts" the API requests made in store actions
    - expanding on the skeleton = extracting the whole of API handling outside the store as a helper with a dictionary and then using it in store actions
    - going even further = extracting model (business logic) from store and using store only as a reactive interface. using Elm architecture as a theme for modeling data used in the app.

- Design Pattern #3: Unleashing The True Potential Of v-model In Complicated Forms And Other Places
    - intro = establishing the event emitter pattern as something with a lot of overheads and work; looking at built-in form event handling: v-model.
    - skeletal idea = building a simple multi-input form that uses v-model to update prop data (vue typically warns you when you update prop directly)
    - extending skeleton = handing functions to forms a la React and letting them handle their own updates

- Design Pattern #4: Faster Development And Component Access With Global Component Installs
    - intro = the logic of not having to call/import each component of your app wherever it is used; where it helps; where it does not help (async imports)
    - writing a simple installer function to import all your components and use Vue.use()
    - automating the idea using a custom nodejs script to do the above
