definitions:
```
const defs = {
    getUserInfo: {
        url: '/user/info',
        input: ...,
        output: ...,
        auth: false
    },
    getUserAPIKey: {
        url: '/user/info/api_key',
        method: 'post',
        input: ...,
        output: ...
    }
}
```


fetcher:
```
const fetch = def => async payload => {
    const res = await axios[method]({ url, data, ...(auth && { headers }) })
    return { data: res.data, error: res.error, actual: res }
}
```

store action:
```
actions: {
    async getUserInfo(ctx, {id}) {
        ctx.commit('fetchingUser')
        const res = await fetch(def.getUserInfo)({ id })
        if (res.data) { ctx.commit('fetchUserSuccess', res.data) }
        if (res.error) { ctx.commit('fetchUserError', res.actual) }
        ctx.commit('fetchUserEnd')
    }
}
```