---
title: Rails4のマイグレーションの書き方まとめ
date: 2014-12-27
tags: rails
---

## はじめに

Ruby 2.2.0 が登場し、Rails 5 の噂も聞くようになった今日この頃ですが、ここで Rails 4 のマイグレーションの書き方についておさらいしておきたいと思います。
Rails 4 系で新しくプロジェクトを作成しようとしている方は必見かもです。


## マイグレーションとは？

この記事ではデータベースのスキーマ・マイグレーションを指します。

そもそもデータベースのテーブル/カラムを定義するのになぜわざわざ「マイグレーション(移住/転住)」が必要なのか、という話についてはこちらの記事が詳しいのでご参照ください。
>
> マイグレーション機能とは
>
> <a href="http://www.rubylife.jp/rails/model/index5.html" target="_blank">http://www.rubylife.jp/rails/model/index5.html</a>
>


## Rails 2, 3 のマイグレーション

ちょっと前のプロジェクトでは下記のようなマイグレーションファイルをよく見かけました。
例えば新しくテーブルを作る場合はこんな具合です。

```ruby
class CreateUsers < ActiveRecord::Migration
  def self.up
    create_table :users do |t|
      t.column :email, :string, null: false
    end
    add_index :users, :email
  end

  def self.down
    remove_index :users, :email
    drop_table :admin_users
  end
end
```

そしてそのテーブルに後からカラムやインデックスを追加するときにこのように書いていました。

```ruby
class AddNameToUsers < ActiveRecord::Migration
  def self.up
    add_column :users, :name, :string, null: false
    add_index :users, :name
  end

  def self.down
    remove_index :users, :name
    remove_column :users, :name
  end
end
```

この書き方は Rails 4 でも可能ではありますが、もう少し分かりやすい形式に書き直すことができます。


## Rails 4 のマイグレーション

先ほど紹介したマイグレーションファイルを、Rails 4 では下記のように書き直すことができます。

```ruby
class CreateUsers < ActiveRecord::Migration
  def change
    create_table :users do |t|
      t.column :email, :string, null: false
      t.index :email
    end
  end
end
```

先ほどのマイグレーションファイルとの違いを見ていきましょう。
`self.up` と `self.down` というメソッドがなくなり、代わりに `change` というメソッド１つになりました。
また、インデックス作成のための `add_index` がなくなり、代わりに `create_table` のブロックの中に `t.index` というメソッドの呼び出しが増えています。
先ほどのマイグレーションファイルと比べると、記述の量が減り、大分シンプルになっています。

もちろんこの構文はロールバックにも対応しており、ロールバック時にはテーブルの削除を行ってくれます。

次にカラム/インデックスを追加する場合の構文について紹介します。

```ruby
class AddNameToUsers < ActiveRecord::Migration
  def change
    change_table :users do |t|
      t.column :name, :string, null: false
      t.index :name
    end
  end
end
```

新しく `change_table` というメソッドが登場しています。
このメソッドを使うことで `create_table` とまったく同じ構文でカラムやインデックスの追加が可能になります。

こちらもロールバック可能で、ロールバック時には `change_table` で追加したカラムやインデックスだけを消してくれます。


## カラムに削除や変更が必要なときはどうするか

Rails 4 ではテーブルの作成やカラムの追加が、キレイにまとまった形で記述できることが分かりました。
ではカラムを削除したり変更する場合はどのようにしたら良いでしょうか。

カラムの属性変更は、テーブルの作成やカラムの追加と同じように `change_table` を使いながら次のように書くことができます。

```ruby
class ChangeNameOnUsers < ActiveRecord::Migration
  def change
    reversible do |dir|
      change_table :users do |t|
        dir.up   { t.change :name, :string, null: true }
        dir.down { t.change :name, :string, null: false }
      end
    end
  end
end
```

`reversible` メソッドを利用することで、`self.up` や `self.down` のようにマイグレーションの方向によって処理内容を変えることができます。
これによって `change_table` を使いながらカラムの属性を変更したりロールバックしたりすることができるようになります。

そしてカラムを削除するときにも、同じ構文が利用できます。
その際には `t.remove` というメソッドを利用します。

```ruby
class RemoveNameFromUsers < ActiveRecord::Migration
  def change
    reversible do |dir|
      change_table :users do |t|
        dir.up   { t.remove :name }
        dir.down { t.string :name, null: false }
      end
    end
  end
end
```

ただし、`reversible` を利用すると少し構文が複雑になってしまいます。
これが嫌な場合は、カラムの削除を下記のように書くことも可能です。

```ruby
class RemoveNameFromUsers < ActiveRecord::Migration
  def change
    remove_column :users, :name, :string, null: false
  end
end
```

削除の構文なのにカラムの型やその他オプションを指定しているのが不思議に感じますが、これはロールバックの際に利用される情報になります。
`reversible` メソッドを利用するよりは大分シンプルになりました。


## さいごに

Rails 4 のおさらいということでマイグレーションファイルの書き方についてまとめてみました。
プロジェクトによってはDB設計が変わることはそうそうないことだとは思いますが、だからこそ忘れがちなので、覚え書きとして活用いただければと思います。
