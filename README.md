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
* リンクを踏んでもページを読み込むのではなくコンテンツのみ読み込んで描画する。

## SPA作成

今回はHTMLファイル1つで完結するように作ります。

段階を踏んで機能を追加しながらSPAを作ります。

* `index.md` の内容を取ってきて表示するだけのページ
* URLに応じて読み込むファイルを変えるページ
* コンテンツに応じて適切なURLを表示するページ
* リンクを制御するページ

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
* `http://USERNAME.github.io/SPA_Sample/1/a`
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

#### 404対策

例えば `http://USERNAME.github.io/SPA_Sample/1/a` のページに遷移しようとした時、`/docs/1/a` というファイルがなければGitHub Pagesでは404ページが表示されてしまいます。

このようにファイルがない場合でも `/docs/1/index.html` をブラウザに返してもらわなければSPAが実現できません。
（逆に言えばこのようなリクエストに対して正しい `index.html` を返す仕組みがサーバーにないとSPAが作れないので注意。）

このような場合、GitHubでは以下のようなファイルを用意します。

##### /docs/404.html

```html
<!DOCTYPE html>
<html lang="ja"><head><title>Redirect</title><script>sessionStorage.redirect=location.href;</script><meta http-equiv="refresh" content="0; url=/XXXX" /></head></html>
```

このファイルを用意した場合、`http://USERNAME.github.io/XXXX/a` というURLは `http://USERNAME.github.io/XXXX/` にリダイレクトされます。

そしてURL情報は `sessionStorage` に保存されます。

後はSPA内で `sessionStorage` 内に前のURL情報が入っていればその情報を使い、そうでない場合は `location.pathname` を使うことで、読み込むべきファイルパスを取得することが出来ます。

今回は特殊な仕様なので、以下のようなリダイレクトを行う404.htmlを作成してください。

```html
<!DOCTYPE html>
<html lang="ja"><head><title>Redirect</title><script>sessionStorage.redirect=location.pathname;location.href=location.pathname.replace( /^(\/[^\/]+\/[^\/]+).*$/, '$1' );</script></head></html>
```

差分で特に重要なのは、`sessionStorage.redirect=location.pathname;` とリダイレクトです。
今回は `location.pathname` のみ保存していますが、これだとGETパラメーターが消失してしまいます。
本来であればちゃんと `location.href` でURLのパスとGETパラメーター等を全て渡すべきです。

この処理は今回だけの例外なので、基本的には一つ上に用意した方のソースをベースにしてください。

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
// 初期化部分。
function Init() {
	// コンテンツを描画するHTML要素を取得します。
	const contents = document.getElementById( 'contents' );

	// 今回必要なパスは sessionStorage.redirect もしくは location.pathname の中に入っています。
	// 今回はここでどちらか値がある方のデータを取得します。
	// ちなみに、どちらも単純に代入しただけだと文字列としてコピーされず、値を変更した時に悲惨なことになるので必ず文字列にしておいた方がよいです。
	const pathname = ( sessionStorage.redirect || location.pathname ) + '';
	// sessionStorageのデータは消しておきます。（消しておかないとリダイレクト以外の方法で来た場合に別のページが表示される。）
	delete sessionStorage.redirect;

	// 今回の特殊事情の関係で、ベースのURLを作成します。
	// http://USERNAME.github.io/SPA_Sample/NUM/XXXXX から /SPA_Sample/NUM/ だけ抽出します。
	const baseurl = pathname.replace( /^(\/[^\/]+\/[^\/]+).*$/, '$1' );

	// 今度はURLからSPAにとってのパスを取得します。
	// /SPA_Sample/NUM/XXXXX の /XXXXX の部分ですが、この中には/が入っている可能性もあります。
	const path = pathname.replace( /^\/[^\/]+\/[^\/]+(.*)$/, '$1' );

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

まず、普通にこのページが呼ばれた場合は `location.pathname` にデータがあります。
しかし、今回用意した `404.html` 経由で来た場合はURLは `http://USERNAME.github.io/SPA_Sample/1/` 固定になってしまいます。
リダイレクト前のURL情報が必要になります。

このリダイレクト前のURLは `sessionStorage.redirect` に入っています。
そこで、この値を取得して存在しない場合は `location.pathname` のデータを使うことにします。

次に `path` ですが、今回はURLの都合上このように少し複雑な正規表現を用いています。
中身はあまり理解しなくともよいです。
とにかく `/SPA_Sample/NUM/XXXXX` における `/XXXXX` の部分を取得していることだけ理解してください。

#### 確認

実際確認していきます。

* https://hirokimiyaoka.github.io/SPA_Sample/1/
  * 前のサンプルと同じ結果が表示されます。
* https://hirokimiyaoka.github.io/SPA_Sample/1/a
  * リダイレクトが走り、URLが `https://hirokimiyaoka.github.io/SPA_Sample/1/` になります。
  * コンテンツは `/docs/1/a.md` です。
* https://hirokimiyaoka.github.io/SPA_Sample/1/dir/
  * 挙動は上と同じです。ディレクトリ階層があっても表示可能です。
  * コンテンツは `/docs/1/dir/index.md` です。
* https://hirokimiyaoka.github.io/SPA_Sample/1/dir/b
  * 挙動は上と同じです。
  * コンテンツは `/docs/1/a.md` です。
* https://hirokimiyaoka.github.io/SPA_Sample/1/notfound
  * 存在しないコンテンツを読み込もうとしたので、エラーページが表示されました。

URLに応じて適切なコンテンツの表示ができ、SPAに近づきつつあるような気がします。

### コンテンツに応じて適切なURLを表示するページ

さて、先程のページは問題点があります。
URLに応じたコンテンツは表示できたものの、リダイレクトによって全てのページURLがおなじになってしまいます。
このままでは良くないのでURLを元に戻す作業をしたいと思います。

#### コピー

`/docs/1/` をフォルダごとコピーして `2` にリネームします。

そして `/docs/2/index.html` を次のように変更します。

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

// コンテンツのレンダリングを行います。
function Render( contents, data ) {
	contents.textContent = data;
}

// 初期化部分。
function Init() {
	// コンテンツを描画するHTML要素を取得します。
	const contents = document.getElementById( 'contents' );

	// 今回必要なパスは sessionStorage.redirect もしくは location.pathname の中に入っています。
	// 今回はここでどちらか値がある方のデータを取得します。
	// ちなみに、どちらも単純に代入しただけだと文字列としてコピーされず、値を変更した時に悲惨なことになるので必ず文字列にしておいた方がよいです。
	const pathname = ( sessionStorage.redirect || location.pathname ) + '';
	// sessionStorageのデータは消しておきます。
	delete sessionStorage.redirect;

	// 今回の特殊事情の関係で、ベースのURLを作成します。
	// http://USERNAME.github.io/SPA_Sample/NUM/XXXXX から /SPA_Sample/NUM/ だけ抽出します。
	const baseurl = pathname.replace( /^(\/[^\/]+\/[^\/]+).*$/, '$1' );

	// 今度はURLからSPAにとってのパスを取得します。
	// /SPA_Sample/NUM/XXXXX の /XXXXX の部分ですが、この中には/が入っている可能性もあります。
	const path = pathname.replace( /^\/[^\/]+\/[^\/]+(.*)$/, '$1' );

	// アドレスバーのURLを書き換えます。書き換えるだけなので履歴は残りません。
	history.replaceState( null, '', pathname );

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

// document.getElementById() を使っても大丈夫になったら初期化関数を実行する。
document.addEventListener( 'DOMContentLoaded', Init );
	</script>
</head>
<body>
<pre id="contents"></pre>
<a href="./a">a.md</a>
</body>
</html>
```

変更箇所は2箇所です。

`Init()` 内の `Fetch()` 直前でアドレスバーを書き換えています。

`history` を使うとアドレスバーの書き換えが可能の他、ブラウザの進む、戻る、履歴の追加などのブラウザ履歴を操作することが可能です。
SPAには欠かせない機能がいろいろ詰め込まれています。

また実験のためにHTMLソースにリンクも追加してあります。

#### 挙動確認

それではページを開いてみます。

https://hirokimiyaoka.github.io/SPA_Sample/2/a

こちらのページに遷移しても、アドレスバーのURLが `https://hirokimiyaoka.github.io/SPA_Sample/2/` に変更されません。
これでリダイレクト対策はできました。

しかし、今回追加したリンクをクリックしても、普通にページを読み込んでしまいます。
これではSPAにはなりません。

これから、ちゃんとしたSPAにします。

### リンクを制御するページ

ついにリンクを制御します。

リンクはクリックすると普通にサーバーにアクセスしてしまうため、ここの制御を乗っ取ってJavaScriptで制御することで、SPAの挙動にすることが出来ます。

そのためには2つの作業が必要です。

* 目的のパスを渡すとコンテンツを更新する仕組みを作る
* リンクを探してページ遷移の処理を行う仕組みを作る

#### 目的のパスを渡すとコンテンツを更新する仕組みを作る

今まで `Fetch()` や `Render()` という関数を用意してきましたが、このままでは多少使いづらいです。
ここで一気にSPAのクラスを作って、そこで管理できるように作り変えます。

`/docs/2/` をフォルダごとコピーして `3` にリネームします。

そして `/docs/3/index.html` を編集していきます。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
	<meta charset="utf-8">
	<title>SPA sample 0</title>
	<script>
// SPAのシステムです。
class App {
	// SPAで更新するコンテンツを設定します。
	constructor( contents ) {
		// コンテンツを更新するHTML要素を持っておきます。
		this.contents = contents;

		// ページに来た初回のレンダリング周りの処理を行います。
		const pathname = ( sessionStorage.redirect || location.pathname ) + '';
		delete sessionStorage.redirect;

		this.baseurl = pathname.replace( /^(\/[^\/]+\/[^\/]+).*$/, '$1' );

		// ページのパスを取得します。
		const path = pathname.replace( /^\/[^\/]+\/[^\/]+(.*)$/, '$1' );

		// アドレスバーのURLを書き換えます。書き換えるだけなので履歴は残りません。
		history.replaceState( null, '', pathname );

		this.renderPage( path );
	}

	// データを元にコンテンツのレンダリングを行います。
	render( data ) {
		this.contents.textContent = data;
	}

	// パスを元にコンテンツのレンダリングを行います。
	renderPage( path ) {
		return Fetch( baseurl, path ).then( ( md ) => {
			this.render( md );
		} ).catch( ( error ) => {
			// 何かしらのエラーが発生しました。
			this.render( '# Error' );
		} );
	}

	// 目的のページに遷移します。
	gotoPage( path ) {
		// 履歴に追加します。
		history.pushState( null, '', path + '' );

		return this.renderPage( path );
	}
}

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

// 初期化部分。
function Init() {
	// SPAの制御を行うクラスをインスタンス化します。
	const app = new App( document.getElementById( 'contents' ) );
}

// document.getElementById() を使っても大丈夫になったら初期化関数を実行する。
document.addEventListener( 'DOMContentLoaded', Init );
	</script>
</head>
<body>
<pre id="contents"></pre>
<a href="./a">a.md</a>
</body>
</html>
```

クラスを使ってかなり書き換えたものの、処理はほぼ増えていません。

ですが重要な変更が加えラッれています。
それがベースのURLやコンテンツのレンダリング対象が一括管理されるようになったことです。

例えば前の `Render( HTML要素, データ )` には書き込む対象のHTML要素とデータが必要でしたが、今回の改修で `app.render( データ )` というようにデータを渡すだけでよくなりました。

次に、ページ遷移が可能になりました。
`app.gotoPage( パス )` でアドレスバーをそのパスのURLに変更し、コンテンツも取得して描画することができるようになっています。

後は、どうにかしてリンクを自分の制御下に置きたいです。

ここで、次のようにリンクを何とかしたいと思います。

* とりあえず `<a>` を見つける。
  * `<button>` などのクリック時の処理はどうやってもこちらから手を出すべきではない。
* `href` の中身が存在しなかったり、`onclick` に何か関数が設定されている場合は何もしない。
* リンク先が自分の制御下にある。
  * 違うページは制御できない。
* `target` が指定されている場合は手を出さない。
  * 普通別のウィンドウ・タブにページが新たに開かれるので、自分の制御下から外れる。

これらの条件を満たすHTML要素に対して何かしらの処理をすれば良さそうです。
今回は `onclick` を無視するので、条件を満たす `<a>` を見つけたら `onclick` にページ遷移する関数を与えればうまくいきそうな気がします。

では実装です。

```js
class App {
	// ～省略～

}
```

