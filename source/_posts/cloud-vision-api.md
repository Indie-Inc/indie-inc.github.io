title: Node.jsとGoogle Cloud Vision API使って色々な画像認識試してみた。
date: 2015-12-24 18:24:41
tags: ["node", "google", "api"]
author: tejitak
---

Googleから機械学習＆デイープラーニングを駆使した画像認識APIであるCloud Vision APIなるものがLimited Previewでリリースされました。早速使ってみました。

## できること
Cloud Vision APIを使うと以下のことができます。

Feature Type  | Description
------------- | -------------
FACE_DETECTION  | 顔認識
LANDMARK_DETECTION  | ランドマークの認識
LOGO_DETECTION | 製品ロゴの認識
LABEL_DETECTION | 画像コンテンツの認識 (ラベリング)
TEXT_DETECTION | 画像内テキストの認識 (OCR)
SAFE_SEARCH_DETECTION | セーフサーチの判定

関連日本語記事
 * Cloud Vision APIの凄さを伝えるべくRasPi botとビデオを作った話  
 http://qiita.com/kazunori279/items/768c7fdf96cdf45a9d16
 * Cloud Vision API はエロ画像をどの程度フィルタできるのか見てみた  
 http://qiita.com/macoshita/items/64ed04d1b8f3883d5532

基本的にはREST APIでFeatureを指定して画像をPOSTします。そうすると、指定したFeatureに関する結果が返ってきます。

現時点のAlpha Previewでのusage limitation（API制限）は以下のようになっています。

Type of Limit	| Usage Limit
------------- | -------------
QPS	| 1
1リクエストあたりの画像サイズ(MB)	| 8 MB
1リクエストあたりの画像数 |	16
1日あたりのリクエスト数 (ラベリング以外)	| 10,000
1日あたりのリクエスト数 (ラベリング)	| 5,000

一回のAPI Requestに16枚の画像を含めれて、1日の制限の10,000回API Callすれば、1日で大体16万枚の画像を処理することができそうですね。

## Setup
事前にLimited Previewの使用申請を行ってください。

* Cloud Vision API Limited Preview  
https://cloud.google.com/vision/

"Sign Up for the Limited Preview"をクリックして申請。
承認されると以下のdocumentにアクセスできるようになる

https://cloud.google.com/vision/docs/


### Server Keyの取得
Google Cloud PlatformのページでCloud Vision APIを有効にして、API Key(サーバーキー)を取得します。

[Cloud PlatformCloud Vision API](https://console.cloud.google.com/apis/api/vision.googleapis.com/overview) を開き、"APIを有効にする"ボタンを押します

[Cloud PlatformのAPI Credential](https://console.cloud.google.com/apis/credentials) へ行って、認証情報タブから"新しい認証情報"というボタンをクリックして、APIキーを選択します。

{% asset_img 2015-12-24_19.06.12.png %}

今回はNode.jsでscriptを動かすので、サーバーキーを選択して、そのまま作成ボタンを押します。

生成されたキーは後ほどコードの中で使用します。


### Nodeモジュール (node-cloud-vision-api)

適当にディレクトリを作って、npm init。そして、cloud vision API用のnode clientのモジュールをinstallする。

* node-cloud-vision-api  
https://github.com/tejitak/node-cloud-vision-api

（↑良いnode clientがなかったので自作しました。フィードバックなどありましたら、ぜひお知らせください。）

```shell
npm install --save node-cloud-vision-api
```

### Sample Code
コードは以下のようになります。
```javascript
'use strict'
const vision = require('node-cloud-vision-api')

// init with auth
vision.init({auth: 'YOUR_API_KEY'})

// construct parameters
const req = new vision.Request({
  image: new vision.Image('/Users/tejitak/temp/test1.jpg'),
  features: [
    new vision.Feature('FACE_DETECTION', 4),
    new vision.Feature('LABEL_DETECTION', 10),
  ]
})

// send single request
vision.annotate(req).then((res) => {
  // handling response
  console.log(JSON.stringify(res.responses))
}, (e) => {
  console.log('Error: ', e)
})
```

一つのAPIリクエストで複数の画像を指定したい場合は、以下のようにannotateにrequestの配列を渡します。
また、もしWeb上にある画像を直接指定したい場合は以下の２つ目のrequestのようにurlを指定します。

```javascript
// 1st image of request is load from local
const req1 = new vision.Request({
  image: new vision.Image({
    path: '/Users/tejitak/temp/test1.jpg'
  }),
  features: [
    new vision.Feature('FACE_DETECTION', 4),
    new vision.Feature('LABEL_DETECTION', 10),
  ]
})

// 2nd image of request is load from Web
const req2 = new vision.Request({
  image: new vision.Image({
    url: 'https://scontent-nrt1-1.cdninstagram.com/hphotos-xap1/t51.2885-15/e35/12353236_1220803437936662_68557852_n.jpg'
  }),
  features: [
    new vision.Feature('FACE_DETECTION', 1),
    new vision.Feature('LABEL_DETECTION', 10),
  ]
})

// send multi requests by one API call
vision.annotate([req1, req2]).then((res) => {
  // handling response for each request
  console.log(JSON.stringify(res.responses))
}, (e) => {
  console.log('Error: ', e)
})
```

## 顔認識とラベリングを行うサンプル
簡単な例としてInstagram上の写真のURLを指定して、どのような結果が取得できるか見てみましょう。

ちなみに、ドキュメントによると顔認識をするためには、眉毛の間の位置などの測定が重要になるので、推奨画像サイズが	1600 x 1200 なんだそうです。一応 640 x 480 でもうまくは動作するけど、もっと精度を上げたかったら推奨サイズでインプットしてね、だそうです。

インスタグラムだと 640x640 が一般的なようですが、一応以下に結果を紹介するようにうまく動いているようです。

### サングラスをかけた女性の写真
まずはInstagram有名人のTaylor Swiftさんから試してみます。
![Taylor Swiftさん](https://scontent-nrt1-1.cdninstagram.com/hphotos-xap1/t51.2885-15/e35/12353236_1220803437936662_68557852_n.jpg)

きちんと顔認識されています。

```json
angerLikelihood: "VERY_UNLIKELY"
blurredLikelihood: "VERY_UNLIKELY"
boundingPoly: Object
detectionConfidence: 0.577574
fdBoundingPoly: Object
headwearLikelihood: "VERY_UNLIKELY"
joyLikelihood: "POSSIBLE"
landmarkingConfidence: 0.19900249
landmarks: Array[34]
panAngle: 0.27318951
rollAngle: 7.2499251
sorrowLikelihood: "VERY_UNLIKELY"
surpriseLikelihood: "VERY_UNLIKELY"
tiltAngle: 11.967749
underExposedLikelihood: "VERY_UNLIKELY
```

感情の認識も力を入れいているようで、xxxLikelihood という値が感情別の評価を表しています。今回の写真のケースでは joyLikelihood のみが POSSIBLE で、他は VERY_UNLIKELY なのでJoy(楽しい)という感情として認識されていますね。

さらにこのlandmarksがすごくて34エントリを展開すると、、
```json
0: Object
  position: Object
    x: 342.14697
    y: 77.257805
    z: 0.0010066124
  type: "LEFT_EYE"
1: Object
  position: Object
    x: 524.90961
    y: 100.77812
    z: 0.84962881
  type: "RIGHT_EYE"
2: Object
  position: Object
    x: 285.53159
    y: 30.352997
    z: 25.532389
  type: "LEFT_OF_LEFT_EYEBROW"
3: Object
  position: Object
    x: 395.71207
    y: 32.727783
    z: -25.809153
  type: "RIGHT_OF_LEFT_EYEBROW"
...
```
こんな感じで、目の場所とか眉毛の場所がpositionの情報付きで返ってきます。その他口とか、顔だけでなく、顔の各パーツが認識されています。

さらにラベリングの方の結果を見てみましょう。
```json
0: Object
  description: "eyewear"
  mid: "/m/0j272k5"
  score: 0.96393067
1: Object
  description: "nose"
  mid: "/m/0k0pj"
  score: 0.86950552
2: Object
  description: "facial expression"
  mid: "/m/01k74n"
  score: 0.86561471
3: Object
  description: "glasses"
  mid: "/m/0jyfg"
  score: 0.76605028
4: Object
  description: "vision care"
  mid: "/m/0h8jxfl"
  score: 0.72531086
5: Object
  description: "smile"
  mid: "/m/019nj4"
  score: 0.57442826
6: Object
  description: "canidae"
  mid: "/m/01z5f"
  score: 0.53674406
```
ちゃんと eyewear とか glasses とかサングラスであることを認識していますね。
そして smile と笑顔であるということも取れています。

### 人の集合
そんな感じで次はInstagramによりフィルター加工された写真にトライ
![加工処理済み写真](https://igcdn-photos-g-a.akamaihd.net/hphotos-ak-xtp1/t51.2885-15/e15/11265033_1590354827848422_135051294_n.jpg)

```
faceAnnotations: Array[2]
  0: Object
  1: Object
labelAnnotations: Array[4]
0: Object
  description: "man"
  mid: "/m/04yx4"
  score: 0.89972556
1: Object
  description: "wedding"
  mid: "/m/081hv"
  score: 0.72876191
2: Object
  description: "bride"
  mid: "/m/0ltfs"
  score: 0.69205159
3: Object
  description: "wedding dress"
  mid: "/m/02c66t"
  score: 0.55351728
```
ちゃんと人が２人写っていて、weddingとかwedding dressも認識されている！！

### 食べ物
次はカレー
![カレー](https://igcdn-photos-b-a.akamaihd.net/hphotos-ak-xtf1/t51.2885-15/e35/11809865_911873062195873_1036356262_n.jpg)

FACE_DETECTIONの結果はなし。ラベリング結果は以下のとおり。
```
0: Object
  description: "food"
  mid: "/m/02wbm"
  score: 0.992246
1: Object
  description: "dish"
  mid: "/m/02q08p0"
  score: 0.98311311
2: Object
  description: "gumbo"
  mid: "/m/01qf88"
  score: 0.83931148
3: Object
  description: "meal"
  mid: "/m/0krfg"
  score: 0.79317588
4: Object
  description: "vegetable"
  mid: "/m/0f4s2w"
  score: 0.61712343
5: Object
  description: "curry"
  mid: "/m/0hr1s_w"
  score: 0.60232091
6: Object
  description: "gulai"
  mid: "/m/01x_c"
  score: 0.59999269
```
Foodとして認識されていますね、そしてびっくりしたのが、curry（カレー）であることも認識されている！

### 本
本は認識できるかな？ついでにOCR (TEXT_DETECTION) もかけてみる
![本](https://igcdn-photos-c-a.akamaihd.net/hphotos-ak-xtf1/t51.2885-15/e35/11821172_1659394597606938_721829481_n.jpg)

ラベリングの結果
```json
0: Object
  description: "poster"
  mid: "/m/01n5jq"
  score: 0.98364943
1: Object
  description: "advertising"
  mid: "/m/011s0"
  score: 0.82112622
2: Object
  description: "party"
  mid: "/m/05_4_"
  score: 0.7021277
3: Object
  description: "graphic design"
  mid: "/m/03c31"
  score: 0.69916856
4: Object
  description: "magazine"
  mid: "/m/058sv"
  score: 0.68585771
5: Object
  description: "tabloid"
  mid: "/m/07mbq"
  score: 0.6555953
6: Object
  description: "newspaper"
  mid: "/m/05jnl"
  score: 0.655531
7: Object
  description: "flyer"
  mid: "/m/0218rg"
  score: 0.5669809
```
広告的なもの？として認識されているようですが、ちゃんと magazine も入っていますね！
OCRの結果はこんな感じでした。

```shell
テストも ビルドも errくN 7-ーイジブロィリPRESS デプロイも ソラウドで加速! モバイル開 Circle CLDeployGate. Crashlytics... 1日100億メッセージをさばくサービスの裏側 開発ノウハウ 大公開 サービスの急拡大に耐えるスケール戦術 データベース設計 MOTT loT時代のプロトコル 次世代言語ElixirでWeb開発 HHVMによるPHP実行速度の高速化 ダリング速度の改善ノウハウ サ実践]
```

完璧ではないですが、まぁまぁ取れてますね！

### まとめ
いやー結構精度の高さにはびっくりしました。試しに使ってみましたが、思ったより断然お手軽に使えます。いろいろな応用ができそうで何か可能性を感じますね。

今回紹介したnode clientである [node-cloud-vision-api](https://github.com/tejitak/node-cloud-vision-api) に関しても不具合・ご要望などありましたら、ぜひご連絡ください。
