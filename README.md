# Re:ゼロから始めるSPA

何がRe:なのか知りませんが、ゼロからSPAページを作りたいと思います。
（SPA周りの処理に関しては使用ライブラリなし、ES2015相当の生JavaScriptです。Markdownのレンダリングだけは許して……。）

## 注意事項

今回はGitHub Pagesで動作させるのでパス周りが若干特殊です。

通常SPAだと `https://domain/` 全てが使えるみたいな感じで使われるでしょう。
このようにすれば、URLの中でも `https://domain/XXXX` の XXXX の部分が直に使えるため便利なのと、絶対パスも `/` から始めるだけでお手軽です。

しかし、今回は通常のGitHub Pagesなので `https://USER.github.io/REPOSITORY/` 以下が制御対象になります。
それ加えその中でもバージョンの違いを意識したディレクトリがあるため、パスの階層が深いところがドキュメントルート扱いになっています。

そのため、例えば通常のSPAでは `/contents` で目的のコンテンツにアクセスできるはすが、`/REPOSITORY/NUM/contents` になってしまいます。

そこら辺は正規表現などでカバーしていますが、多少面倒くさいです。

## そもそもSPAって？

SPAは `Single Page Application` の略でWebページの一つの形です。

通常Webページはブラウザがサーバーにリクエストを投げると結果をHTML形式のデータで返してきます。（サーバー側のHTMLのレンダリング）

この受け取ったデータをブラウザが画面にレンダリングすることで、ページを表示しています。

このように二段階のレンダリングを経て表示されるのですが、これが同じようなページであればサーバーの処理も通信量も馬鹿になりません。

そこで、初めのレンダリングが全て終わった後全てのレンダリングの更新をJSにやらせて、サーバーとはその差分データだけやり取りさせることで、サーバーの処理も通信量も抑えることが可能です。

レンダリング後のやり取りは通常のネイティブアプリケーションと同じような挙動を取っているため、SPAはWebページと言うよりアプリケーションとして扱われているようです。

### SPAの実現のために必要なこと

ではこのSPAを実現するにはどんな機能が必要でしょう？

まずサーバー側です。
サーバー側は次の内どれかの機能を有していればSPAが実現できます。

* 未知のパス（SPAで処理をするパス）に対してSPAのページを返す。
  * 普通のWebサーバーはURL＝ファイルの場所だが、SPAの場合URL＝ファイルの場所ではない。
    * SPAで処理すべきパスにはファイルがないので、404エラーになってしまう。
  * そのため、サーバー側でこのパスはSPAのHTMLを返すなどの処理が必要になる。
    * 強いところだとサーバーサイドでレンダリングまでやってしまうこともある。
* 何らかの手段でリダイレクトを行える。
  * ファイルが存在しなくともちゃんとSPAのトップページにリダイレクトができれば、いろいろ対応が可能になる。

今回GitHub PagesはSPAに対応していませんが、404ページを自作できるのでそれを上手く使ってSPAっぽい挙動にします。

一方JavaScript側では次のような処理が必要になるでしょう。

* 最初のレンダリングの後は全てJavaScriptで管理する。
* リンクをクリックしてもページ遷移のリクエストをサーバーに送らずにコンテンツの更新を行う。
  * サーバーにはコンテンツだけ要求する。
* ブラウザの戻るボタンを押しても適切にコンテンツを更新する。

ここら辺ができればSPAと名乗れそうです。

### SPAを作ってみる

それではSPAを作っていきましょう。
今回はシンプルにHTMLファイル1つに全てを詰め込みます。

以下のようなSPAを目標にします。

* URL末尾に `.md` をつけたMarkdownファイルを読み込んでページに表示する。
  * 例えばURLが `https://domain/XXX` であれば、`https://domain/XXX.md` を読み込むということ。
  * もしURLが `/` で終わる場合は `index` というコンテンツへのアクセスとして扱う。
  * 今回は最後の仕上げ以外はMarkdownのソースを表示することとする。
* リンクを踏んでもページを読み込むのではなくコンテンツのみ読み込んで一部だけ再レンダリングする。
  * リンクを見つけてブラウザから処理を奪う。
* ブラウザの戻るなどが発生しても、再読込せず再レンダリングする。
  * ブラウザの履歴を操作する。

## SPA作成手順

段階を踏んで機能を追加しながらSPAを作ります。

* `index.md` の内容を取ってきて表示するだけのページ
* URLに応じて読み込むファイルを変えるページ
* コンテンツに応じて適切なURLを表示するページ
* リンクを制御するページ
* ブラウザ履歴を制御するページ
* 仕上げ

GitHub Pagesで作るので、フォルダ構成は以下になります。

```text
/docs/         ... GitHub Pagesのドキュメントルート
  0/           ... 各バージョンのSPAのドキュメントルート
    index.html ... SPA本体
    index.md   ... SPAのトップページのコンテンツ
    *.md       ... その他Markdownファイルはコンテンツとして適当に入れる
  1/
  :
  5/
```

URLは次のようになるはずです。

```text
http://USERNAME.github.io/SPA_Sample/NUM/
```

またリポジトリは `SPA_Sample` という想定で作っているので、ごく一部この値が入っている箇所があります。
（可能な限り取り除いている。）

### index.md の内容を取ってきて表示するだけのページ

初めの目標は `http://USERNAME.github.io/REPOSITORY/0/` にアクセスした時 `/REPOSITORY/0/index.md` を読み込んで表示することにします。

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
		// 結果をテキストにして取り出し、エラーとして処理します。
		return result.text().catch( ( error ) => { return error; } ).then( () => {
			throw new Error( result );
		} );
	} );
}

// コンテンツのレンダリングを行います。
function Render( contents, data ) {
	contents.textContent = data;
}

// 初期化部分。
function Init() {
	// コンテンツをレンダリングするHTML要素を取得します。
	const contents = document.getElementById( 'contents' );

	// 今回の特殊事情の関係で、ベースのURLを作成します。
	// http://USERNAME.github.io/REPOSITORY/NUM/XXXXX から /REPOSITORY/NUM/ だけ抽出します。
	// location.pathname は /REPOSITORY/NUM/XXXXX の部分を取得可能です。
	const baseurl = location.pathname.replace( /^(\/[^\/]+\/[^\/]+).*$/, '$1' );

	// 今度はURLからSPAにとってのパスを取得します。
	// /REPOSITORY/NUM/XXXXX の /XXXXX の部分ですが、この中には/が入っている可能性もあります。
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
やっていることは `http://USERNAME.github.io/REPOSITORY/NUM/XXXX` というURLだった場合、取得できるパスは `/REPOSITORY/NUM/XXXX` ですがここから `/REPOSITORY/NUM` を抽出しています。
GitHub Pagesで使うことを前提としているので、あえてこの部分を抽出して利用します。
（この作業は普通のSPAを作る時には必要ないことがほとんどかと思います。）

次に重要なのは `fetch()` についてです。
普通に使いたいところなのですが、404エラーなども正常に結果を受け取った判定になる特徴があります。
（エラーになるのはネットワークエラーとか。）

そこで404などにも対応すべく、ちょっとラップ処理をした `Fetch()` 関数を定義しています。

#### /docs/0/index.md

こちらはコンテンツです。

```md
# index.md

test
```

#### 動作サンプル

サンプルは実際に設置してあるSPAがあるのでそちらを貼っておきます。

https://hirokimiyaoka.github.io/SPA_Sample/0/

とりあえずMarkdownのソースを表示できているのが確認できます。

単にMarkdownのファイルを読み込んでいるだけなので、まだまだSPAっぽさは感じません。

### URLに応じて読み込むファイルを変えるページ

次にURLの内容に応じて読み込むファイルを変えるページを作ります。

具体的には以下のような感じです。

* `http://USERNAME.github.io/REPOSITORY/1/`
  * `/docs/1/index.md`
* `http://USERNAME.github.io/REPOSITORY/1/a`
  * `/docs/1/a.md`
* `http://USERNAME.github.io/REPOSITORY/1/dir/`
  * `/docs/1/dir/index.md`
* `http://USERNAME.github.io/REPOSITORY/1/dir/b`
  * `/docs/1/dir/b.md`
* `http://USERNAME.github.io/REPOSITORY/1/notfound`
  * このファイルは用意せず、エラーページとして見る。

#### Jekyllの無効化

ここら辺からGitHub Pagesで可動するJekyllが邪魔になる場合があるので無効化します。

`/docs/.nojekyll` という空ファイルを設置しておいてください。
これで機能を無効化することが出来ます。

#### 404対策

例えば `http://USERNAME.github.io/REPOSITORY/1/a` のページに遷移しようとした時、`/docs/1/a` というファイルがなければGitHub Pagesでは404ページが表示されてしまいます。

このようにファイルがない場合でも `/docs/1/index.html` をブラウザに返してもらわなければSPAが実現できません。
（逆に言えばこのようなリクエストに対して正しい `index.html` を返す仕組みがサーバーにないとSPAが作れないので注意。）

このような場合、GitHubでは以下のようなファイルを用意することが多いようです。（今回はこのファイルを使わないでください。）

##### /docs/404.html

```html
<!DOCTYPE html>
<html lang="ja"><head><title>Redirect</title><script>sessionStorage.redirect=location.href;</script><meta http-equiv="refresh" content="0; url=/REPOSITORY" /></head></html>
```

このファイルを用意した場合、`http://USERNAME.github.io/REPOSITORY/XXXXX` というURLは `http://USERNAME.github.io/REPOSITORY/` にリダイレクトされます。
（もちろんカスタムドメインを使った場合は `http://domain/` にリダイレクトできるよう、上の `/REPOSITORY` を `/` に書き換えます。）

そしてURL情報は `sessionStorage` に保存されます。

後はSPA内で `sessionStorage` 内に前のURL情報が入っていればその情報を使い、そうでない場合は `location.pathname` を使うことで、読み込むべきファイルパスを取得することが出来ます。

今回は特殊な仕様なので、以下のようなリダイレクトを行う404.htmlを作成してください。

```html
<!DOCTYPE html>
<html lang="ja"><head><title>Redirect</title><script>sessionStorage.redirect=location.pathname;const path=location.pathname.replace( /^(\/[^\/]+\/[^\/]+).*$/, '$1' );if(path!==location.pathname){location.href=path;}</script></head></html>
```

差分で特に重要なのは、`sessionStorage.redirect=location.pathname;` とリダイレクトです。
今回は `location.pathname` のみ保存していますが、これだとGETパラメーターが消失してしまいます。
本来であればちゃんと `location.href` でURLのパスとGETパラメーター等を全て渡すべきです。

リダイレクトは `http://USERNAME.github.io/XXXX/NUM/XXXXX` というURLに遷移した場合、`http://USERNAME.github.io/XXXX/NUM/`にリダイレクトします。
今回 `NUM` の部分は0～5でサンプルごとに異なるものの設置しているファイルが1つなので、それに対応するために正規表現でリダイレクト先を変更しています。

この処理は今回だけの例外なので、基本的には一つ上に用意した方のソースをベースにしてください。

#### 前のファイルのコピー

`/docs/0` をフォルダごとコピーして、フォルダ名を `1` に変更してください。
以後機能を追加するたびに同じように前のフォルダをコピーして編集していきます。

#### 必要コンテンツの追加

今のうちに必要なMarkdownファイルを用意しておきます。

また `/docs/1/index.md` も書き換えておきます。

##### /docs/1/index.md

```md
# index.md

Link

* [dir](./dir/)
* [a](./a)
```

##### /docs/1/a.md

```md
# a.md

a page!!!!
```

##### /docs/1/dir/index.md

```md
# dir/index.md

* [Back](../)
* [b](./b)
```

##### /docs/1/dir/b.md

```md
# dir/b.md

[Link](./)
```

#### ソースの改変

ではコピーした `/docs/1/index.html` のJS部分だけ修正していきます。

今回の目標はパスに応じで今まで固定だった `/index.md` を読み込む部分を適切に変えることです。

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
		// 結果をテキストにして取り出し、エラーとして処理します。
		return result.text().catch( ( error ) => { return error; } ).then( () => {
			throw new Error( result );
		} );
	} );
}
```

`Fetch()` 内では `path` の扱いをもう少ししっかりしています。
なにもない場合は `/` に書き換え、末尾が `/` の場合は `index` を追加し、最後に末尾に `.md` を追加します。

```js
// 初期化部分。
function Init() {
	// コンテンツをレンダリングするHTML要素を取得します。
	const contents = document.getElementById( 'contents' );

	// 今回必要なパスは sessionStorage.redirect もしくは location.pathname の中に入っています。
	// 今回はここでどちらか値がある方のデータを取得します。
	// ちなみに、どちらも単純に代入しただけだと文字列としてコピーされず、値を変更した時に悲惨なことになるので必ず文字列にしておいた方がよいです。
	const pathname = ( sessionStorage.redirect || location.pathname ) + '';
	// sessionStorageのデータは消しておきます。（消しておかないとリダイレクト以外の方法で来た場合に別のページが表示される。）
	delete sessionStorage.redirect;

	// 今回の特殊事情の関係で、ベースのURLを作成します。
	// http://USERNAME.github.io/REPOSITORY/NUM/XXXXX から /REPOSITORY/NUM/ だけ抽出します。
	const baseurl = pathname.replace( /^(\/[^\/]+\/[^\/]+).*$/, '$1' );

	// 今度はURLからSPAにとってのパスを取得します。
	// /REPOSITORY/NUM/XXXXX の /XXXXX の部分ですが、この中には/が入っている可能性もあります。
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
しかし、今回用意した `404.html` 経由で来た場合はURLは `http://USERNAME.github.io/REPOSITORY/1/` 固定になってしまいます。
リダイレクト前のURL情報が必要になります。

このリダイレクト前のURLは `sessionStorage.redirect` に入っています。
そこで、この値を取得して存在しない場合は `location.pathname` のデータを使うことにします。

次に `path` ですが、今回はURLの都合上このように少し複雑な正規表現を用いています。
中身はあまり理解しなくともよいです。
とにかく `/REPOSITORY/NUM/XXXXX` における `/XXXXX` の部分を取得していることだけ理解してください。

#### 確認

実際確認していきます。

* https://hirokimiyaoka.github.io/SPA_Sample/1/
  * 前のサンプルと同じ結果が表示されます。
* https://hirokimiyaoka.github.io/SPA_Sample/1/a
  * リダイレクトが走り、URLが `https://hirokimiyaoka.github.io/SPA_Sample/1/` になります。
  * 表示コンテンツは `/docs/1/a.md` です。
* https://hirokimiyaoka.github.io/SPA_Sample/1/dir/
  * 挙動は上と同じです。ディレクトリ階層があっても表示可能です。
  * 表示コンテンツは `/docs/1/dir/index.md` です。
* https://hirokimiyaoka.github.io/SPA_Sample/1/dir/b
  * 挙動は上と同じです。
  * 表示コンテンツは `/docs/1/a.md` です。
* https://hirokimiyaoka.github.io/SPA_Sample/1/notfound
  * 存在しないコンテンツを読み込もうとしたので、エラーページが表示されました。

URLに応じて適切なコンテンツの表示ができ、SPAに近づきつつあるような気がします。

### コンテンツに応じて適切なURLを表示するページ

さて、先程のページは問題点があります。
URLに応じたコンテンツは表示できたものの、リダイレクトによって全てのページURLが固定されてしまいます。
このままではWebページとして機能しているとは言えないので、URLをリダイレクト前の状態に戻す作業をしたいと思います。

#### コピー

`/docs/1/` をフォルダごとコピーして `2` にリネームします。

そして `/docs/2/index.html` を次のように変更します。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
	<meta charset="utf-8">
	<title>SPA sample 2</title>
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
		// 結果をテキストにして取り出し、エラーとして処理します。
		return result.text().catch( ( error ) => { return error; } ).then( () => {
			throw new Error( result );
		} );
	} );
}

// コンテンツのレンダリングを行います。
function Render( contents, data ) {
	contents.textContent = data;
}

// 初期化部分。
function Init() {
	// コンテンツをレンダリングするHTML要素を取得します。
	const contents = document.getElementById( 'contents' );

	// 今回必要なパスは sessionStorage.redirect もしくは location.pathname の中に入っています。
	// 今回はここでどちらか値がある方のデータを取得します。
	// ちなみに、どちらも単純に代入しただけだと文字列としてコピーされず、値を変更した時に悲惨なことになるので必ず文字列にしておいた方がよいです。
	const pathname = ( sessionStorage.redirect || location.pathname ) + '';
	// sessionStorageのデータは消しておきます。
	delete sessionStorage.redirect;

	// 今回の特殊事情の関係で、ベースのURLを作成します。
	// http://USERNAME.github.io/REPOSITORY/NUM/XXXXX から /REPOSITORY/NUM/ だけ抽出します。
	const baseurl = pathname.replace( /^(\/[^\/]+\/[^\/]+).*$/, '$1' );

	// 今度はURLからSPAにとってのパスを取得します。
	// /REPOSITORY/NUM/XXXXX の /XXXXX の部分ですが、この中には/が入っている可能性もあります。
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

この機能を実装するには2つの作業が必要です。

* 目的のパスを渡すとコンテンツを更新する仕組みを作る。
  * `https://hirokimiyaoka.github.io/SPA_Sample/3/a` の場合目的のパスは `/a` 等になる。
  * `/a` を渡すと `/SPA_Sample/3/a` のファイルを読み込む。
* リンクを探してページ遷移の処理を行う仕組みを作る。
  * ここでブラウザデフォルトの挙動であるサーバーへのリクエストを止める。

#### 目的のパスを渡すとコンテンツを更新する仕組みを作る

今まで `Fetch()` や `Render()` という関数を用意してきましたが、このままでは多少使いづらいです。
なぜなら、コンテンツのレンダリング先やパスの管理が関数では行えず、引数として与えなければならないからです。
（例えばページ遷移はパスを渡すだけでやってほしい。）

ここでそのような情報を持てるようにSPAの制御を行うクラスを作って手軽に使えるようにします。

`/docs/2/` をフォルダごとコピーして `3` にリネームします。

そして `/docs/3/index.html` を編集していきます。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
	<meta charset="utf-8">
	<title>SPA sample 3</title>
	<script>
// SPAのシステムです。
class App {
	// SPAで更新するHTML要素を設定します。
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
		return Fetch( this.baseurl, path ).then( ( md ) => {
			this.render( md );
		} ).catch( ( error ) => {
			// 何かしらのエラーが発生しました。
			this.render( '# Error' );
		} );
	}

	// 目的のページに遷移します。
	gotoPage( path ) {
		// 履歴に追加します。
		history.pushState( null, '', this.baseurl + path );

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
		// 結果をテキストにして取り出し、エラーとして処理します。
		return result.text().catch( ( error ) => { return error; } ).then( () => {
			throw new Error( result );
		} );
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

クラスを使ってかなり書き換えたものの、内部処理はほぼ変えていません。

重要な変更としては先程あったコンテンツのレンダリング先やパス等をクラス内で管理するようになったことで、いくつかの関数がかなり使いやすくなっています。

例えば前の `Render( HTML要素, データ )` には書き込む対象のHTML要素とデータが必要でしたが、今回の改修で `app.render( データ )` というようにデータを渡すだけでよくなりました。

今回実装したメソッドを一覧で書いておきます。

* `render( data )`
  * 第一引数はMarkdownのソースで文字列。
  * 実際にコンテンツのレンダリングを行います。
* `renderPage( path )`
  * 第一引数はパスで、 `https://hirokimiyaoka.github.io/SPA_Sample/3/XXXX` の場合は `/XXXX`
  * パスで指定されたMarkdownをダウンロードし、上の `render()` を使ってコンテンツのレンダリングを行います。
* `gotoPage( path )`
  * 第一引数は上と同じくパスです。
  * ブラウザにページ履歴を追加した上で、上の `renderPage()` を使います。

下から順番に依存している形になります。
細かく別れ過ぎではないかと思われますが、一番上は単なるエラーページの表示などのプログラムによるページの描画に必要です。
二番目は三番目とほぼ同じに見えますが、一番最初のレンダリングはブラウザ履歴に追加してはいけないため明確に区別する必要があります。

さて、これでページ遷移周りの準備は整いました。
後は、どうにかしてリンクを自分の制御下に置きたいです。

ここで、次のようにリンクを何とかしたいと思います。

* とりあえず `<a>` を見つける。
  * `<button>` などのクリック時の処理はどうやってもこちらから手を出せないため除外する。
  * JavaScriptでのクリック処理を無視する場合、HTMLのみでページ遷移できるのは `<a>` のみ。
* `href` の中身が存在しなかったり、`onclick` に何か関数が設定されている場合は何もしない。
  * `href` があっても `onclick` が設定されている場合は無視する。
* リンク先が自分の制御下にある。
  * 違うページは制御できないためURLの最初が一致する場合のみページ遷移処理を行う。
* `target` が指定されている場合は手を出さない。
  * 普通別のウィンドウ・タブにページが新たに開かれるので制御下から外れる。

これらの条件を満たすHTML要素に対して何かしらの処理をすれば良さそうです。
今回は `onclick` を無視するので、条件を満たす `<a>` を見つけたら `onclick` にページ遷移する関数を与えれば `addEventListener()` 利用時と異なり、複数回実行されても自分の処理が多重登録されることもなく上手くいきそうな気がします。

では実装です。

```js
class App {
	// ～省略～

	// 目的のページに遷移する関数を作ります。
	jumpPage( path ) {
		return ( e ) => {
			// クリック時の諸々のデフォルト処理を止めます。これでページ遷移を潰します。
			e.preventDefault();
			// SPAでのページ遷移を行います。
			this.gotoPage( path );
			return false;
		};
	}

	// リンクを探してページ遷移処理に書き換えます。
	convertAnchor( target ) {
		// targetに指定がない場合は、document.bodyを指定します。
		if ( target === undefined ) { target = document.body }
		// ベースとなるURLを作ります。
		const baseurl = location.protocol + '//' + location.host + this.baseurl;
		// <a>を探します。
		const anchors = target.getElementsByTagName( 'a' );
		for ( let i= 0 ; i < anchors.length ; ++i ) {
			// hrefが存在しないかonclickが設定されているかtargetが指定されている場合は無視します。
			if ( !anchors[ i ].href || anchors[ i ].onclick || anchors[ i ].target ) { continue; }
			// <a>のhrefはURLのフルパスが記載されているので、制御下のURLかどうか判定します。
			// URLがbaseurlから始まっていない場合は無視します。
			if ( anchors[ i ].href.indexOf( baseurl ) !== 0 ) { continue; }

			anchors[ i ].onclick = this.jumpPage( anchors[ i ].href.replace( baseurl, '' ) );
		}
	}
}

// ～省略～

// 初期化部分。
function Init() {
	// SPAの制御を行うクラスをインスタンス化します。
	const app = new App( document.getElementById( 'contents' ) );

	// 一度はページ全体のリンクを調べます。
	app.convertAnchor();
}
```

メソッドを追加したので説明です。

* `jumpPage( path )`
  * 第一引数はパス。
  * 返り値として渡したパスのページに遷移する関数を生成して返します。
* `convertAnchor( target )`
  * 第一引数はHTML要素。（省略した場合は `<body>` である `document.body` になる。）
  * 渡したHTML要素内の条件を満たす `<a>` に対し、ページ遷移を行う処理を追加します。

今回重要なのがクリック時の挙動です。
`convertAnchor()` では条件に合う `<a>` を見つけると、そこの `onclick` に対してブラウザによるページ遷移を殺してSPAによるページ遷移を行う関数を登録する必要があります。

そこで重要なのが、`jumpPage()` です。
この `jumpPage()` は `App`クラスのメソッドですが、やっていることは `onclick` に登録する関数の作成です。

この生成した関数はちょっと特殊で `onclick` で呼び出された時の第一引数 `e` に含まれれる `preventDefault()` を使うことでブラウザのデフォルトの挙動を封じることが出来ます。
これでクリックしてもサーバーにリクエストを飛ばすことはなくなり、`onclick` に設定した関数の処理だけが実行されます。

後は `gotoPage()` を使えばページ遷移ができます。

#### 実際の確認

https://hirokimiyaoka.github.io/SPA_Sample/3/

こちらにアクセスした後リンクをクリックします。

ページが再読込されずコンテンツが書き換わるかと思います。

かなりSPAっぽい挙動になってきました。

### ブラウザ履歴を制御するページ

さて、だいぶSPAっぽい挙動になってきたのですが、まだ一つ足りていない機能があります。

ブラウザの戻るへの対応です。

現在戻ってもコンテンツは更新されません。
それは、現在ページ遷移すると正しいURLを履歴に追加していますが、履歴を追加しただけなので戻ってもページ遷移が発生しません。
ここのイベントを取得して戻る進むなどのブラウザ操作があった場合に適切にコンテンツをレンダリングします。

`/docs/3/` をフォルダごとコピーして `4` にリネームします。

そして `/docs/4/index.html` に次の処理を追加します。

```js
class App {
	// SPAで更新するHTML要素を設定します。
	constructor( contents ) {
		// コンテンツを更新するHTML要素を持っておきます。
		this.contents = contents;

		// ページに来た初回のレンダリング周りの処理を行います。
		const pathname = ( sessionStorage.redirect || location.pathname ) + '';
		delete sessionStorage.redirect;

		this.baseurl = pathname.replace( /^(\/[^\/]+\/[^\/]+).*$/, '$1' );

		// ブラウザ履歴が変更される時にレンダリングを行います。
		window.addEventListener( 'popstate', () => { return this.onPopState(); }, false );

		// ページのパスを取得します。
		const path = pathname.replace( /^\/[^\/]+\/[^\/]+(.*)$/, '$1' );

		// アドレスバーのURLを書き換えます。書き換えるだけなので履歴は残りません。
		history.replaceState( null, '', pathname );

		this.renderPage( path );
	}

	// ブラウザ履歴変更時の処理です。
	onPopState() {
		// URLはすでに変更済みなので、現在のURLからレンダリング用のパスを取り出してレンダリングを行います。
		this.renderPage( location.pathname.replace( this.baseurl, '' ) );
	}


	// ～省略～
}
```

重要なのは `popstate` イベントとその挙動です。

このイベントはブラウザの戻るや進むで履歴が取り出されたときのイベントです。
そして、このイベントが呼び出されている時すでにURLは履歴から取り出したURLとなっています。

これが分かってしまえば、後はこのイベントでレンダリングの更新を行えば問題なくブラウザの戻る・進むに対応できます。

これでほぼSPAみたいなものです。

では、最後に仕上げをします。

### 仕上げ

このSPAはまだまだ粗が多いのですが、最後に2つ機能を実装して終わりにします。

* Markdownのレンダリング
  * 今回はCommonMarkを使います。
* 新規追加リンクの監視

このSPAはもう一つ問題を抱えていて、それはちゃんとコンテンツをレンダリングした後、リンクを適切にページ遷移処理に変更する事ができていません。
そこで、Markdownもきっちりレンダリングしつつ、変更があった（新しく作られた）リンクをSPA制御下に置くための監視処理を実装します。

`/docs/4/` をフォルダごとコピーして `5` にリネームします。

次にMarkdownのレンダリングのために、CommonMarkを用意します。

https://github.com/commonmark/commonmark.js/

こちらのページの `/dest/commonmark.min.js` をダウンロードして `/docs/5/` 内に入れてください。

そして `/docs/5/index.html` を次のように書き換えます。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
	<meta charset="utf-8">
	<title>SPA sample 4</title>
	<script src="./commonmark.min.js"></script>
	<script>
// SPAのシステムです。
class App {
	// SPAで更新するHTML要素を設定します。
	constructor( contents ) {
		// コンテンツを更新するHTML要素を持っておきます。
		this.contents = contents;

		// CommonMarkの初期化を行います。
		this.reader = new commonmark.Parser();
		this.writer = new commonmark.HtmlRenderer();

		// ページに来た初回のレンダリング周りの処理を行います。
		const pathname = ( sessionStorage.redirect || location.pathname ) + '';
		delete sessionStorage.redirect;

		this.baseurl = pathname.replace( /^(\/[^\/]+\/[^\/]+).*$/, '$1' );

		// ブラウザ履歴が変更される時にレンダリングを行います。
		window.addEventListener( 'popstate', () => { return this.onPopState(); }, false );

		// ページのパスを取得します。
		const path = pathname.replace( /^\/[^\/]+\/[^\/]+(.*)$/, '$1' );

		// アドレスバーのURLを書き換えます。書き換えるだけなので履歴は残りません。
		history.replaceState( null, '', pathname );

		this.renderPage( path );
	}

	// ブラウザ履歴変更時の処理です。
	onPopState() {
		// URLはすでに変更済みなので、現在のURLからレンダリング用のパスを取り出してレンダリングを行います。
		this.renderPage( location.pathname.replace( this.baseurl, '' ) );
	}

	// データを元にコンテンツのレンダリングを行います。
	render( data ) {
		// MarkdownをHTMLに変換します。
		this.contents.innerHTML = this.writer.render( this.reader.parse( data ) );

		// コンテンツが更新されたので、コンテンツ内のリンクを探して制御下に置きます。
		this.convertAnchor( this.contents );
	}

	// パスを元にコンテンツのレンダリングを行います。
	renderPage( path ) {
		return Fetch( this.baseurl, path ).then( ( md ) => {
			this.render( md );
		} ).catch( ( error ) => {
			// 何かしらのエラーが発生しました。
			this.render( '# Error' );
		} );
	}

	// 目的のページに遷移します。
	gotoPage( path ) {
		// 履歴に追加します。
		history.pushState( null, '', this.baseurl + path );

		return this.renderPage( path );
	}

	// 目的のページに遷移する関数を作ります。
	jumpPage( path ) {
		return ( e ) => {
			// クリック時の諸々のデフォルト処理を止めます。これでページ遷移を潰します。
			e.preventDefault();
			// SPAでのページ遷移を行います。
			this.gotoPage( path );
			return false;
		};
	}

	// リンクを探してページ遷移処理に書き換えます。
	convertAnchor( target ) {
		// targetに指定がない場合は、document.bodyを指定します。
		if ( target === undefined ) { target = document.body }
		// ベースとなるURLを作ります。
		const baseurl = location.protocol + '//' + location.host + this.baseurl;
		// <a>を探します。
		const anchors = target.getElementsByTagName( 'a' );
		for ( let i= 0 ; i < anchors.length ; ++i ) {
			// hrefが存在しないかonclickが設定されているかtargetが指定されている場合は無視します。
			if ( !anchors[ i ].href || anchors[ i ].onclick || anchors[ i ].target ) { continue; }
			// <a>のhrefはURLのフルパスが記載されているので、制御下のURLかどうか判定します。
			// URLがbaseurlから始まっていない場合は無視します。
			if ( anchors[ i ].href.indexOf( baseurl ) !== 0 ) { continue; }

			anchors[ i ].onclick = this.jumpPage( anchors[ i ].href.replace( baseurl, '' ) );
		}
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
		// 結果をテキストにして取り出し、エラーとして処理します。
		return result.text().catch( ( error ) => { return error; } ).then( () => {
			throw new Error( result );
		} );
	} );
}

// 初期化部分。
function Init() {
	// SPAの制御を行うクラスをインスタンス化します。
	const app = new App( document.getElementById( 'contents' ) );

	// 一度はページ全体のリンクを調べます。
	app.convertAnchor();
}

// document.getElementById() を使っても大丈夫になったら初期化関数を実行する。
document.addEventListener( 'DOMContentLoaded', Init );
	</script>
</head>
<body>
	<article style="max-width:800px;margin:auto;">
		<a href="/SPA_Sample/5/" style="display:block;text-align:center;">Top</a>
		<hr />
		<div id="contents"></div>
	</article>
</body>
</html>
```

変更点は `render()` 内でCommonMarkを使ったMarkdownのレンダリングを行っているところと、その後変更箇所に対してリンクを検索しているところです。

特に今回のリンクは次のような性質を持っています。

* ページ全体にリンクがあるかもしれないので、一度は全てのリンクを調べておきたい。
* コンテンツが更新されるところは限られているので、更新時にその中だけ調べて処理を置き換えれば問題がない。

これにより、今回はまだ更新処理が入っていなかった `render()` 周りの更新がメインとなっています。

ページリンクがちゃんとつながり、一番上にあるTopリンクを押してもリロードなくページが更新されます。
これで最低限それっぽいSPAになったのではないでしょうか。

## まとめ

SPAをゼロから作ってみました。

SPAの実現には以下の準備・実装が必要でした。

* ファイルが見つからない時にSPAのページを返す何らかのサーバー上の仕組み
  * SPAではファイルが存在しないパスでリクエストが飛んでくるため、サーバー側で対応するか別の手段でのリダイレクトが必要。
  * GitHub Pagesでは404ページにリダイレクトとURLの引き継ぎ処理を書いておくことで実現できた。
* リンクの制御
  * 書き換えるべきリンクを探し、その処理をSPA制御下に置くよう書き換えた。
  * 初めはページ全体、次は更新処理が走ったところだけ書き換えを行った。
* ブラウザ履歴の制御
  * リダイレクトからページが読み込まれた場合は今のURLを書き換えた。
  * SPAのページ遷移が発生した場合は、新しいURLの履歴を追加した。
  * ブラウザの戻る・進むが発生した場合は、現在のURLがすでに書き換わっているのでそこに応じた再レンダリング処理を追加した。

いくつか粗はあるものの、ここら辺を抑えておけば思ったよりも楽に実装できたかなと思います。
