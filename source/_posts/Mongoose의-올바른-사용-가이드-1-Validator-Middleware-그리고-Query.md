---
title: 'Mongoose의 올바른 사용 가이드 1 - Validator, Middleware 그리고 Query'
desc: Mongoose에서 문서를 꼼꼼하게 읽지 않으면 저지를 수 있는 실수들을 저지르지 않기 위해 작성된 가이드입니다.
date: 2018-09-10 11:39:00
---

Mongoose에서 문서를 꼼꼼하게 읽지 않으면 저지를 수 있는 실수들을 저지르지 않기 위해 작성된 가이드입니다. Mongoose의 기초적인 사용법과 모델 등에 관해선 다루지 않습니다. 쿼리와 사용할 때 유용한 몇가지 옵션등도 소개하고 있습니다.

<!-- more -->

본 문서를 작성할 당시 Mongoose의 최신 버전은 5.2.13 이었습니다. 버전이 바뀜에 따라 API도 변경됐을 수 있으니 만약 오류가 있다면 제보해주세요.

{% asset_img "cover.jpg" "Photo by Samuel Austin on Unsplash" %}
> Photo by Samuel Austin on Unsplash

# Query

쿼리를 실행(execute)하는 방법에는 두 가지가 있습니다.

1. 콜백 함수를 넘기기
2. `.exec()`으로 `Promise`같이 쓰기

쿼리 인스턴스의 함수 중 콜백을 넘기지 않았을 때 `Promise`를 반환하는(정확히는 `Promise-Like`) 함수는 `create()`, `save()`, `remove()`, `exec()` 등 일부고 그 외엔 모두 자기 자신(쿼리 인스턴스)를 다시 반환하기 때문에, 아래와 같은 구문은 옳지 않습니다. 쿼리 인스턴스에 대해 `await`를 쓰는 것은 말이 안되죠.

따라서 쿼리를 최종 실행시키려면 반드시 `.exec()` 함수를 쓰거나 쿼리의 마지막 매개변수로 콜백 함수를 넘겨줘야 합니다.

```javascript
const user = await User.findOne({ name: 'a1p4ca' }) // Wrong
const transactions = await Transaction
    .find({ user: user._id })
    .limit(5).exec() // Correct
```

# Middleware와 Validators

## Validators

MongoDB에 입력한 데이터의 무결성을 보장합니다.

```javascript
const userSchema = new Schema({
    age: {
        type: Number,
        min: [13, 'Too young'],
        max: 130
    }
})
```

## Middlewares

미들웨어는 MongoDB에서 특정 작업(저장 등)이 실행되기 전/후에 실행되는 함수들입니다.

```javascript
userSchema.pre('save', function (next) {
    this.firstName = capitalize(this.firstName)
    this.lastName = capitalize(this.lastName)
    next()
})
```

## Validation과 Middleware을 우회하는 함수를 주의하세요

Middleware와 Validator가 호출되지 않는 경우가 있습니다. 이 경우 잘못된 데이터를 DB에 저장하려고 해도 모델을 검사하는 모든 작업을 우회하여 그대로 DB에 때려박을 것입니다.

위와 같은 일은 Mongoose가 ORM을 이용한 데이터 핸들링과 DB에 직접 데이터를 때려박는 두가지 기능을 모두 제공하기 때문에 발생합니다. ~~버그가 아니라 기능.~~ 문제는 이러한 '기능'이 문서에 명확하게 나와있지 않고, 단지 조그마한 각주로 예외에 대해서 설명하고 있을 뿐이라는 것입니다.

예시를 들어보겠습니다. **만약 아래의 함수들 중 어느 하나를 특정 옵션 없이 그대로 사용하고 있다면 이미 위 문제를 겪고 있는 것이고, 따라서 해당 코드를 다시 작성해야 할 것입니다.**

```javascript
User.update()
User.findOneAndUpdate()
User.findByIdAndUpdate()

User.remove()
User.findOneAndRemove()
User.findByIdAndRemove()
```

# CRUD

아래 예시들은 Create, Read, Update, Delete (CRUD) 쿼리들을 Validator와 미들웨어를 통과하도록 사용하는 적절한 방법을 제시합니다.

## Create

아래 두 예제는 정확하게 같은 일을 합니다. `Model.create(doc)` 는 `new Model(doc).save()` 과 일치합니다.

```javascript
new User({
    name: 'Brian'
}).save()

User.create({
    name: 'Brain'
})
```

## Read

어떻게 사용하든 상관없습니다. 다만 주의해야 할 점은 `findOne()`와 `find()`는 서로 다른 미들웨어(각각 `findOne`과 `find`)를 호출합니다. 또한 `findById(id)`는 `findOne({ _id: id })`와 같습니다.

```javascript
await User.findOne({}).exec()
await User.findById({}).exec()
```

## Update

쿼리 요청 횟수를 줄이기 위해서는 `find()` ⇒ document modify ⇒ `save()` 의 과정보다 `where()` 을 활용하는 것이 더 효율적이겠습니다. 물론 Validator와 미들웨어도 잘 통과해야하구요. 그러나 `update()` 는 기본적으로 Validator는 실행하지 않습니다. 따라서 옵션으로 `runValidators`를 켜줘야 실수를 막을 수 있습니다. 아래 예제처럼요.

```javascript
await User.where({ _id: id })
    .update({ name: 'Omega' })
    .setOptions({ runValidators: true })
    .exec()
```

`update()`에서 유용한 몇가지 옵션을 소개합니다. 괄호는 디폴트 값입니다.

- `upsert` (`false`) : 만약 참이면 존재하지 않는 문서의 경우 새로 생성합니다.
- `multi` (`false`) : 만약 참이면 여러개의 문서를 업데이트합니다.
- `overwrite` (`false`) : HTTP에서 PUT과 PATCH의 차이로 보시면 됩니다. 만약 참이라면 기존의 문서를 전달한 객체로 교체하고(물론 _id 등은 제외), 거짓이면 전달받은 객체에서 명시한 필드의 값만 변경합니다.

MongoDB에서 `findOneAndUpdate()`는 `findOne()`과 `update()` 쿼리의 조합이 아니라 별개의 쿼리로 실행됩니다. 그리고 Mongoose에서 `Query.findByIdAndUpdate()` 는 `Query.findOneAndUpdate({ _id: id }, ...)` 의 alias입니다(추가적으로
 `Query.findById()` 역시 MongoDB의 자체 쿼리가 아닌 `Query.findOne({ _id: id })` 의 alias입니다). 따라서 두 쿼리를 호출하면 동일하게 `findOneAndUpdate` 훅이 발생합니다.

```javascript
UserSchema.pre('findOneAndUpdate', () => console.log('called'))

await User.findByIdAndUpdate(/* ... */)
    .setOptions({ runValidators: true })
    .exec() // called
await User.findOneAndUpdate(/* ... */)
    .setOptions({ runValidators: true })
    .exec() // called
```

## Delete

삭제에서는 데이터 검증이 중요하지 않습니다. 하지만 미들웨어는 호출해야 할 수도 있습니다. 예를 들면, 어떤 도큐먼트가 삭제될 때 함께 정리되어야 하는 의존성이 존재하는 경우가 있을 수 있겠죠. 따라서 미들웨어 호출을 보장하기 위해선 아래와 같이 먼저 도큐먼트를 불러온 뒤 삭제해야 합니다.

```javascript
const user = await User.findById(req.params.id).exec()
await user.remove()
res.send({ data: user })
```

아래 중 `remove()`의 경우 `remove` 훅을 발생시키지 않습니다. 그러나 `findByIdAndRemove()` 와 `findOneAndRemove()` 을 호출하면 `findOneAndRemove` 훅이 발생합니다.

```javascript
// 주의하세요
User.remove()
User.findByIdAndRemove()
User.findOneAndRemove()
```

# Clarify Middleware Behavior

## Intro

Mongoose 4.x 까지는 미들웨어의 호출 순서가 상식적이지 않았습니다.

```javascript
const mongoose = require('mongoose')
const Schema = mongoose.Schema

run().catch(error => console.error(error.stack))

async function run() {
    await mongoose.connect('mongodb://localhost:27017/test')

    const schema = new Schema({ name: String })
    schema.post('save', function (doc, next) {
    console.log('post save 1')
    next()
    });

    schema.post('save', function(doc) {
    console.log('post save 2')
    });

    const Person = mongoose.model('Person', schema)

    const p = new Person({ name: 'Taco' })
    await p.save()
    console.log('save promise resolved!')
}
```

```
post save 2
post save 1
```

예제를 보면 `post save 1` 을 출력하는 비동기 미들웨어가 먼저 등록되었기 때문에 `post save 2` 가 출력되기 전에 실행되었어야 하는데, 결과는 그렇지 않습니다. 약간 다른 예제를 보겠습니다. 

```javascript
const mongoose = require('mongoose')
const Schema = mongoose.Schema

run().catch(error => console.error(error.stack))

async function run() {
    await mongoose.connect('mongodb://localhost:27017/test')

    const schema = new Schema({ name: String })
    schema.post('findOne', function (doc, next) {
    console.log('post find 1')
    next()
    })

    schema.post('findOne', function (doc) {
    console.log('post find 2')
    })

    const Person = mongoose.model('Person', schema)

    await Person.findOne()
    console.log('find promise resolved!')
}
```

```
post find 1
post find 2
```

이전 예제와는 달리 예상했던 정상적인 출력 결과를 볼 수 있습니다. 왜냐하면 findOne 훅은 [kareem](https://www.npmjs.com/package/kareem)을 미들웨어 라이브러리로 사용하기 때문입니다. Mongoose 4.x에서 kareem을 사용하지 않는 `save`, `remove` 등
 [https://github.com/Automattic/mongoose/blob/4.x/lib/schema.js#L15-L27](https://github.com/Automattic/mongoose/blob/4.x/lib/schema.js#L15-L27) 을 제외한 훅에 매개변수가 1개인 미들웨어를 등록할 경우에는 (`.post('save', doc ⇒ {})`) 이벤트를 등록하는 것으로(`.on('save', doc ⇒ {})`)  [치환했으나](https://github.com/Automattic/mongoose/blob/4.x/lib/schema.js#L1179-L1183)(이는 pre 훅도 비슷합니다) 매개변수가 2개인(`(doc, next)`) 비동기 미들웨어의 경우는 별도로 처리했기 때문입니다.

이런 문제를 버전 5에서는 거의 모든 훅을 kareem으로 처리해서 해결하게 됩니다. 따라서 모든 미들웨어들은 항상 등록된 순서대로 호출된다고 보장할 수 있죠. 덕분에 모든 미들웨어에 `async/await`함수를 사용할 수 있게 되었습니다. 아래는 그 예제입니다.

```javascript
schema.pre('save', async function () {
    await something.async()
    console.log('saving..')
})

schema.post('save', async function (doc) {
    await Promise(resolve => setInterval(() => resolve(), 1000))
    console.log('saved!')
})
```

## 'this' in middleware

Mongoose는 미들웨어에 `this`를 바인딩하여 유용하게 사용할 수 있습니다. 다만 미들웨어 함수에서 `this`는 해당 훅의 종류에 따라 다릅니다. 아래 표에 정리해두었습니다.

| this | middleware |
|:----:|:----------:|
| document | `validate`, `save`, `remove`, `init(synchronous)` |
| query | `count`, `find`, `findOne`, `findOneAndRemove`, `findOneAndUpdate`, `update`, `updateOne`, `updateMany` |
| aggregation object | `aggregate` |
| model | `insertMany` |

# Type of Middlewares

미들웨어 함수의 매개변수의 개수, 미들웨어가 `pre`인지 `post`인지에 따라 Mongoose가 미들웨어 함수에 전달하는 인자의 값도 많이 달라집니다.

## pre

`pre` 미들웨어는 쿼리(hooked method)가 실행되기 전에 모두 실행되는 함수들입니다.

Mongoose는 `pre` 훅을 Serial과 Parallel, 두가지 타입으로 설명합니다.  

Serial 미들웨어는 우리가 일반적으로 사용하는 훅인데요, 하나의 미들웨어가 끝나야 다음 미들웨어가 실행됩니다. `Promise`를 반환하는 미들웨어의 경우, resolve가 되어야만 다음 미들웨어를 실행하죠.

```javascript
schema.pre('save', function (next) {
    // do stuff
    next()
})

schema.pre('save,', async function () {
    await somethingAsync()
    await coolStuff()
})
```

여기서 주의할 점은, `next()`는 반드시 모든 비동기 작업을 마친 후 실행해야 한다는 것입니다. 다음 미들웨어를 호출하는 함수이기 때문에, 모든 미들웨어가 실행된 후에는 쿼리가 실행되기 때문이죠. 물론 쿼리가 실행되기 전까지 끝나지 않아도 문제가 없는 작업이라면 상관없습니다.

Parallel 미들웨어는 각 미들웨어에서 `done()` 함수가 호출되기 전까지는 쿼리를 실행하지 않습니다. 오랜 시간이 걸리는 비동기 작업을 하는 미들웨어들이 많을 경우 매우 유용한 옵션입니다. Parallel 미들웨어를 사용하기 위해서는 반드시 함수의 2번째 인자로 true를 주어야 합니다.

```javascript
schema.pre('save', true, function(next, done) {
    next()
    setTimeout(done, 100)
})

schema.pre('save', true, function(next, done) {
    next()
    setTimeout(done, 200)
})
```

`pre` 훅에서 에러가 발생하면, 그 이후에 예정되어 있던 미들웨어 함수들과 쿼리는 실행되지 않습니다.

## post

`post` 훅은 모든 `pre`  훅의 미들웨어, 그리고 쿼리(hooked method)가 끝난 뒤에 발동합니다.

미들웨어 함수의 매개변수가 1개 혹은 2개인 경우, 첫 번째 매개변수는 해당 hooked method의 결과값을, 두 번째 매개변수는 다음 미들웨어를 호출하는 `next` 함수입니다. 매개변수가 2개인 경우 비동기 미들웨어로 인식하여 `next` 를 호출하기 전까진 다음 미들웨어가 실행되지 않습니다.

또한 `pre` 훅과 마찬가지로 `Promise`를 반환할 수도 있습니다.

```javascript
schema.post('save', function (doc, next) {
    console.log(doc.name + 'saved!')
    setTimeout(next, 1000)
})

schema.post('save', async function(doc) {
    await somethingAsync()
    await somethingCool()
})
```

미들웨어 함수의 매개변수가 3개라면 약간 다릅니다. 이 경우 Mongoose 에서는 Error Handling Middleware라고 특별하게 분류하는데, 첫번째 매개변수는 `error`, 그리고 두번째와 세번째는 각각 해당 hooked method의 결과값과 `next` 함수가 됩니다.  **이 미들웨어 함수는 에러가 발생했을 때만 실행됩니다.** 에러가 발생했을 때 로깅을 하거나 읽기 쉽게 출력하고 싶을 때 유용합니다.

```javascript
schema.post('save', function (err, doc, next) {
    if (error.name === 'MongoError && error.code === 11000) {
        next(new Error('There was a duplicate key error'))
    } else {
        next()
    }
})
```

유의할 점은, `next()`에 에러를 전달하지 않았어도, 발생한 에러는 사라지지 않습니다. 이미 던져진 에러는 가던대로 흘러가죠.

## init

`init` 훅은 조금 특별합니다. `init` 훅을 제외한 모든 훅들은 비동기 미들웨어를 지원합니다. 그러나 `init` 훅은 비동기 미들웨어를 허용하지 않습니다. 왜냐하면 [`init()` 함수](https://mongoosejs.com/docs/api.html#document_Document-init)가 동기이기 때문이죠.

`Document.prototype.init()` 함수가 `init` 훅을 발생시키는데, 해당 함수는 네이티브 Mongo 드라이버가 반환한 오브젝트(document)를 Mongoose의 document 인스턴스로 변환하는 역할을 합니다. 따라서 미들웨어의 호출 시점도 `init`이 호출되기 전(pre)와 후(post)로 구분됩니다. 아래에 Mongoose 문서에 있는 예제를 붙여넣었습니다.

```javascript
const schema = new Schema({ title: String, loadedAt: Date });

schema.pre('init', pojo => {
    assert.equal(pojo.constructor.name, 'Object'); // Plain object before init
});

const now = new Date();
schema.post('init', doc => {
    assert.ok(doc instanceof mongoose.Document); // Mongoose doc after init
    doc.loadedAt = now;
});

const Test = db.model('TestPostInitMiddleware', schema);

return Test.create({ title: 'Casino Royale' }).
    then(doc => Test.findById(doc)).
    then(doc => assert.equal(doc.loadedAt.valueOf(), now.valueOf()));
```

# Conclusion

중요한 것은 쿼리를 요청할 때는 항상 미들웨어와 Validator를 정상적으로 거치도록 적절한 옵션을 주어 사용해야한다는 것입니다. 또한 쿼리 요청은 비용이 크기 때문에 되도록이면 `findOneAndUpdate()`등 요청의 횟수를 줄이는 쿼리를 사용하거나 쿼리의 체이닝을 통한 최적화를 고려해야 합니다.

p.s - 원래 이 가이드의 시작은 [이 글](https://twm.me/correct-way-to-use-mongoose/)의 번역이었는데, Best Practice라고 하면서 잘못된 정보와 별로 좋지 않은 사용법만 알려주길래 초장만 번역하고 글의 거의 모든 부분을 새로 작성하였습니다.

