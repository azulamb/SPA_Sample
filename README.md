# Re:ゼロから始めるSPA

何がRe:なのか知りませんが、ゼロからSPAページを作りたいと思います。

## そもそもSPAって？

SPAは `Single Page Application` の略で、Webページの一つの形です。

通常Webページはブラウザがサーバーにリクエストを投げると結果をHTML形式のデータで返してきます。（サーバー側のHTMLのレンダリング）

この受け取ったデータをブラウザが画面にレンダリングすることで、ページを表示しています。

このように二段階のレンダリングを経て表示されるのですが、これが同じようなページであればサーバーの処理も通信量も馬鹿になりません。

そこで、初めのレンダリングが全て終わった後全てのレンダリングの更新をJSにやらせて、サーバーとはその差分データだけやり取りさせることで、サーバーの処理も通信量も抑えることが可能です。

レンダリング後のやり取りは通常のネイティブアプリケーションと同じような挙動を取っているため、SPAはWebページと言うよりアプリケーションとして扱われているようです。

### SPAの実現のために必要なこと

ではこのSPAを実現するにはどんな機能が必要でしょう？

* 最初のレンダリングの後は全てJSでレンダリングの更新を行う
* リンクをクリックしてもページ遷移のリクエストをサーバーに送らず、内々で処理する
* ブラウザの戻るボタンを押しても内々で処理する

ここら辺ができればSPAと名乗れそうです。

### SPAを作ってみる

全て手書きでSPAを実現しましょう。

とりあえず以下のようなSPAを作ります。

* URL末尾に `.md` をつけたMarkdownファイルを読み込んでページに表示する。
   * 今回はソースだけ表示する。
   * `/` で終わる場合は `index` というファイル名として扱う。

## SPA作成

今回はHTMLファイル1つで完結するように作ります。

段階を踏んで機能を追加しながらSPAを作ります。

* `index.md` の内容を取ってきて表示するだけのページ
* URLに応じて読み込むファイルを変えるページ

### index.md の内容を取ってきて表示するだけのページ

初めの目標は `index.md` を取得して表示することとします。

今回は以下のような構造で作ります。

* `http://USERNAME.github.io/SPA_Sample/NUM/` をベースのアドレスとする。
  * GitHub Pagesでサクッと動かすことを目標とする。
  * NUMはサンプルの番号で、初めは 0 で機能を追加するたびに数値を増やしていく。
  * Markdownのファイルもサンプルごとに同じフォルダにまとめておくことにする。

ではまずページのレンダリングを行います。
今回は必ず `/SPA_Sample/0/index.md` を読み込んで表示することにします。

#### /docs/0/index.html

```html
<!DOCTYPE html>
<html lang="ja">
<head>
	<meta charset="utf-8">
	<title>SPA sample 0</title>
	<script>
// コンテンツを取得します。
function Fetch( baseurl, path ) {
	// pathの簡易チェックを行います。
	if ( path === '/' ) { path += 'index'; }
	// pathに .md を追加します。
	path += '.md';
	// fetch()を使いやすいようにラップします。
	return fetch( baseurl + path ).then( ( result ) => {
		// fetch()が失敗するのはネットワークトラブルなどであり、404エラーなどでは失敗扱いになりません。
		// そこで result.ok で結果がエラーでない場合は結果テキストを返し、そうでない場合は失敗扱いにします。
		if ( result.ok ) { return result.text(); }
		throw result;
	} );
}

// コンテンツのレンダリングを行います。
function Render( contents, data ) {
	contents.textContent = data;
}

// 初期化部分。
function Init() {
	// コンテンツを描画するHTML要素を取得します。
	const contents = document.getElementById( 'contents' );

	// 今回の特殊事情の関係で、ベースのURLを作成します。
	// http://USERNAME.github.io/SPA_Sample/NUM/XXXXX から /SPA_Sample/NUM/ だけ抽出します。
	// location.pathname は /SPA_Sample/NUM/XXXXX の部分を取得可能です。
	const baseurl = location.pathname.replace( /^(\/[^\/]+\/[^\/]+).*$/, '$1' );

	// 今度はURLからSPAにとってのパスを取得します。
	// /SPA_Sample/NUM/XXXXX の /XXXXX の部分ですが、この中には/が入っている可能性もあります。
	// 今回は / 固定にします。必要なファイルは index.md になります。
	const path = '/';

	// コンテンツを取得します。
	Fetch( baseurl, path ).then( ( md ) => {
		// 取得したコンテンツをレンダリングする。
		Render( contents, md );
	} );
}

// document.getElementById() を使っても大丈夫になったら初期化関数を実行する。
document.addEventListener( 'DOMContentLoaded', Init );
	</script>
</head>
<body>
<pre id="contents"></pre>
</body>
</html>
```

今回まず無視しても良いところは `baseurl` の作成部分です。
GitHub Pagesで使うことを前提としているので、少し特殊な作りになってしまいます。
この作業は普通のSPAを作る時には必要ないことがほとんどかと思います。

次に重要なのは `fetch()` についてです。
普通に使いたいところなのですが、404エラーなども正常に結果を受け取った判定になる特徴があります。
（エラーになるのはネットワークエラーとか。）

そこで404などにも対応すべく、ちょっとラップ処理をしています。

#### /docs/0/index.md

こちらはコンテンツです。

```md
# index.md

test
```

#### 動作サンプル

https://hirokimiyaoka.github.io/SPA_Sample/0/

とりあえずMarkdownのソースを表示できているのが確認できます。

SPAっぽさは何一つ感じません。

### URLに応じて読み込むファイルを変えるページ

次にURLの内容に応じて読み込むファイルを変更するページを作ります。

具体的には以下のような感じです。

* `http://USERNAME.github.io/SPA_Sample/1/`
  * `/docs/1/index.md`
* `http://USERNAME.github.io/SPA_Sample/1/a.md`
  * `/docs/1/a.md`
* `http://USERNAME.github.io/SPA_Sample/1/dir/`
  * `/docs/1/dir/index.md`
* `http://USERNAME.github.io/SPA_Sample/1/dir/b`
  * `/docs/1/dir/b.md`
* `http://USERNAME.github.io/SPA_Sample/1/notfound`
  * このファイルは用意せず、エラーページとして見る。

#### Jekyllの無効化

ここら辺からGitHub Pagesで可動するJekyllが邪魔になる場合があるので無効化します。

`/docs/.nojekyll` という空ファイルを設置しておいてください。

#### 前のファイルのコピー

`/docs/0` をフォルダごとコピーして、フォルダ名を `1` に変更してください。

#### 必要コンテンツの追加

今のうちに必要なMarkdownファイルを用意しておきます。

##### /docs/1/a.md

```md
# a.md

test
```

##### /docs/1/dir/index.md

```md
# dir/index.md

test

##### /docs/1/dir/b.md

```md
# dir/b.md

test
```

#### ソースの改変

ではコピーした `/docs/1/index.html` のJS部分だけ修正していきます。

今回の目標はパスに応じで今まで固定だった `/index.md` を読み込む部分を変化させることです。

```js
// コンテンツを取得します。
function Fetch( baseurl, path ) {
	// pathの簡易チェックを行います。
	if ( !path ) { path = '/'; }
	// pathの末尾が / の場合は index を追加します。
	if ( path.match( /\/$/ ) ) { path += 'index'; }
	// pathに .md を追加します。
	path += '.md';
	// fetch()を使いやすいようにラップします。
	return fetch( baseurl + path ).then( ( result ) => {
		// fetch()が失敗するのはネットワークトラブルなどであり、404エラーなどでは失敗扱いになりません。
		// そこで result.ok で結果がエラーでない場合は結果テキストを返し、そうでない場合は失敗扱いにします。
		if ( result.ok ) { return result.text(); }
		throw result;
	} );
}
```

`Fetch()` は `path` の扱いをもう少ししっかりしています。
なにもない場合は `/` に書き換え、末尾が `/` の場合は `index` を追加し、最後に末尾に `.md` を追加します。

```js
function Init() {
	// コンテンツを描画するHTML要素を取得します。
	const contents = document.getElementById( 'contents' );

	// 今回の特殊事情の関係で、ベースのURLを作成します。
	// http://USERNAME.github.io/SPA_Sample/NUM/XXXXX から /SPA_Sample/NUM/ だけ抽出します。
	// location.pathname は /SPA_Sample/NUM/XXXXX の部分を取得可能です。
	const baseurl = location.pathname.replace( /^(\/[^\/]+\/[^\/]+).*$/, '$1' );

	// 今度はURLからSPAにとってのパスを取得します。
	// 本来なら location.pathname の中身をそのまま使うので、以下のように文字列にします。
	// const path = location.pathname + ''; // 文字列にしないとうっかり path を書き換えた時悲惨なことになるので注意。
	// /SPA_Sample/NUM/XXXXX の /XXXXX の部分ですが、この中には/が入っている可能性もあります。
	const path = location.pathname.replace( /^\/[^\/]+\/[^\/]+(.*)$/, '$1' );

	// コンテンツを取得します。
	Fetch( baseurl, path ).then( ( md ) => {
		// 取得したコンテンツをレンダリングする。
		Render( contents, md );
	} ).catch( ( error ) => {
		// 何かしらのエラーが発生しました。
		// この場合は単純に # Error というコンテンツを表示します。
		Render( contents, '# Error' );
	} );
}
```

`Init()` では `path` の取得方法とエラー時のコンテンツの表示周りを調整しています。

ここに書いてあるとおり、本来ならばSPAでは `const path = location.pathname + '';` で良いのですが、今回はURLの都合上このように少し複雑な正規表現を用いています。
中身はあまり理解しなくともよいです。

とにかく `/SPA_Sample/NUM/XXXXX` における `/XXXXX` の部分を取得していることだけ理解してください。
