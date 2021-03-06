# 画像投稿機能の作成

## 学ぶこと
ここではユーザー登録に画像投稿機能を追加して、
画像の取り扱い方法に関して  
学びます。  

## 手順
大まかに以下の流れで作業を実施します。  
1. 画像投稿フォームの追加
2. バリデーションの追加
3. テーブルに画像の保存先pathを登録するカラムを追加
4. 画像の保存処理
5. 画像の表示

## 画像投稿フォームの追加
変更箇所は2箇所です。
1. formタグに `enctype="multipart/form-data"`を追記
2. 画像投稿フォームの追加

### formタグに `enctype="multipart/form-data"`を追記
こちらはフォームに画像などのファイルを投稿する機能をつける場合に必要になります。  

```php
// register.blade.php

<form method="POST" action="{{ route('register') }}" enctype="multipart/form-data"> // 編集

```

### 画像投稿フォームの追加

パスワード入力欄の下に以下のコードを追記します。  
画像を投稿するためのinputタグと、  
バリデーションでエラーになった場合の警告を表示するためのコードを追記してます。  

```php
// register.blade.php

<div class="form-group row">
    <label for="picture" class="col-md-4 col-form-label text-md-right">Profile picutre</label>

    <div class="col-md-6">
        <input id="picture" type="file" name="picture"
          class="form-control{{ $errors->has('picture') ? ' is-invalid' : '' }}"
        >

        @if ($errors->has('picture'))
            <span class="invalid-feedback" role="alert">
                <strong>{{ $errors->first('picture') }}</strong>
            </span>
        @endif
    </div>
</div>
```

## バリデーションの追加

```php
// RegisterController.php

 protected function validator(array $data)
 {
     return Validator::make($data, [
         'name' => ['required', 'string', 'max:255'],
         'email' => ['required', 'string', 'email', 'max:255', 'unique:users'],
         'password' => ['required', 'string', 'min:6', 'confirmed'],
         'picture' => ['required', 'image', 'mimes:jpeg,png,jpg,gif', 'max:2048'], // 追加
     ], [], [
         'name' => 'ユーザー名',
         'email' => 'メールアドレス',
         'password' => 'パスワード',  
         'picture' => 'プロフィール画像'  // 追加   
     ]);
 }
```

## テーブルに画像の保存先pathを登録するカラムを追加
以前のカリキュラムでも実施済みですが、テーブルにカラムを追加する手順は以下になります。  
1. マイグレーションファイルの作成
2. マイグレーションファイルの編集
3. マイグレーションの実行

### マイグレーションファイルの作成
以下のコマンドを実行してください。
```php
php artisan make:migration add_picture_path_to_users --table=users
```

### マイグレーションファイルの編集

`up` メソッドを以下の通り編集します。

```php
class AddPicturePathToUsersTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::table('users', function (Blueprint $table) {
            $table->string('picture_path')->after('remember_token');
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn('picture_path'); 
        });
    }
}

```

### マイグレーションの実行
```php
php artisan migrate:fresh
```

## 画像の保存処理
次はいよいよ画像の保存です。  
画像はMySQLなどのDBに保存できません。  
画像を保存する場合、画像を保存するためのディレクトリが必要になります。  

どのように管理するかというと、
1. 画像をどこかのディレクトリに保存
2. 1の画像が保存されているpath(住所のようなもの)をDBに保存
という形をとります。  


画像を保存するためのメソッドを作成
メソッドの戻り値はファイルの保存先にします。  
この戻り値をDBに保存します。  

また、ファイル名はランダムになるようにします。  
ユーザーが指定した名前をそのまま使用する場合、  
ファイル名が重複する可能性があるためです。  
```php
// RegisterController

 private function saveProfileImage($image)
 {
     // デフォルトではstorage/appに画像が保存されます。 
     // 第2引数にpublicをつけることで、storage/app/publicに保存されます。 
     // 今回は、/images/profilePictureをつけて、
     // storage/app/public/images/profilePictureに画像が保存されるようにしています。
     // 自分で指定しない場合、ファイル名は自動で設定されます。  
     $imgPath = $image->store('images/profilePicture', 'public');

    return 'storage/' . $imgPath;
 }
```

以下のコマンドでシンボリックリンク(ショートカットのようなもの)をpublicフォルダに作成します。  
// public直下のstorageがstorage/app/publicのショートカットになります。

```php
php artisan storage:link
```


アカウント登録するときのメソッド(create)に、  
1. 画像の保存を実施するメソッドの追加
2. DBに1で保存した画像のpathを保存するメソッドを追加します。  
```php
 // RegisterController

 protected function create(array $data)
 {
     $imgPath = $this->saveProfileImage($data['picture']);

     return User::create([
         'name' => $data['name'],
         'email' => $data['email'],
         'password' => Hash::make($data['password']),
         'picture_path' => $imgPath,
     ]);
 }
```

データを挿入するカラムが増えたので、`$fillable` に `picture_path` を追加します。
```php
// User.php
protected $fillable = [
  'name', 'email', 'password', 'picture_path'
];
```



## 画像の表示
最後にヘッダーのユーザー名の左側に画像を表示できるよう、
以下のコードを追加します。  

```php
// app.blade.php

<li class="nav-item">
  <img height="40px" src="{{ asset(Auth::user()->picture_path) }}" >
</li>

```

## おまけ
同じ要領でDiaryにも画像を投稿できます。  
余裕がある人は復習もかねて試してみましょう。

### テストデータの挿入

#### 画像の用意
任意の画像を以下のディレクトリに保存  
※ディレクトリがない場合は作成  
`storage/images/profilePicture/`

ここでは`default.jpg`という名前のファイルを保存した前提で進めます。  

#### マイグレーションファイルに画像ファイルのpathを追加

先ほど保存した画像のpathをDBに保存
```php
// UsersTableSeeder

 public function run()
 {
     DB::table('users')->insert([
         'name' => 'pikopoko',
         'email' => 'pikopoko@gmail.com',
         'picture_path' => 'storage/images/profilePicture/default.jpg', // 追加
         'password' => bcrypt('123456'),
         'created_at' => Carbon::now(),
         'updated_at' => Carbon::now(),
     ]);
 }
```

#### マイグレーションの実行
```php
php artisan migrate:fresh --seed
``` 
ログインしたアカウントで画像が表示されているか確認


## 参考リンク
[ファイルストレージ](https://readouble.com/laravel/5.7/ja/filesystem.html)