# Lets begin

This article will show some tools ant tricks to work with vue more comfortable and fast.

In context of current article we will:
1. Get rid from global storage. `Goodbye Vuex`. Dont panic and skip this article, just watch till the end :)
2. Use DI container for services and state management
3. Know how to reuse common code in our projects

## Just reuse it!

##### Problem:
There are many common tasks per every project we do all the times and just copy/past with refactoring from prev project.
Those parts are too domain related to be moved to package but still very common.

##### Example:
Lets look on most common case - Auth.
Very common for all projects but at same time very domain related.

our Auth service should have relation to User, UserRepository, Storage, Logger
and those most likely will be different per projects, or at least some parts.

```typescript
export class Auth {
  private userRepo!: UserRepo //  <----
  private storage!: Storage //  <----
  private logger: Logger | null = null  //  <----
  private currentUser: User | null = null   //  <----
  //.... other code....
}
```

Those parts could not be published to npm or moved to separate repo. 


##### Solution:
So here come our hero: `Dependency Injection` (DI) and `inversion of control` (IoC).

Lets replace  `UserRepo`, `Storage`, `Logger`, `User` classes to  interfaces

```typescript
export class Auth<U> {
   userRepo!: UserRepoI
   storage!: StorageI
   logger: LoggerI | null = null
   currentUser: (U & UserI) | null = null
  //.... other code....
}
```

Now we can move this Service to npm package.
And later install as dependency to our project.
```typescript

// ...some vue related imports...
import {Auth} from '@aeq/client-http-auth'
import {User} from './entities/User.ts'
import {UserRepo} from './repos/UserRepo.ts'
import {StorageService} from './services/StorageService.ts'
import {LoggerService} from './services/LoggerService.ts'

const auth = new Auth<User>()
auth.userRepo = new UserRepo(...)
auth.storage = new StorageService(...)
auth.logger = new LoggerService(...)

new Vue(...)
```

Of course this is not very comfortable so lets refactor our code using DI container in example of `typedi` package.
and with the power of Typescipt experimental feature `Decorators`
So our Auth service will look like

```typescript
import { Inject, Service, Token } from 'typedi'

export const UserRepoService = new Token<UserRepoI>()
export const StorageService = new Token<StorageI>()
export const LoggerService = new Token<LoggerI>()

@Service()
export class Auth<U> {
  @Inject(UserRepoService)
  private userRepo!: UserRepoI
  @Inject(StorageService)
  private storage!: StorageI
  @Inject(LoggerService)
  private logger: LoggerI | null = null
  private currentUser: (U & UserI) | null = null
  //...
}
```

and now in our `main.ts` we can do like this:

```typescript

// ...some vue related imports...
import { Container } from 'typedi'
import { Auth, LoggerService, StorageService, UserRepoService } from '@aeq/client-http-auth'
import {UserRepo} from './repos/UserRepo.ts'
import {StorageService} from './services/StorageService.ts'
import {LoggerService} from './services/LoggerService.ts'

Container.set(UserRepoService, UserRepo)
Container.set(StorageService, localStorage)
Container.set(LoggerService, LoggerService)


new Vue(...)
```
Lets improve this code a bit more:
We will Create new class `AppServiceProvider`. Its responsibility will be Service configurations.

```typescript
import { Container } from 'typedi'
import { Auth, LoggerService, StorageService, UserRepoService } from '@aeq/client-http-auth'
import {UserRepo} from './repos/UserRepo.ts'
import {StorageService} from './services/StorageService.ts'
import {LoggerService} from './services/LoggerService.ts'

export class AppServiceProvider{

  static boot(){
     Container.set(UserRepoService, UserRepo)
     Container.set(StorageService, localStorage)
     Container.set(LoggerService, LoggerService)
  }

}
```
So now our `main.ts` will look like:

```typescript
// ...some vue related imports...
AppServiceProvider.boot()

new Vue(...)
```

But how to use Auth service now?
- EASY!

Lets look to `AuthLoginForm.vue` component 

```typescript
import { Component, Emit, Vue } from 'vue-property-decorator'
import { Inject } from 'vue-typedi'
@Component
export default class AuthLoginForm extends Vue {
  user: User = new User()

  @Inject()
  auth!: Auth<User>

  @Emit('success')
  async loginCom () {
    return this.auth.login(this.user)
  }
}
```
And since Auth was configured as singleton, Container will return
always same instance and we will have shared state.
So we do not need any global storage. Every time we need shared state,
we will use Singleton stateful service.
As in example with Auth

```typescript
  //.... imports ....
@Service()
export class Auth<U> {
  //.... other code....
  private currentUser: (U & UserI) | null = null
  //.... other code....
  async login (cred: AuthLoginCredential): Promise<void> {
    // login logic
    this.fetchUser()
  }

  async fetchUser(){
    // some code
    this.currentUser = await this.userRepo.me() as U & User
    // some code
  }

  //.... other code....
}
```

And later somewhere in vue pages

```vue
<script lang="ts">
import { Component, Emit, Vue } from 'vue-property-decorator'
import { Inject } from 'vue-typedi'
import { User } from '../../../entities/User'
import { Auth } from '@aeq/client-http-auth'


@Component
export default class AuthLoginForm extends Vue {
  credentials: User = new User()

  @Inject()
  auth!: Auth<User>
  
}
</script>

<template>
 <div>Hello {{auth.currentUser.name}}</div>
</template>
```

That`s it!

But you may ask:
- WAIT! But how can i control mutations of global state? Vuex provides devtool with snapshots.

Yes. Sometimes we need to have ability control global state.
And for this reason we may use Proxy javascript object.

lets create some ES module and call it global store:

```typescript
export class GlobalStore {
  auth: Auth<User>
}

```


in `AppServiceProvider`

```typescript
import { Container } from 'typedi'
import { Auth, LoggerService, StorageService, UserRepoService } from '@aeq/client-http-auth'
import {UserRepo} from './repos/UserRepo.ts'
import {StorageService} from './services/StorageService.ts'
import {LoggerService} from './services/LoggerService.ts'
import {GlobalStore, theGlobalStore} from './store/GlobalStore.ts'

export class AppServiceProvider{

  static boot({store}: {store: GlobalStore}){
     Container.set(UserRepoService, UserRepo)
     Container.set(StorageService, localStorage)
     Container.set(LoggerService, LoggerService)

     
     AppServiceProvider.debugServices()

  }
  
 static debugServices(){
    const p = new Proxy(theGlobalStore, {
       set(obj: GlobalStore, prop: string, value: any) {
         console.log('Prop was changed from ', obj[prop])
         obj[prop] = value;
         console.log('To ', value)
       }
    })
  }
}
```

For more options we can use `on-change` package based on proxies and write small interface like devtool in google chrome when needed.
Estimation for this will be near `1-2 days`

But for most cases console.log will be more then enough

And wy we should get rid from VUEX at all?
... well this theme deserves a separate article :)
In short it is ugly tool that force you write a lot of code.
And the more code the more bugs.
No ide support...
No TS support...
No classes support...


### Lets go more!

Some packages we already have:
#### `@aeq/http-errors`
normalized and consistent typed Errors for http common errors.
Abilities:
- ApiError
- AppError
- NetworkError
- NotFoundError
- UnauthorizedError

#### `@aeq/api-laravel`

laravel api abstraction layer. wrappes axios instance. But we are free to inject any object for 
ajax request with same interface like axios.
Abilities:
- Throws typed exception on api errors. uses `@aeq/http-errors`
- Abstraction over Laravel validation errors
examples:
```typescript
error.getError('name') //returns string
error.addError('name', 'required')
error.clear()
error.hasAny 
```
and so on...
- transform classes to plain objects. uses `class-transformer` package
```typescript
class User {
  name: string
  @Expose({name: 'full_name'})
  get fullName(){}
}
const api = Container.get(ApiLaravel)
api.post('users', new User())
```
#### `@aeq/executors` 

Implementations of Command pattern. Inspired by Java Executors.
Each `executor` simply wrap some function and controls its execution.

```typescript
async function fetchUsersCommand(): Promise<User[]>{
  return api.get('users')
}

const fetchUsers = new Executor(fetchUsersCommand)
fetchUsers.isRunning // false
fetchUsers.run() // returns promise
fetchUsers.isRunning // true
fetchUsers.wasRun
fetchUsers.wasLastRunFine
``` 
and so on...

List of available executors: `CacheExecutor`, `Debouncer`, `Executor`, `HoldExecutor`, `LadderExecutor`
upcomming `InfiniteLoader`

#### `@aeq/form` 

Some kind of Executor specilized on forms handling

#### Small utility packages
`@aeq/promise`
`@aeq/obj`
`@aeq/str`
`@aeq/arr`


### SUMMARIZE the workflow

Lets take a look on simple login example to show the work flow


##### 1. create vuew app in `main.ts`
```typescript
import 'reflect-metadata'
import Vue from 'vue'
import App from './App.vue'
import router from './router'
import store from './store'
import { AppServiceProvider } from './providers/AppServiceProvider'
import { theStore } from './store/theGlobalStore'

AppServiceProvider.boot()

Vue.config.productionTip = false

new Vue({
  router,
  store,
  data: {
    store: theStore
  },
  render: h => h(App)
}).$mount('#app')

```

##### 2. Configure AppServiceProvider

```typescript
export class AppServiceProvider {
  static boot () {
    const axiosInstance = this.createAxios()

    Container.set(StorageService, localStorage)
    Container.set(LoggerService, null)
    Container.set(HttpService, axiosInstance)
    Container.set({ id: UserRepoService, type: UserRepo })
    store.auth = Container.get(Auth) as Auth<User>
    AuthServiceProvider.boot() // configures axios interseptors to send Bearer token per each request
    
  }

  private createAxios () {
    const axiosInstance = axios.create({
      baseURL: theApiLaravelConfig.url,
      headers: {
        'Content-Type': 'application/json'
      },
      paramsSerializer: (params: any) => {
        return Qs.stringify(params, { encodeValuesOnly: true })
      }
    })
    return axiosInstance
  }
}

```

##### 3. Prepare Config service 
```typescript
import { Service } from 'typedi'

@Service()
export class Config {
  readonly isDev: boolean = process.env.VUE_APP_APP_ENV === 'local'
  readonly appId: string = process.env.VUE_APP_APP_ID || ''
  readonly clientId: string = process.env.VUE_APP_CLIENT_ID || ''
  readonly clientSecret: string = process.env.VUE_APP_CLIENT_SECRET || ''
  readonly apiUrl: string = process.env.VUE_APP_API_URL || ''
  readonly assetsUrl: string = process.env.VUE_APP_ASSETS_URL || ''
}
```


##### 3. Create User Entity

We should implement `AuthUser` to ensure our entity will fit Auth service interface

```typescript
import { Responsibility } from '../Responsibility/Responsibility'
import { User as AuthUser } from '@aeq/client-http-auth'

export class User implements AuthUser {
  id: string = ''
  name: string = ''
  phone: string = ''
  email: string = ''
  password: string = ''
  remember_token: string = ''
  notify_email: boolean = false
  notify_phone: boolean = false
  notify_web: boolean = false
  current_organization_id: string = ''
  responsibilities: Responsibility[] = []

  get username (): string {
    return this.email
  }

  set username (v: string) {
    this.email = v
  }

  constructor (data: Partial<User> = {}) {
    Object.assign(this, data)
  }
}

```

##### 4 Create abstraction for User requests `UserRepo`

We are using `class-transformer` package for mapping plain js objects to Entities and transform Entities to plain objects



```typescript
import { UserQuery } from './UserQuery'
import { User } from './User'
import { Inject, Service } from 'typedi'
import { TransformPlainToClass } from 'class-transformer'
import { ApiLaravelTransformable } from '@aeq/api-laravel'
import { AuthToken, BearerCredential, UserRepo as AuthUserRepo } from '@aeq/client-http-auth'
import { Config } from '../../config/config'

@Service()
export class UserRepo implements AuthUserRepo {
  @Inject()
  api!: ApiLaravelTransformable

  @Inject()
  cfg!: Config

  @TransformPlainToClass(AuthToken)
  async login (username: string, password: string): Promise<AuthToken> {
    const c = new BearerCredential({
      username,
      password,
      clientId: this.cfg.clientId,
      client_secret: this.cfg.clientSecret,
      remember_me: true
    })
    return this.api.post('authenticate', c)
  }

  @TransformPlainToClass(User)
  async register (user: User): Promise<User> {
    return this.api.post('users', user)
  }

  @TransformPlainToClass(User)
  async me (q: UserQuery = new UserQuery()): Promise<User> {
    return this.api.get('users/me', q)
  }

  @TransformPlainToClass(User)
  async update (user: User): Promise<User> {
    return this.api.put(`users/${user.id}`, user)
  }

  async resetPassword (email: string): Promise<void> {
    const payload: Partial<User> = { email }
    await this.api.post('/password/reset', payload)
  }

  async ping (): Promise<string> {
    return this.api.get('/ping')
  }
}
```
`@TransformPlainToClass` derorator will map plain object returend from method to Class provided in args
for more info look `class-transformers` docs 


##### 5. App.vue

```vue
<template>
  <div :class="$style.main">
    <router-view/>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'
import { theStore } from './store/theGlobalStore'

export default Vue.extend({
  name: 'App',
  components: {},
  created () {
    theStore.auth.init()
  }
})
</script>
<style lang="scss" module>
.main {

}
</style>
```
For scoping styles lets use css modules. They are more flexible neither `vue scoped styles`.

##### 6. Login page
```vue
<template>
  <div>
      <auth-login-form @success="redirect"/>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator'
import AuthLoginForm from '../../components/Auth/Login/AuthLoginForm.vue'

@Component({
  components: { AuthLoginForm }
})
export default class Login extends Vue {
  $refs: any

  redirect () {
    this.$router.replace({ name: 'project.list' })
  }
}
</script>

<style lang="scss">
.login {
  height: 100%;
}
</style>

```

##### 7. create login form

We are using `vue-typedi` package to inject dependencies from DI container to Vue components

```vue
<template>
  <div>
    <base-input
      label="Login"
      name="login"
      type="text"
      v-model="credentials.username"
      :error-messages="loginErr.username"
    />

    <base-input
      id="password"
      label="Password"
      name="password"
      type="password"
      v-model="credentials.password"
      :error-messages="loginErr.password"
    />
  <base-btn
    color="primary"
    text
    @click="login.run()"
    :loading="login.isRunning"
  >
    Login
  </base-btn>
  </div>
</template>

<script lang="ts">
import { Component, Emit, Vue } from 'vue-property-decorator'
import { Inject } from 'vue-typedi'
import { User } from '../../../sdk/User/User'
import { Auth } from '@aeq/client-http-auth'
import { Form } from '@aeq/form'

@Component
export default class AuthLoginForm extends Vue {
  credentials: User = new User()

  @Inject()
  auth!: Auth<User>

  login = new Form(this.loginCom, this.credentials)

  get loginErr (): Partial<User> {
    return {
      username: this.login.getError('username'),
      password: this.login.getError('password'),
    }
  }

  @Emit('success')
  async loginCom (user: User) {
    return this.auth.login(user)
  }
}
</script>
```

THats it! Hope you like new strategy!

