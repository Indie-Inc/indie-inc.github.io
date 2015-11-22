title: Mongo DB Mongooseで非同期Validation
date: 2015-11-22 16:15:30
tags: mongoDB
---

こんばんはtejitakです。今日からIndie Incのエンジニアブログを書き始めます！w

MongoDBで定義したスキーマのデータを更新するときにvalidationを行うには以下のように行います。

## 通常のvalidationを行う例

``` js
// schemaの定義と同時にvalidateを定義する場合
var mongoose = require('mongoose');
var Schema = mongoose.Schema

var usernameLengthValidator = (value) => {
  // 64文字以上はerror
  return value.length < 64
}

var User = new Schema({
  "username": { type: String, default: '', validator: [{validator: usernameLengthValidator ,
      message: '{VALUE} should be less than 64 characters!'}]}
})

// schemaの定義の後にvalidateを定義する場合
User.path('username').validate(usernameValidator, '{VALUE} should be less than 64 characters!')

```

通常は上記のような方法でvalidateすれば良いのですが、例えばtwitterのuserIdのようなものを想定してみてください。

Userエントリをデータベースに更新する前に、重複IDが存在するかをDBの中から検索する必要があります。

mongooseのfindのAPIは非同期のものしか提供されていないため、validationも非同期に行う必要があります。

非同期validationは以下のコードのようにvalidateの関数の引数にcallbackを足して、処理の終了時にそれを呼んであげれば実現できます。

## 非同期にvalidationを行う例
``` js

// callbackを引数に定義し、非同期処理が完了したタイミングでsuccessならtrue, failならfalseをcallbackに渡して実行する
var asyncValidator = (value, callback) => {
  // Userデータの中に同じusernameが存在するかどうかをcount queryで判別する
  return mongoose.model('User').where({'username': value}).count((err, count) => {
    callback(count === 0, 'The username is already token!')
  })
}

var User = new Schema({
  "username": { type: String, default: '', validator: [
    {validator: usernameLengthValidator, message: '{VALUE} should be less than 64 characters!'},
    {validator: asyncValidator , message: ''}
  ]}
})
```

こんな感じで実装できました。

後で探してみると [mongoose-unique-validator](https://www.npmjs.com/package/mongoose-unique-validator) というツールもありました。このライブラリはmongooseのpluginを使って、上記と同様の countクエリーを発行しているようです。

以上mongoDBに関するTips初投稿でした。
こんな感じで些細なTipsなどを共有していこうかなと思っています。

よろしくお願いします！
