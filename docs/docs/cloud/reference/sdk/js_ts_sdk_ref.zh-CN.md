
<a name="readmemd"></a>

**[@langchain/langgraph-sdk](https://github.com/langchain-ai/langgraph/tree/main/libs/sdk-js)**

***

## [@langchain/langgraph-sdk](https://github.com/langchain-ai/langgraph/tree/main/libs/sdk-js)

### 类 (Classes)

- [AssistantsClient](#classesassistantsclientmd)
- [Client](#classesclientmd)
- [CronsClient](#classescronsclientmd)
- [RunsClient](#classesrunsclientmd)
- [StoreClient](#classesstoreclientmd)
- [ThreadsClient](#classesthreadsclientmd)

### 接口 (Interfaces)

- [ClientConfig](#interfacesclientconfigmd)

### 函数 (Functions)

- [getApiKey](#functionsgetapikeymd)


<a name="authreadmemd"></a>

**@langchain/langgraph-sdk**

***

## @langchain/langgraph-sdk/auth

### 类 (Classes)

- [Auth](#authclassesauthmd)
- [HTTPException](#authclasseshttpexceptionmd)

### 接口 (Interfaces)

- [AuthEventValueMap](#authinterfacesautheventvaluemapmd)

### 类型别名 (Type Aliases)

- [AuthFilters](#authtype-aliasesauthfiltersmd)


<a name="authclassesauthmd"></a>

[**@langchain/langgraph-sdk**](#authreadmemd)

***

[@langchain/langgraph-sdk](#authreadmemd) / Auth

## 类: Auth\<TExtra, TAuthReturn, TUser\>

定义于: [src/auth/index.ts:11](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/index.ts#L11)

### 类型参数 (Type Parameters)

• **TExtra** = \{\}

• **TAuthReturn** *extends* `BaseAuthReturn` = `BaseAuthReturn`

• **TUser** *extends* `BaseUser` = `ToUserLike`\<`TAuthReturn`\>

### 构造函数 (Constructors)

#### new Auth()

> **new Auth**\<`TExtra`, `TAuthReturn`, `TUser`\>(): [`Auth`](#authclassesauthmd)\<`TExtra`, `TAuthReturn`, `TUser`\>

##### 返回 (Returns)

[`Auth`](#authclassesauthmd)\<`TExtra`, `TAuthReturn`, `TUser`\>

### 方法 (Methods)

#### authenticate()

> **authenticate**\<`T`\>(`cb`): [`Auth`](#authclassesauthmd)\<`TExtra`, `T`\>

定义于: [src/auth/index.ts:25](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/index.ts#L25)

##### 类型参数 (Type Parameters)

• **T** *extends* `BaseAuthReturn`

##### 参数 (Parameters)

###### cb

`AuthenticateCallback`\<`T`\>

##### 返回 (Returns)

[`Auth`](#authclassesauthmd)\<`TExtra`, `T`\>

***

#### on()

> **on**\<`T`\>(`event`, `callback`): `this`

定义于: [src/auth/index.ts:32](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/index.ts#L32)

##### 类型参数 (Type Parameters)

• **T** *extends* `CallbackEvent`

##### 参数 (Parameters)

###### event

`T`

###### callback

`OnCallback`\<`T`, `TUser`\>

##### 返回 (Returns)

`this`


<a name="authclasseshttpexceptionmd"></a>

[**@langchain/langgraph-sdk**](#authreadmemd)

***

[@langchain/langgraph-sdk](#authreadmemd) / HTTPException

## 类: HTTPException

定义于: [src/auth/error.ts:66](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/error.ts#L66)

### 继承 (Extends)

- `Error`

### 构造函数 (Constructors)

#### new HTTPException()

> **new HTTPException**(`status`, `options`?): [`HTTPException`](#authclasseshttpexceptionmd)

定义于: [src/auth/error.ts:70](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/error.ts#L70)

##### 参数 (Parameters)

###### status

`number`

###### options?

####### cause?

`unknown`

####### headers?

`HeadersInit`

####### message?

`string`

##### 返回 (Returns)

[`HTTPException`](#authclasseshttpexceptionmd)

##### 覆盖 (Overrides)

`Error.constructor`

### 属性 (Properties)

#### cause?

> `optional` **cause**: `unknown`

定义于: node\_modules/typescript/lib/lib.es2022.error.d.ts:24

##### 继承自 (Inherited from)

`Error.cause`

***

#### headers

> **headers**: `HeadersInit`

定义于: [src/auth/error.ts:68](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/error.ts#L68)

***

#### message

> **message**: `string`

定义于: node\_modules/typescript/lib/lib.es5.d.ts:1077

##### 继承自 (Inherited from)

`Error.message`

***

#### name

> **name**: `string`

定义于: node\_modules/typescript/lib/lib.es5.d.ts:1076

##### 继承自 (Inherited from)

`Error.name`

***

#### stack?

> `optional` **stack**: `string`

定义于: node\_modules/typescript/lib/lib.es5.d.ts:1078

##### 继承自 (Inherited from)

`Error.stack`

***

#### status

> **status**: `number`

定义于: [src/auth/error.ts:67](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/error.ts#L67)

***

#### prepareStackTrace()?

> `static` `optional` **prepareStackTrace**: (`err`, `stackTraces`) => `any`

定义于: node\_modules/@types/node/globals.d.ts:28

格式化堆栈跟踪的可选覆盖

##### 参数 (Parameters)

###### err

`Error`

###### stackTraces

`CallSite`[]

##### 返回 (Returns)

`any`

##### 参见 (See)

https://v8.dev/docs/stack-trace-api#customizing-stack-traces

##### 继承自 (Inherited from)

`Error.prepareStackTrace`

***

#### stackTraceLimit

> `static` **stackTraceLimit**: `number`

定义于: node\_modules/@types/node/globals.d.ts:30

##### 继承自 (Inherited from)

`Error.stackTraceLimit`

### 方法 (Methods)

#### captureStackTrace()

> `static` **captureStackTrace**(`targetObject`, `constructorOpt`?): `void`

定义于: node\_modules/@types/node/globals.d.ts:21

在目标对象上创建 .stack 属性

##### 参数 (Parameters)

###### targetObject

`object`

###### constructorOpt?

`Function`

##### 返回 (Returns)

`void`

##### 继承自 (Inherited from)

`Error.captureStackTrace`


<a name="authinterfacesautheventvaluemapmd"></a>

[**@langchain/langgraph-sdk**](#authreadmemd)

***

[@langchain/langgraph-sdk](#authreadmemd) / AuthEventValueMap

## 接口: AuthEventValueMap

定义于: [src/auth/types.ts:218](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L218)

### 属性 (Properties)

#### assistants:create

> **assistants:create**: `object`

定义于: [src/auth/types.ts:226](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L226)

##### assistant\_id?

> `optional` **assistant\_id**: `Maybe`\<`string`\>

##### config?

> `optional` **config**: `Maybe`\<`AssistantConfig`\>

##### graph\_id

> **graph\_id**: `string`

##### if\_exists?

> `optional` **if\_exists**: `Maybe`\<`"raise"` \| `"do_nothing"`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### name?

> `optional` **name**: `Maybe`\<`string`\>

***

#### assistants:delete

> **assistants:delete**: `object`

定义于: [src/auth/types.ts:229](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L229)

##### assistant\_id

> **assistant\_id**: `string`

***

#### assistants:read

> **assistants:read**: `object`

定义于: [src/auth/types.ts:227](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L227)

##### assistant\_id

> **assistant\_id**: `string`

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

***

#### assistants:search

> **assistants:search**: `object`

定义于: [src/auth/types.ts:230](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L230)

##### graph\_id?

> `optional` **graph\_id**: `Maybe`\<`string`\>

##### limit?

> `optional` **limit**: `Maybe`\<`number`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### offset?

> `optional` **offset**: `Maybe`\<`number`\>

***

#### assistants:update

> **assistants:update**: `object`

定义于: [src/auth/types.ts:228](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L228)

##### assistant\_id

> **assistant\_id**: `string`

##### config?

> `optional` **config**: `Maybe`\<`AssistantConfig`\>

##### graph\_id?

> `optional` **graph\_id**: `Maybe`\<`string`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### name?

> `optional` **name**: `Maybe`\<`string`\>

##### version?

> `optional` **version**: `Maybe`\<`number`\>

***

#### crons:create

> **crons:create**: `object`

定义于: [src/auth/types.ts:232](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L232)

##### cron\_id?

> `optional` **cron\_id**: `Maybe`\<`string`\>

##### end\_time?

> `optional` **end\_time**: `Maybe`\<`string`\>

##### payload?

> `optional` **payload**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### schedule

> **schedule**: `string`

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

##### user\_id?

> `optional` **user\_id**: `Maybe`\<`string`\>

***

#### crons:delete

> **crons:delete**: `object`

定义于: [src/auth/types.ts:235](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L235)

##### cron\_id

> **cron\_id**: `string`

***

#### crons:read

> **crons:read**: `object`

定义于: [src/auth/types.ts:233](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L233)

##### cron\_id

> **cron\_id**: `string`

***

#### crons:search

> **crons:search**: `object`

定义于: [src/auth/types.ts:236](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L236)

##### assistant\_id?

> `optional` **assistant\_id**: `Maybe`\<`string`\>

##### limit?

> `optional` **limit**: `Maybe`\<`number`\>

##### offset?

> `optional` **offset**: `Maybe`\<`number`\>

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

***

#### crons:update

> **crons:update**: `object`

定义于: [src/auth/types.ts:234](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L234)

##### cron\_id

> **cron\_id**: `string`

##### payload?

> `optional` **payload**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### schedule?

> `optional` **schedule**: `Maybe`\<`string`\>

***

#### store:delete

> **store:delete**: `object`

定义于: [src/auth/types.ts:242](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L242)

##### key

> **key**: `string`

##### namespace?

> `optional` **namespace**: `Maybe`\<`string`[]\>

***

#### store:get

> **store:get**: `object`

定义于: [src/auth/types.ts:239](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L239)

##### key

> **key**: `string`

##### namespace

> **namespace**: `Maybe`\<`string`[]\>

***

#### store:list\_namespaces

> **store:list\_namespaces**: `object`

定义于: [src/auth/types.ts:241](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L241)

##### limit?

> `optional` **limit**: `Maybe`\<`number`\>

##### max\_depth?

> `optional` **max\_depth**: `Maybe`\<`number`\>

##### namespace?

> `optional` **namespace**: `Maybe`\<`string`[]\>

##### offset?

> `optional` **offset**: `Maybe`\<`number`\>

##### suffix?

> `optional` **suffix**: `Maybe`\<`string`[]\>

***

#### store:put

> **store:put**: `object`

定义于: [src/auth/types.ts:238](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L238)

##### key

> **key**: `string`

##### namespace

> **namespace**: `string`[]

##### value

> **value**: `Record`\<`string`, `unknown`\>

***

#### store:search

> **store:search**: `object`

定义于: [src/auth/types.ts:240](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L240)

##### filter?

> `optional` **filter**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### limit?

> `optional` **limit**: `Maybe`\<`number`\>

##### namespace?

> `optional` **namespace**: `Maybe`\<`string`[]\>

##### offset?

> `optional` **offset**: `Maybe`\<`number`\>

##### query?

> `optional` **query**: `Maybe`\<`string`\>

***

#### threads:create

> **threads:create**: `object`

定义于: [src/auth/types.ts:219](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L219)

##### if\_exists?

> `optional` **if\_exists**: `Maybe`\<`"raise"` \| `"do_nothing"`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

***

#### threads:create\_run

> **threads:create\_run**: `object`

定义于: [src/auth/types.ts:224](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L224)

##### after\_seconds?

> `optional` **after\_seconds**: `Maybe`\<`number`\>

##### assistant\_id

> **assistant\_id**: `string`

##### if\_not\_exists?

> `optional` **if\_not\_exists**: `Maybe`\<`"reject"` \| `"create"`\>

##### kwargs

> **kwargs**: `Record`\<`string`, `unknown`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### multitask\_strategy?

> `optional` **multitask\_strategy**: `Maybe`\<`"reject"` \| `"interrupt"` \| `"rollback"` \| `"enqueue"`\>

##### prevent\_insert\_if\_inflight?

> `optional` **prevent\_insert\_if\_inflight**: `Maybe`\<`boolean`\>

##### run\_id

> **run\_id**: `string`

##### status

> **status**: `Maybe`\<`"pending"` \| `"running"` \| `"error"` \| `"success"` \| `"timeout"` \| `"interrupted"`\>

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

***

#### threads:delete

> **threads:delete**: `object`

定义于: [src/auth/types.ts:222](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L222)

##### run\_id?

> `optional` **run\_id**: `Maybe`\<`string`\>

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

***

#### threads:read

> **threads:read**: `object`

定义于: [src/auth/types.ts:220](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L220)

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

***

#### threads:search

> **threads:search**: `object`

定义于: [src/auth/types.ts:223](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L223)

##### limit?

> `optional` **limit**: `Maybe`\<`number`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### offset?

> `optional` **offset**: `Maybe`\<`number`\>

##### status?

> `optional` **status**: `Maybe`\<`"error"` \| `"interrupted"` \| `"idle"` \| `"busy"` \| `string` & `object`\>

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

##### values?

> `optional` **values**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

***

#### threads:update

> **threads:update**: `object`

定义于: [src/auth/types.ts:221](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L221)

##### action?

> `optional` **action**: `Maybe`\<`"interrupt"` \| `"rollback"`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>
