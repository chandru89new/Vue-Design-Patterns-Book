# Contents

- [Introduction To Vue Design Patterns](00-intro.md)
    - [What is this book all about?](00-intro.md#what-is-this-book-all-about)
    - [What are design patterns?](00-intro.md#what-are-design-patterns)
    - [How can design patterns help?](00-intro.md#how-can-design-patterns-help)
    - [Who is this book for?](00-intro.md#who-is-this-book-for)
    - [A note on code samples](00-intro.md#a-note-on-code-samples)

- [Design Pattern #1: Creating Components Made Easy](01-creating-components-made-easy.md)
    - [Introduction](01-creating-components-made-easy.md#introduction)
    - [Use Case / Example](01-creating-components-made-easy.md#use-case--example)
    - [Renderless Components and Scoped Slots](01-creating-components-made-easy.md#renderless-components-and-scoped-slots)
    - [Renderless to Render Component: Custom Slots and Slot Props](01-creating-components-made-easy.md#renderless-to-render-component-named-slots-with-props)
    - [Going Deeper with Global Source (via Store)](01-creating-components-made-easy.md#going-deeper-with-global-source-via-store)

- [Design Pattern #2: Unleashing The True Potential Of v-model In Complicated Forms And Other Places](02-v-model-on-steroids.md)
    - [Introduction](02-v-model-on-steroids.md#introduction)
    - [Two-way Communication: Child Components, Props and Event Emitters](02-v-model-on-steroids.md#two-way-communication-child-components-props-and-event-emitters)
    - [Letting Child Components Run the Roost](02-v-model-on-steroids.md#letting-child-components-run-the-roost)
    - [Using with Store](02-v-model-on-steroids.md#using-with-store)
    - [But What About The Loading States?](02-v-model-on-steroids.md#but-what-about-the-loading-states)

- [Design Pattern #3: Extracting API Response Handlers Out of Store Actions](03-extracting-api-out-of-store.md)
    - [Introduction](03-extracting-api-out-of-store.md#introduction)
    - [Extracting Fetch](03-extracting-api-out-of-store.md#extracting-fetch)
    - [Extending the Library With Definitions](03-extracting-api-out-of-store.md#extending-the-library-with-definitions)
    - [Using Transformations In Definitions](03-extracting-api-out-of-store.md#using-transformations-in-definitions)

- Design Pattern #3: Abstract Store
    - intro = repetitive code in store example;
    - building on previous pattern: all logic outside store, so why should your app's data model be inside the store?
    - extracting model vs abstracting model
    - generic mutations to get rid of all others
    - generic actions (while keeping some special actions that involve multiple models)

- Design Pattern #5: Faster Development And Component Access With Global Component Installs
    - intro = the logic of not having to call/import each component of your app wherever it is used; where it helps; where it does not help (async imports)
    - writing a simple installer function to import all your components and use Vue.use()
    - automating the idea using a custom nodejs script to do the above
