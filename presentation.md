---
marp: true
---

# Rails 6.1

@willnet

---

# フリーランスRails技術顧問をしつつ、空いた時間でsavanna.ioなどを開発しています

![bg](savanna-lion-face.png)

---

# Active Storageのサムネイル管理方法の変更

- [Track Active Storage variants in the database by georgeclaghorn · Pull Request #37901 · rails/rails](https://github.com/rails/rails/pull/37901)
- これまで、サムネイルを表示するときには次のような挙動になっていた(ストレージがS3だと仮定)
  1. サムネイルがS3に存在するかどうかをチェック
  2. 存在しなければ元画像をダウンロードしてサムネイルを作りS3にアップロードする
  3. サムネイルの画像用のURLを作成する
- この方式は次の問題があった
  - 毎回S3にサムネイルが存在するか確認しにいくので、その分レスポンスを返すまでに時間がかかる
  - S3の仕様的に、存在確認後にすぐアップロードすると、しばらくの間アップロードしたのにファイルがない、ということになる
    - たぶん存在確認の結果をしばらくキャッシュしてるんじゃないかな…

---

- これを解決するために、サムネイルが存在しているかどうかを保持しておくテーブルが新設された
- `Rails.application.config.active_storage.track_variants = true`とすると新しいやりかたが有効になる

---

# CSRFトークンのエンコード形式の変更

- [Accept and default to base64_urlsafe CSRF tokens. by dragonsinth · Pull Request #18496 · rails/rails](https://github.com/rails/rails/pull/18496)
- CSRFトークンのエンコード方式がbase64からurlsafe版のbase64に変更されました
  - base64はa-z,A-Z,0-9,+,/でエンコードする方式
  - `+`と`/`はURLエンコードしないといけない→それぞれ`-`と`_`に変更した
  - Rubyにはurlsafe版のbase64エンコード用メソッドがある`Base64.urlsafe_encode64`
- CSRFトークンをcookieとしてそのまま送りたいようなケースで、クライアント側でエンコード・デコードが必要になるのがだるいのでこうなったとのこと

---

- 僕らの生活的にはあまり影響はないのだけど、「6.1アップグレード時にCSRFトークンのエンコード方式が変わる」というのは影響がある
  - ローリングアップデートなどで6.0と6.1のサーバが混在しているタイミングがあると、エンコード形式とデコード形式が変わってCSRFトークンのチェックがうまくいかずにエラーになる
  - 6.1にアップグレードしたあとに障害があり、6.0にロールバックしたらエンコード形式とデコード形式が変わって(ry
  - 6.0形式でエンコードされたものを6.1でデコードすることはできるようになっている
  - `Rails.application.config.action_controller.urlsafe_csrf_tokens = true`で挙動を切り替えることができるので、いったんfalseの状態で6.1にアップグレードし、折を見て一気にtrueにするのがよさそうです

---

# whereに新しいオペレータが追加

- [Support `where` with comparison operators (`>`, `>=`, `<`, and `<=`) by kamipo · Pull Request #39613 · rails/rails](https://github.com/rails/rails/pull/39613)


```ruby
posts = Post.order(:id)

posts.where("id >": 9).pluck(:id)  # => [10, 11]
posts.where("id >=": 9).pluck(:id) # => [9, 10, 11]
posts.where("id <": 3).pluck(:id)  # => [1, 2]
posts.where("id <=": 3).pluck(:id) # => [1, 2, 3]
```

---

- 単に書き方が違うだけで `posts.where("id > ?", 9).pluck(:id)` と機能的には同じ…ではない
- 新しい書き方は、[ActiveRecord::Attribute](http://api.rubyonrails.org/classes/ActiveRecord/Attributes/ClassMethods.html#method-i-attribute)の設定を反映してくれる

---

```ruby
class Post < ActiveRecord::Base
  attribute :created_at, :datetime, precision: 3
end

time = Time.now.utc # => 2020-06-24 10:11:12.123456 UTC

posts = Post.order(:id)
posts.create!(created_at: time) # => #<Post id: 1, created_at: "2020-06-24 10:11:12.123000">

# SELECT `posts`.* FROM `posts` WHERE (created_at >= '2020-06-24 10:11:12.123456')
posts.where("created_at >= ?", time) # => []

# SELECT `posts`.* FROM `posts` WHERE `posts`.`created_at` >= '2020-06-24 10:11:12.123000'
posts.where("created_at >=": time) # => [#<Post id: 1, created_at: "2020-06-24 10:11:12.123000">]
```
---

- attributeメソッドを使ってデフォルト以外の設定にする、というケースはそれほどないけどいざ設定するぞ！となったときに便利なので、新しい書き方でいけるときは積極的に使っていくのが良いのかなと思いました
- そもそもattributeメソッドはけっこう便利に使えるケースがありそうなので覚えておくと良さそう
  - http://api.rubyonrails.org/classes/ActiveRecord/Attributes/ClassMethods.html#method-i-attribute

---

# ActiveModel::Errorsの改善

- [Model error as object by lulalala · Pull Request #32313 · rails/rails](https://github.com/rails/rails/pull/32313)
  - ActiveModel::Errorsは実質的に2つのHashオブジェクトから構成されている
    - messages
      - ex: `{:email=>["を入力してください"]}`
    - details
      - ex: `{:email=>[{:error=>:blank}]}`
  - Hashをゴニョゴニョするのがめんどくさいケースが有る
    - ex: 特定のメッセージに紐づくdetailを取得したいときに、messagesのindexを取得して`details[:email][index]`としなければいけない

---

- 今回の修正により、ActiveModel::Errorsは実質的にActiveModel::Errorオブジェクト(新設)の集合となった
- これによりエラー内容に対しての細かい操作が、よりオブジェクト指向っぽい書き方ができるようになった(Hashもオブジェクトではあるけどね)
  - 関連先のエラーもいい感じに表現できている(ActiveModel::Errorsのネストができる)
  - Errorsにwhereメソッドが生えた
    - `model.errors.where(:name, :foo, bar: 3).first`
  - messageに紐づくdetailが簡単に取得できるようになった  　
- 非互換な変更なので対応が必要なものもある
  - `book.errors[:title] << 'is not interesting enough.'` のような、直接Hashをいじるような操作はdepricatedになった
    - `book.errors.add(:title, 'is not interesting enough.')` のように書く
    - `book.errors.each do |attribute, error_message| ...` は depreated
    - `book.errors.each do |error|` のように変更する
  - 具体的な嬉しいケースは思いつかないけど、hanicaみたいな関連モデルを大量に保存するようなアプリケーションだと、エラー内容の解析やらなんやらが楽になってよいのでは

---

# ARにsigned_idメソッドとfind_signedメソッドが追加

- [Add signed ids to Active Record by dhh · Pull Request #39313 · rails/rails](https://github.com/rails/rails/pull/39313)
  - idを暗号化かつ改ざん検知ができるトークンを生成してそれでfindできるようにしたsigned_idメソッドとfind_signedメソッドが追加されました
  - トークンに期限を含めることができます
  - ↓例にあるようにパスワードリセット機能などを作るときに便利そう

```ruby
signed_id = User.first.signed_id expires_in: 15.minutes, purpose: :password_reset

User.find_signed signed_id # => nil, since the purpose does not match

travel 16.minutes
User.find_signed signed_id # => nil, since the signed id has expired

travel_back
User.find_signed signed_id, purpose: :password_reset # => User.first

User.find_signed! "bad data" # => ActiveSupport::MessageVerifier::InvalidSignature
```

---

## [Active storage add proxying by fleck · Pull Request #34477 · rails/rails](https://github.com/rails/rails/pull/34477)


- Active Storageでアップロードしたファイルへのアクセスは次のような挙動だった
  1. まずクライアントがRailsサーバへリクエストを送る
  2. 時間制限(デフォルト5分)つきのファイルへのURLを生成してリダイレクトする
  3. クライアントはファイルをダウンロードできる
- 制限付きのリンクがいい感じに生成されてべんりなのだけど、広く公開して問題ないようなファイルだと無駄が多い
- ↓のようにするとファイルの中身を直接返すURLを生成する
  - 設定でデフォルトの挙動を変更することもできる

```
<%= image_tag rails_storage_proxy_path(@user.avatar) %>
```
---

## [Deprecate inconsistent behavior that merging conditions on the same column by kamipo · Pull Request #39328 · rails/rails](https://github.com/rails/rails/pull/39328)

- `AR.merge`の挙動変更
  - Rails6.1までのmergeは同じカラムがmergeされるときに、Hashのように後勝ち(mergeの引数側が採用される)になるケースと、ならないケース(両方のクエリが統合される)がある
  - 後勝ちになったりならなかったりするのは意図的なものではないので後勝ちになるように修正する
  - 6.1では後勝ちにならないケースでdeprecateメッセージが表示され、6.2でデフォルト後勝ちになる
  - 6.1で6.2の挙動(後勝ちにしたければrewhereオプションを指定する)

---

```ruby
# Rails 6.1 で後勝ちになるパターン(IN句)
Author.where(id: [david.id, mary.id]).merge(Author.where(id: bob)) # => [bob]

# Rails 6.1 で両方の指定が採用されるパターン
# where id between "davidのid" and "maryのid" and id = bobのid
Author.where(id: david.id..mary.id).merge(Author.where(id: bob)) # => []

# 6.1で6.2相当の挙動を使うにはrewhereを使う
Author.where(id: david.id..mary.id).merge(Author.where(id: bob), rewhere: true) # => [bob]

# Rails 6.2ではどのようなケースでも後勝ち
Author.where(id: [david.id, mary.id]).merge(Author.where(id: bob)) # => [bob]
Author.where(id: david.id..mary.id).merge(Author.where(id: bob)) # => [bob]
```
---

後勝ちにしない、を明示的にするためのメソッドも追加ずみ [Support `relation.and` for intersection as Set theory by kamipo · Pull Request #39558 · rails/rails](https://github.com/rails/rails/pull/39558)

```ruby
david_and_mary = Author.where(id: [david, mary])
mary_and_bob   = Author.where(id: [mary, bob]) # => [bob]

david_and_mary.merge(mary_and_bob) # => [mary, bob]

david_and_mary.and(mary_and_bob) # => [mary]
david_and_mary.or(mary_and_bob)  # => [david, mary, bob]
```
