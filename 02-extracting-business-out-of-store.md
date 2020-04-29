# Design Pattern #2: Extracting API Response Handlers Out of Store Actions

## Introduction

A general trend in Vue applications that make use of Vuex and connect to REST APIs is to use `actions` in the Vuex Store to handle the asynchronous API requests and to trigger mutations based on the request responses.

This serves you well but there is bound to be a lot of business logic inside store this way. Most notably, the transformations.


It is generally considered good practice to decouple the structure of the API response and the structure of the state you derive from it. That is, your components (or your entire app) should be agnostic to how the API response looks. All the transformations to the API responses are done in one place (ie, in your store instead of in your components) so that when API changes, you have only one place to worry about.

This pattern ends up in people writing code that modifies/transforms API responses in the `actions`. It's logical because `actions` is typically where all API calls are made.


API changes may not be common if you are working on a product that's mature and stable but in a new project (or during a revamp), API responses may be in flux. Big changes can happen overnight and frontend integration may be a bottleneck in getting to tests.

To avoid this, you can extract all the API-handling logic outside the store in what can look like a simple helper with a dictionary of rules.

This way, all API-related interface is handled in a separate module and your Vuex Store simply makes use of that module.

Let's take a look at some examples.

## The Interceptor

Imagine a simple Todo application with these features:

- it lets multiple users sign-in/signup
- it lets users create, edit, delete or view "projects". Each project can contain many todos.
- it lets users create, edit, delete or view the todos for each project.

Now, imagine the Vuex store for this application. Typically, the store would be modularized with each entity (users, projects, todos) getting their own modules, but let's just show a quick summary of them here as a single entity:

```js
import Vue from 'vue'
import Vuex from 'vuex'
import axios from 'axios'
Vue.use(Vuex)

export default new Vuex.Store({
    state: {
        user: { data: null loading: false, error: null },
        projects: { data: [], loading: false, error: null },
        todos: { data: [], loading: false, error: null }
    },
    getters: {},
    mutations: {},
    actions: {
        /** user related stuff */
        async getUserInfo(ctx, payload) {},
        async updateUserInfo(ctx, payload) {},

        /** project-related stuff */
        async getProjects(ctx, payload) {},
        async addProject(ctx, payload) {}
        async editProject(ctx, payload) {},
        async deleteProject(ctx, payload) {},

        /** todo-related stuff */
        async getTodosForProject(ctx, payload) {},
        async addTodo(ctx, payload) {},
        async editTodo(ctx, payload) {},
        async deleteTodo(ctx, payload) {},
    }
})
```

While I have not listed the exact things going inside each of the actions, I think it is quite easy to spot some common theme here.

- Each action is going to be an API call. It's going to be an HTTP GET, POST, PUT or DELETE method.
- Each action is typically getting the response from the API, handling errors (if any), and if success, handling the response and transforming it to hand it over the mutation.

As an example, let's take the case of an `axios` success response. The actual API response is wrapped in `response.data` (whereas `response.status` gives you the HTTP status the API responded in).

If you take the example of a [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) (HTML API) response, you need to handle two things: first, convert a successful response into JSON through `response.json()` and then hand it over to the next `then()` of the Promise; in case of an error, you have to handle that too.

Obviously, it doesn't make any sense to write these things over and over again in every action. That's why `axios` has things like interceptors which help in handling requests and responses from API calls at a global level. But that is not enough. We want to be able to handle every API response in a different way but bring them all together under one umbrella module that our store `actions` can make use of.

Let's take some actions from our store above and expand their internals.

First off, let's take the case of getting a todo and showing it on the UI.

The UI component iterates over a simple `Todo` component:

```vue
<template>
    <div class="todo">
        <div v-if="!editMode">
            <input type="checkbox" :checked="todo.checked" :id="todo.id" />
            <label :for="todo.id">{{ todo.description }} </label>
        </div>
        <div v-if="editMode">
            <input type="text" v-model="todo.description" />
        </div>
        <span class="controls">
            <button @click="() => editMode = true" v-if="!editMode">Edit</button>
            <button @click="saveTodo" v-if="editMode">Save</button>
            <button @click="deleteTodo(todo.id)">Delete</button>
        </span>
    </div>
</template>
<script>
...
</script>
```

Basically:

- the component can be in `editMode` or not. If it is not, it lists the Todo item, showing a checkbox, a label for the description of the todo and two buttons: Edit and Delete.
- if user clicks on Edit, it toggles the `editMode` to `true` and the checkbox and label are replaced by a text input which the user can edit. The Edit button is replaced by the Save button which will help the user save the todo (by calling an action, perhaps)

What's most important to note here is the shape of the `todo` prop being sent to this component:

```js
todo = {
    id: string | number,
    description: string,
    checked: boolean
}
````


```js
async getTodosForProject(ctx, payload) {
    ctx.commit('fetchTodosStart')
    try {
        const res = await axios.get('https://example.com/todos/', { params: { id: payload.id }})
        const todos = res.data.todos.map(todo => ({

        }))
        ctx.commit('fetchTodosSuccess', todos)
    } catch (e) {
        ctx.commit('fetchTodosError', res.data)
    }
    ctx.commit('fetchTodosDone')
}
async addTodo(ctx, payload) {}
async editTodo(ctx, payload) {}
async deleteTodo(ctx, payload) {}
```




## Interceptor With Definitions

