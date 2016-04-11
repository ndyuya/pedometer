# pedometer
Monacaとニフティクラウドmobile backendを使用した歩数計アプリのサンプル。

## 概要

クラウド上に計測した歩数を保存することができる歩数計アプリのサンプルです。
アプリの最初に会員認証を要求し、会員毎／１日毎のデータを保存します。
このため、機種変更への対応や１台の端末を複数人で共有するなどの場面でも利用可能です。

## 使い方

手順に従って作業を進めれば歩数計アプリを試すことができます。
クラウドとの連携を処理する一部のコードを自身で追記する通常版のフローと、
コード補完済みでAPIキーの設定のみで動作確認ができる簡易版のフローを用意しています。
お好きな方でお試しください。
なお、事前準備の項目はどちらのフローで進める場合でも必須です。

### 事前準備（通常版／簡易版 共通）

* [Monaca](https://ja.monaca.io/)の準備
 * [Monaca](https://ja.monaca.io/)の利用登録
 * [Monacaデバッガー](https://ja.monaca.io/debugger.html)のインストール
 * プロジェクトファイルのインポート
  1. [Download ZIP](https://github.com/ndyuya/pedometer/archive/master.zip)から、zipファイルをダウンロードする
  1. [Monaca](https://ja.monaca.io/)で「新規プロジェクト」＞「Import Project」からダウンロードしたzipファイルをインポートする
* [ニフティクラウドmobile backend](http://mb.cloud.nifty.com/)の準備
 * [ニフティクラウドmobile backend](http://mb.cloud.nifty.com/)の利用登録
 * [mobile backendのダッシュボード](https://console.mb.cloud.nifty.com/)で新しいアプリを作成する

### 通常版

1. [Monaca](https://ja.monaca.io/)に追加したプロジェクトを開く。
1. www/js/app.js を開く。（以降の編集作業は全てこのファイル）
1. www/js/app.js の `(1) SDKの初期化` の部分に以下のように記載する。「YOUR_APPLICATION_KEY」の部分は[mobile backend](https://console.mb.cloud.nifty.com/)で作成したアプリのアプリケーションキーに、「YOUR_CLIENT_KEY」の部分はクライアントキーに置き換えて記載すること。
   ```javascript
   // (1) SDKの初期化
   var ncmb = new NCMB("YOUR_APPLICATION_KEY",
                       "YOUR_CLIENT_KEY");
   ```

1. www/js/app.js の `(2) 会員の新規登録の処理` の部分に以下のように記載する。
   ```javascript
   // (2) 会員の新規登録の処理
   var signUp = function(email, password){
     var user = new ncmb.User({userName: email, password: password, mailAddress: email});
     user.signUpByAccount()
         .then(function(user){
           return ncmb.User.login(user);
         })
         .then(function(user){
           $('body').trigger('loginComplete');
         })
         .catch(function(err){
           $('body').trigger('ncmbError', [err, 'signup']);
         });
   };
   ```

1. www/js/app.js の `(3) 会員のログインの処理` の部分に以下のように記載する。
   ```javascript
   // (3) 会員のログインの処理
   var login = function(email, password){
     ncmb.User.login(email, password)
              .then(function(user){
                $('body').trigger('loginComplete');
              })
              .catch(function(err){
                $('body').trigger('ncmbError', [err, 'login']);
              });
   };
   ```

1. www/js/app.js の `(4) ログアウトの処理` の部分に以下のように記載する。
   ```javascript
   // (4) ログアウトの処理
   var logout = function(){
     ncmb.User.logout()
              .then(function(){
                $('body').trigger('logoutComplete');
              })
              .catch(function(err){
                $('body').trigger('ncmbError', [err, 'logout']);
              });
   };
   ```

1. www/js/app.js の `(5) クラウド上で歩数を管理する「Steps」クラスを定義する` の部分に以下のように記載する。
   ```javascript
   var Steps = ncmb.DataStore('Steps');
   ```

1. www/js/app.js の `(6) アプリ内に保持しいている未同期の歩数データをクラウドと同期させる処理` の部分に以下のように記載する。
   ```javascript
   var syncCloud = function(data, waitingList){
     // 今から保存する歩数データへのアクセス権限を自分自身だけに限定するためのACLを作る
     var currentUser = ncmb.User.getCurrentUser();
     var acl = new ncmb.Acl();
     acl.setUserReadAccess(currentUser, true)
        .setUserWriteAccess(currentUser, true);

     // 保存するデータを構築する
     var steps = new Steps({
       objectId: data.objectId,
       date: data.date,
       count: data.count,
       acl: acl
     });

     // save/updateメソッドでクラウド上へ保存する
     (!steps.objectId ? steps.save() : steps.update())
       .then(function(obj){
         Pedometer.steps[data.date].objectId = obj.objectId;
         $('body').trigger('syncNext', [waitingList]);
       })
       .catch(function(err){
         Pedometer.steps[data.date].synced = false;
         $('body').trigger('ncmbError', [err, 'syncCloud']);
         $('body').trigger('syncNext', [waitingList]);
       });
   };
   ```

1. www/js/app.js の `(7) ログイン完了時の処理` の部分に以下のように記載する。
   ```javascript
   var loginComplete = function(today){
     // ログイン完了後に自身の今日の歩数をクラウドから取得してPedometerに設定
     Steps.equalTo('date', today)
          .fetchAll()
          .then(function(objects){
            if (objects.length > 0) {
              var currentSteps = {
                count: objects[0].get('count'),
                objectId: objects[0].get('objectId'),
                date: objects[0].get('date')
              };
              Pedometer.setSteps(currentSteps);
            }
            Pedometer.refresh();
          })
          .catch(function(err){
            $('body').trigger('ncmbError', [err, 'loginComplete']);
          });
   };
   ```

1. [Monacaデバッガー](https://ja.monaca.io/debugger.html)で動作確認をする

### 簡易版

1. [Monaca](https://ja.monaca.io/)に追加したプロジェクトを開く
1. index.htmlの89行目を以下のように変更する
   ```html
   <script src="js/app.completed.js"></script>
   ```
1. [mobile backend](https://console.mb.cloud.nifty.com/)で作成したアプリのアプリケーションキーをコピーして、Monaca IDE上のwww/js/app.completed.jsの22行目にある「YOUR_APPLICATION_KEY」を置き換える
1. [mobile backend](https://console.mb.cloud.nifty.com/)で作成したアプリのクライアントキーをコピーして、Monaca IDE上のwww/js/app.completed.jsの23行目にある「YOUR_CLIENT_KEY」を置き換える
1. [Monacaデバッガー](https://ja.monaca.io/debugger.html)で動作確認をする
