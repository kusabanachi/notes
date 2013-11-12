## Ubuntuにmonodevelop + F# のインストール
* Ubunutu12.04で確認
* 2013年11月12日時点での情報

### monodevelopをインストールする
* パッケージからのインストールだとmonodevelopのバージョンが低く、F#が使えないのでソースコードからインストールする

#### monodevelopのソースコードをダウンロードする
```
    $ git clone git://github.com/mono/monodevelop.git
```

#### monodevelopビルドと実行に必要なパッケージを取得する
```
    $ sudo apt-get build-dep monodevelop
    $ sudo apt-get install mono-gmcs
    $ sudo apt-get install mono-runtime-sgen
```

#### ビルドするmonodevelopのバージョンを指定する
* apt-getで得られるmonoはバージョン2.10.8.1
* monodevelop-4.0.10以降をビルドしようとすると、mono 2.10.9以降が必要と言われてエラーになるのでビルド可能なmonodevelop-4.0.1を指定する

```
    $ cd monodevelop
    $ git tag                         // タグ一覧が見える
    $ git checkout monodevelop-4.0.1
```

#### monodevelopをビルドする
* 以降はREADME.mdに書いてある通りにビルドする

```
    $ git submodule update --init --recursive
    $ ./configure
    $ make
```

* ビルドが成功すれば、以降このディレクトリで`make run`を実行すればインストールせずにmonodevelopを起動できるようだ
* `sudo make install`でインストールすることもできる


### F#をインストールする
* fsharpパッケージはSaucy Salamander以降の提供のようなので、こちらもソースコードからインストールする

```
    $ cd ../
    $ git clone git://github.com/fsharp/fsharp
    $ cd fsharp
```

#### F#をビルドする
* README.mdに書いてある通りにビルドしてインストールする

```
    $ ./autogen.sh --prefix=/usr
    $ make
    $ sudo make install
```

* prefixの指定をしないとmonoと同じパスでないと警告がでる (mono 3.0以降が推奨という警告も出る)

### monodevelopでF#を使う
1. monodevelopを起動させる
2. Tools -> Add-in Manager -> Galleryタブ -> Language bindings -> F# Language Binding -> Install

* FileやWorkspaceの新規作成からF#が選べられるようになる

######おわり

