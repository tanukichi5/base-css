# ベースCSS

個人的に普段使用しているベースのCSSです。



## とりあえずコンパイル環境を構築

ここでの環境構築はnpm-scriptsでscssをコンパイルするだけの最小の環境です。

Sassコンパイルするために必要なパッケージをインストールします。

```
npm i
```

`.scss`ファイルを編集後に以下のコマンドでコンパイルされます。

```
npm run sass
```





### node-sass(LibSass)環境に持っていく場合

- DartSassで書かれているのでnode-sass(LibSass)の環境で動かす場合は各ファイルの`@use`の行の記述を消して`@import`で各ファイルを読み込めば多分動くと思います。
- 一部除算にsass:mathモジュールのdiv関数を使用しているので書き換える必要がある

```scss
math.div($number, $number * 0 + 1)
↓修正
$number / ($number * 0 + 1)
```



## mixin

いくつかmixinを組み込んでいるので、その使い方を紹介します。

mixinは`src/scss/foundation/mixin`に格納しています。

### 準備

はじめにmixinを使用したいファイルで`_global.scss`を読み込む必要があります。

```scss
@use 'path/to/global' as *;
```

読み込むことでmixinの他に設定済みの変数やfunctionも使用可能になります



## ブレイクポイントの指定 min-screen, max-screen

メディアクエリ用のmixin（ブレイクポイント）
**min-screen**を使用することを推奨（モバイルファーストで書くため）

### ブレイクポイントは以下のように変数で定義しています

**_breakpoints.scss**

```scss
$breakpoints: (
  'sm': 640px,
  'md': 768px,
  'lg': 1024px,
  'xl': 1280px
  ) !default;
```

ブレイクポイント変数の呼び出し方
screen関数を使用します（`src/sass/foundation/functions/_screen.scss`）
第一引数に`$breakpoints`のkeyを指定

_screen.scss

```scss
@function screen($size) {
  @return map-get($breakpoints, $size);
}
```

```scss
screen("md") //768px
```

### usage

第一引数にscreen関数で**ブレイクポイント値**を指定

```scss
@include min-screen([ブレイクポイント]) {
  //ブレイクポイント以上の画面サイズにスタイルを適用
}

@include max-screen([ブレイクポイント]) {
  //ブレイクポイント以下の画面サイズにスタイルを適用
}
```

```scss
//コンパイル前
.section {
  width: 100%;
  @include min-screen(screen("md")) {
    width: 700px;
  }
  @include max-screen(screen("sm")) {
    width: 500px;
  }
}
```

### output

```css
//コンパイル後
.section {
  width: 100%;
}

@media only screen and (min-width: 768px) {
  .section {
    width: 700px;
  }
}

@media only screen and (max-width: 640px) {
  .section {
    width: 500px;
  }
}
```



## font-size : pxをremに変換して出力

入力されたpxをremに変換して出力するmixinです。

`remove-unit`関数は単位を外す処理をしています。
`$base`はbodyやhtml要素で指定しているフォントサイズを設定。
何もしていない場合はブラウザの基本のフォントサイズである16pxでOK。

※bodyやhtml要素でfont-sizeを62.5%（10px）と指定している場合は`$base: 10px`というように書き換えてください。

**_font-size.scss**

```scss
@use "sass:math";
@use '../functions/_remove-unit' as *;

@mixin font-size($size, $base: 16px) {
	$rem: math.div(remove-unit($size), remove-unit($base)) * 1rem;
	font-size:$rem;
}
```

### usage

- 第一引数に**px**

```scss
//使い方
@include font-size([pxを指定]);
```

```scss
//コンパイル前
.text {
  @include font-size(20px);
}
```

### output

```css
.text {
  font-size: 1.25rem;
}
```



## state : セレクターの状態変化の指定

離れた位置にあるセレクターの状態を変化させるmixinです
主に`:hover`や`.-active`,`.-current`などのステートクラスが付与された状態に変化させることができます。

**_state.scss**

```scss
@use '../functions/_is-inside' as *;
@use '../functions/_parent' as *;

@mixin state ($target, $state) {
  //ネストの外か内を判定
  @if is-inside($target) {
    @at-root #{selector-replace(&, $target, $target + $state)} {
      @content
    }
  } @else {
    @if $target == parent(#{&})  {
      @at-root #{selector-replace(&, parent(#{&}), $target + $state)} {
        @content
      }
    } @else {
      @at-root #{selector-replace(&, parent(#{&}), $target + $state + " " + parent(#{&}))} {
        @content
      }
    }
  }
}
```

### usage

- 第一引数に**対象のセレクター**
- 第二引数に**対象のセレクターに付与したい状態**（`:hover`,`.-active`など）

```scss
//使い方
@include state("[対象のセレクター]", "[対象のセレクターに付与したい状態]") {
  //ここにスタイルを記述
}
```

```scss
//コンパイル前
.card {
  width: 200px;
  height: auto;
}

.card__text {
  @include state(".card", ":hover") {
    opacity: 0.8;
  }
  @include state(".card", ".-active") {
    color: #f00;
  }
}
```

### output

```css
//コンパイル後
.card {
  width: 200px;
  height: auto;
}

.card:hover .card__text {
  opacity: 0.8;
}

.card.-active .card__text {
  color: #f00;
}
```



## col : 均等に横並びにするレイアウト

**※IEがなくなればこのmixinは不要です。**

flexBoxで均等割にしたい場合に使用します
均等横並びは`justify-content: space-between;`でも可能ですが、数が少ないと左右に寄る欠点があります。その欠点をを解消したものがこのmixinです。

また:nth-childなどを駆使してmarginを消すなどの処理も必要ありません。

**_col.scss**

```scss
@use "sass:math";

@mixin col($col: 3, $marginRight: 0, $marginTop: 0, $parent: "none"){
	width: calc((100% - #{$marginRight * $col +px} * (#{$col} - #{$col - 1})) / #{$col} - 0.1px);
	margin-right: #{math.div($marginRight, 2) +px};
	margin-left: #{math.div($marginRight, 2) +px};
	margin-bottom: #{$marginTop+px};
	@if $parent != "none"{
		@at-root #{$parent} {
			margin-right: -#{math.div($marginRight, 2) +px};
			margin-left: -#{math.div($marginRight, 2) +px};
			margin-bottom: -#{$marginTop+px};
		}
	}
}
```

### usage

親要素に`display: flex;`、`flex-wrap: wrap;`を設定

- 第一引数に**分割数**
- 第二引数に**左右の余白**
- 第三引数に**上余白**
- 第四引数に**display: flex;が指定されている親要素**

```scss
//使い方
@include col([分割数], [左右の余白], [上余白], "[display: flex;が指定されている親要素]");
```

```html
<div class="card">
  <div class="card__item"></div>
  <div class="card__item"></div>
  <div class="card__item"></div>
  <div class="card__item"></div>
</div>
```

```scss
//コンパイル前
.card {
  display: flex;
  flex-wrap: wrap;
}
.card__item {
  @include col(3, 20, 10, ".card");
}
```

### output

```css
//コンパイル後
.card {
  display: -webkit-box;
  display: -ms-flexbox;
  display: flex;
  -ms-flex-wrap: wrap;
  flex-wrap: wrap;
}

.card__item {
  width: calc((100% - 60px * (3 - 2)) / 3 - 0.1px);
  margin-right: 10px;
  margin-left: 10px;
  margin-bottom: 10px;
}

.card {
  margin-right: -10px;
  margin-left: -10px;
  margin-bottom: -10px;
}
```

[**codepenでプレビュー**](https://codepen.io/tanukichi5/pen/YzZgZEx)

##### そもそもIEがなければ...

IEを無視できる案件であればグリッドを使用することでキレイに書くことができます

```css
display: grid;
grid-template-columns: 1fr 1fr;
gap: 20px;
```

IEでも使用できなくはないですが、autoprefixerを使用しても`grid-template`の指定が必須だったり面倒な記述が増えます。



## object-fit

**※IEがなくなればこのmixinは不要です。**

IEで非対応CSSプロパティである`object-fit`を有効にするもの
jsの`object-fit-images`の読み込みが必須。

[object-fit-imagesのgithub](https://github.com/fregante/object-fit-images)

**_object-fit.scss**

```scss
@mixin object-fit($type, $position: center center) {
    object-fit: $type;
    object-position: $position;
    font-family: 'object-fit: #{$type}; object-position: #{$position}', sans-serif;
}
```

- 第一引数に`cover`などobject-fitの値を指定
- 第二引数に`top center`や`50% 50%`など位置を指定

### usage

```scss
//使い方
@include object-fit([coverやcontainなどプロパティ], [top centerなど位置を指定]);
```

```scss
//コンパイル前
.iamge1 {
  @include object-fit(cover)
}

.iamge2 {
  @include object-fit(contain, top 50%)
}
```

### output

```css
//コンパイル後
.iamge1 {
  -o-object-fit: cover;
  object-fit: cover;
  -o-object-position: center center;
  object-position: center center;
  font-family: "object-fit: cover; object-position: center center", sans-serif;
}

.iamge2 {
  -o-object-fit: contain;
  object-fit: contain;
  -o-object-position: top 50%;
  object-position: top 50%;
  font-family: "object-fit: contain; object-position: top 50%", sans-serif;
}
```



## hover

このmixinはPCやスマホ、タブレットなどホバー効果を有効、無効を自動で切り替えます。

- PCではホバー効果を有効
- スマホ、タブレットではホバー効果を無効
- キーボードでのフォーカスにホバー効果を有効

基本的には**hoverメディアクエリ**を使用して判定します。
**スマホのホバー効果を無効にする手間がなくなります**

### 事前準備

#### [focus-visible](https://github.com/WICG/focus-visible)のポリフィルをインストール

`:focus-visible`は、タブ移動でフォーカスの当たった要素の`outline`を表示させる疑似クラス
safariでは使えないため

[focus-visible](https://github.com/WICG/focus-visible)のポリフィルのGitHub

```
npm install --save focus-visible
```

```javascript
import focusVisible from "focus-visible";
```

```css
//下記cssを追記
.js-focus-visible :focus:not(.focus-visible) {
  outline: 0;
}
```

#### IEはユーザーエージェント判別で事前にbodyもしくはhtmlにclassを付与しておく

hoverメディアクエリはIEでは使用できないため。
このmixinでは`.ua-ie`というclassが付与されている前提になっています。

**_hover.scss**

```scss
@use '../functions/_is-inside' as *;

//スマホではホバー効果を適用しないhover mixin
//フォーカス時にもホバー効果を適用
@mixin hover ($target: null, $mobile: false) {
  @if $target == null {
    @media (hover: hover) {
      &:hover {
        @content
      }
    }
    //ie用(事前にjsでユーザーエージェントでbodyにclass付与)
    @at-root .ua-ie &:hover {
      @content;
    }
    //キーボードでフォーカスしたとき(主にtabキー移動)
    //safariは「環境設定」の「詳細」タブで操作中の項目を強調表示にチェックが必要
    &.focus-visible:focus {
      @at-root .js-focus-visible & {
        @content
      }
    }
    //スマホでホバー有効設定の場合
    @if $mobile == true {
      @media (hover: none) {
        &:active {
          @content
        }
      }
    }
  } @else {
    //ネストの外か内を判定
    @if is-inside($target) {
      //ネストの内側
      @media (hover: hover) {
        @at-root #{selector-replace(&, $target, $target + ":hover")} {
          @content
        }
      }
      //ie用(事前にjsでユーザーエージェントでbodyにclass付与)
      @at-root #{selector-replace(&, $target, $target + ":hover")} {
        @at-root .ua-ie & {
          @content;
        }
      }
      //キーボードでフォーカスしたとき(主にtabキー移動)
      @at-root #{selector-replace(&, $target, $target + ".focus-visible:focus")} {
        @at-root .js-focus-visible & {
          @content
        }
      }
      //スマホでホバー有効設定の場合
      @if $mobile == true {
        @media (hover: none) {
          @at-root #{selector-replace(&, $target, $target + ":active")} {
            @content
          }
        }
      }
    } @else {
      //ネストの外側
      @media (hover: hover) {
        @at-root #{$target + ":hover" + " " + &} {
          @content
        }
      }
      //ie用(事前にjsでユーザーエージェントでbodyにclass付与)
      @at-root #{$target + ":hover" + " " + &} {
        @at-root .ua-ie & {
          @content
        }
      }
      //キーボードでフォーカスしたとき(主にtabキー移動)
      @at-root #{$target + ".focus-visible:focus" + " " + &} {
        @at-root .js-focus-visible & {
          @content
        }
      }
      //スマホでホバー有効設定の場合
      @if $mobile == true {
        @media (hover: none) {
          @at-root #{$target + ":active" + " " + &} {
            @content
          }
        }
      }
    }
  }
}
```

※Androidだとホバーを無効にしても`:focus`が反応してしまうので、スマホのホバーは`:active`で対応しています。

### usage

- 第一引数に**ホバーする要素**を指定（指定しない場合は直前のネストになります）
- 第二引数は**スマホ、タブレットでホバー効果を有効にするか**の指定（指定しない場合は無効）

```scss
//使い方
@include hover([ホバーする要素], [スマホ、タブレットでホバー効果の有無(trueもしくはfalse)]) {
  //ホバー時のスタイル
}
```

```scss
//コンパイル前
.card {
  @include hover() {
    background: #ddd;
  }
}

.card__text {
  @include hover(".card") {
    color: #f00;
  }
}

.card__image {
  @include hover(".card", true) {
    transform: scale(1.1);
  }
}
```

### output

```css
//コンパイル後
@media (hover: hover) {
  .card:hover {
    background: #ddd;
  }
}

.ua-ie .card:hover {
  background: #ddd;
}

.js-focus-visible .card.focus-visible:focus {
  background: #ddd;
}

@media (hover: hover) {
  .card:hover .card__text {
    color: #f00;
  }
}

.ua-ie .card:hover .card__text {
  color: #f00;
}

.js-focus-visible .card.focus-visible:focus .card__text {
  color: #f00;
}

@media (hover: hover) {
  .card:hover .card__image {
    -webkit-transform: scale(1.1);
    transform: scale(1.1);
  }
}

.ua-ie .card:hover .card__image {
  -webkit-transform: scale(1.1);
  transform: scale(1.1);
}

.js-focus-visible .card.focus-visible:focus .card__image {
  -webkit-transform: scale(1.1);
  transform: scale(1.1);
}

@media (hover: none) {
  .card:active .card__image {
    -webkit-transform: scale(1.1);
    transform: scale(1.1);
  }
}
```



## aspect-ratio : 画像などの比率維持

画像などのレスポンシブ対応でpaddingの上下%は親の幅が基準になる仕様を利用する方法をmixin化したものです。

**_aspect-ratio.scss**

```scss
@mixin aspect-ratio($width, $height, $first: true) {
  position: relative;

  &::before {
    content: '';
    display: block;
    padding-top: ($height / $width) * 100%;
  }

  @if $first == true {
    & > :first-child {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
    }
  }
}
```

### usage

- 第一引数に**幅**
- 第二引数に**高さ**
- 第三引数は**false**で子要素に`position: absolute;`などをデフォルトで当てない（任意）

```html
<div class="image">
    <img src="/image.jpg" alt="画像">
</div>
```

```scss
.image {
  @include aspect-ratio(1920, 1080);
}
```

### output

```css
.image::before {
  content: '';
  display: block;
  padding-top: 56.25%;
}

.image > :first-child {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}
```