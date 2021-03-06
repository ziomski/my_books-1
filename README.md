## Moje książki

Przykładowa aplikacja korzystająca z gemów *Carrierwave*
i *SimpleForm* oraz biblioteki JavaScript *Isotope*.

Linki do dokumentacji Carrierwave:

* [Home](https://github.com/jnicklas/carrierwave) –
  classier solution for file uploads for Rails,
  Sinatra and other Ruby web frameworks
* [Wiki](https://github.com/jnicklas/carrierwave/wiki)
* [Cropping Images](http://railscasts.com/episodes/182-cropping-images-revised?view=asciicast) –
  RailsCasts \#182
* A quick and simple image placeholder services:
  - [placehold](http://placehold.it/)
  - [placekitten](http://placekitten.com/)

*Simple Form*:

* [README](https://github.com/plataformatec/simple_form)


## Carrierwave na konsoli

Dopisujemy do *Gemfile*:

```ruby
gem 'rmagick', '~> 2.13.2'
gem 'carrierwave', '~> 0.10.0'

gem 'jcrop-rails-v2', '~> 0.9.12.3'

gem 'simple_form', '~> 3.0.1'
gem 'quiet_assets', '~> 1.0.2'
```

odinstalowujemy gem *turbolinks*, instalujemy nowy zestaw gemów
i uruchamiamy generator *simple_form:install*:

```sh
bundle install
rails generate simple_form:install
```

Kilka okładek pobrałem z [Surrealistyczne okładki książek Daniela Mroza](http://booklips.pl/galeria/surrealistyczne-okladki-ksiazek-daniela-mroza/),
z [Galerii okładek](http://home.agh.edu.pl/~evermind/okladki/galeria_okladek.htm),
[GitHub Octodex](http://octodex.github.com/),
i [Sticky Comics](http://www.stickycomics.com/where-did-you-meet/).

Obrazki zapisałem w katalogu *public*:

* Stanisław Jerzy Lec, Myśli nieuczesane. Wydawnictwo Literackie
* Stanisław Lem, Cyberiada. Wydawnictwo Literackie
* John Ronald Reuel Tolkien, Hobbit. Wydawnictwo Iskry
* John Ronald Reuel Tolkien, Rudy Dżil i jego pies. Wydawnictwo Amber
* The Kimonoctocat
* Where did you meet?

Jak działa Carrierwave? Na konsoli Rails wykonujemy kolejno:

```ruby
class MyUploader < CarrierWave::Uploader::Base
  storage :file
end
file = File.open('public/lem-cyberiada.jpg')
uploader = MyUploader.new
uploader.store! file
```

Jeśli nie było błędów, to w katalogu *public/uploads/* powinna się
pojawić kopia pliku *lem-cyberiada.jpg*.


## Scaffolding

Generujemy szablon CRUD dla modelu *Book*:

```sh
rails generate scaffold Book author title isbn price:integer
rake db:migrate
```

Generujemy uploader dla atrybutu *cover*, który dodajemy
via *migration* do modelu *Book*:

```sh
rails g uploader Cover
  create  app/uploaders/cover_uploader.rb
rails g migration add_cover_to_books cover:string
rake db:migrate
```

Wygenerowany uploader dopisujemy do modelu *Book*:

```ruby
class Book < ActiveRecord::Base
  mount_uploader :cover, CoverUploader
end
```

Dodatkowe atrybuty, z których będziemy korzystać dopisujemy w metodzie
*book_params* w kontrolerze *BooksController*:

```ruby
# Never trust parameters from the scary internet, only allow the white list through.
def book_params
  params.require(:book).permit(:author, :title, :isbn, :price,
      :cover, :remove_cover, :cover_cache, :remote_cover_url)
end
```

Teraz sprawdzimy na konsoli Rails, czy nie zrobiliśmy jakiś błędów:

```ruby
b = Book.new
b.title = "Octocat Story"
b.cover = File.open 'public/kimonotocat.jpg'
b.save!
b.cover.url           #=> "/uploads/book/cover/1/kimonotocat.jpg"
b.cover.current_path  #=> ".../public/uploads/book/cover/1/kimonotocat.jpg
b.cover.identifier    #=> "kimonotocat.jpg"
```

Na koniec odkomentowujemy w *cover_uploader.rb* metodę *extension_white_list*:

```ruby
class CoverUploader < CarrierWave::Uploader::Base
  def extension_white_list
    %w(jpg jpeg png)
  end
end
```

I dodajemy dwie wersje obrazków:

```ruby
class CoverUploader < CarrierWave::Uploader::Base
  include CarrierWave::RMagick

  process :resize_to_fit => [400, 400]

  version :thumb do
    process :resize_to_fit => [200, 400]
  end
end
```

Dokumentacja [CarrierWave::RMagick](http://rdoc.info/github/jnicklas/carrierwave/CarrierWave/RMagick):

* [instance methods (a-d)](http://www.imagemagick.org/RMagick/doc/image1.html)
* [instance methods (e-o)](http://www.imagemagick.org/RMagick/doc/image2.html)
* [instance methods (p-w)](http://www.imagemagick.org/RMagick/doc/image3.html)
  - [resize_to_fill](http://www.imagemagick.org/RMagick/doc/image3.html#resize_to_fill)
  - [resize_to_fit](http://www.imagemagick.org/RMagick/doc/image3.html#resize_to_fit)


## Rails 4

Zaczynamy od poprawianie szablonów.

Na początek szablon częściowy *_form.html.erb*,
w którym skorzystamy z gemu *simple_form*:

```rhtml
<%= simple_form_for(@book) do |f| %>
  <%= f.error_notification %>
  <div class="control-group string optional">
    <div class="controls">
    <% if @book.cover? %>
      <%= image_tag @book.cover_url(:thumb) %>
    <% end %>
    </div>
  </div>
  <div class="form-inputs">
    <%= f.input :cover, label: "Upload local file", as: :file %>
    <%= f.hidden_field :cover_cache %>
    <%= f.input :remote_cover_url, label: "or input URL" %>
    <% unless @book.new_record? %>
      <%= f.input :remove_cover, label: "remove cover", as: :boolean %>
    <% end %>
  </div>
  <div class="form-inputs">
    <%= f.input :author %>
    <%= f.input :title %>
    <%= f.input :isbn %>
    <%= f.input :price %>
  </div>
  <div class="form-actions">
    <%= f.button :submit %>
  </div>
<% end %>
```

W widoku *show.html.erb* dodajemy okładkę:

```rhtml
<p id="notice"><%= notice %></p>
<div class="cover">
  <%= image_tag @book.cover_url if @book.cover? %>
</div>
```

## Strona główna aplikacji

W *config/routes.rb* ustawiamy *root url* aplikacji:

```ruby
MyBooks::Application.routes.draw do
  resources :books
  root 'books#index'
```

W widoku *index.html.erb* skorzystamy z biblioteki
[Isotope](http://isotope.metafizzy.co/beta/).
Pobieramy plik [isotope.pkgd.min.js](http://isotope.metafizzy.co/beta/isotope.pkgd.min.js)
i zapisujemy go w katalogu *vendor/assets/javascripts/*.

Bibliotekę dopisujemy do pliku *app/assets/javascripts/application.js*
(i usuwamy dyrektywę *require_tree*):

```js
//= require jquery
//= require jquery_ujs
//= require isotope.pkgd.min
```

Usuwamy też *require_tree* z pliku *app/assets/stylesheets/application.css*
i dopisujemy *scaffold*:

```css
 *= require books
 *= require scaffolds
 *= require_self
```

W samouczku [The Asset Pipeline](http://edgeguides.rubyonrails.org/asset_pipeline.html)
jest więcej szczegółów na ten temat.

Dopisujemy do pliku *books.css.scss* ([SASS](http://sass-lang.com/)):

```scss
$book-background: #F0EAC8;
body {
  width: 100%;
}
.book {
  width: 200px;
  background-color: $book-background;
  margin-bottom: 20px;
}
```
i do pliku *books.js*:

```js
(function() {
  var configureIsotope = function() {
    $('#books-container').isotope({
      itemSelector: '.book',
      layoutMode: 'masonry',
      masonry: {
        gutter: 20
      }
    });
  };
  document.addEventListener("DOMContentLoaded", configureIsotope);
  // window.addEventListener("load", configureIsotope);
})();
```

*Uwaga:* Przy okazji usuwamy niepotrzebny plik
*books.js.coffee* ([CoffeeScript](http://coffeescript.org/)).

Teraz zajmiemy się widokiem *books/index.html.erb*:

```rhtml
<h1>My Books</h1>
<p><%= link_to 'New Book', new_book_path %></p>

<div id="books-container">

  <% @books.each do |book| %>
  <div class="book">
    <%= image_tag(book.cover_url(:thumb)) if book.cover? %>
    <%= book.author %><br>
    <%= book.title %><br>
    <%= book.isbn %><br>
    <%= book.price %><br>
    <div class="book-actions">
      <%= link_to 'Show', book %>
      <%= link_to 'Edit', edit_book_path(book) %>
      <%= link_to 'Destroy', book, method: :delete, data: { confirm: 'Are you sure?' } %>
    </div>
  </div>
  <% end %>
</div>

<%= javascript_include_tag "books" %>
```

### Refaktoryzacja widoku *index.html.erb*

Zastępujemy pętlę w widoku *index.html.erb* szablonem częściowym:

```rhtml
<h1>My Books</h1>
<p><%= link_to 'New Book', new_book_path %></p>
<div id="books-container">
  <%= render partial: 'book', collection: @books %>
</div>
<%= javascript_include_tag "books" %>
```

Następnie tworzymy nowy widok częściowy *_book.html.erb* o zawartości:

```rhtml
<div class="book">
  <%= image_tag(book.cover_url(:thumb)) if book.cover? %>
  <%= book.author %><br>
  <%= book.title %><br>
  <%= book.isbn %><br>
  <%= book.price %><br>
  <div class="book-actions">
    <%= link_to 'Show', book %>
    <%= link_to 'Edit', edit_book_path(book) %>
    <%= link_to 'Destroy', book, method: :delete, data: { confirm: 'Are you sure?' } %>
  </div>
</div>
```


## Jcrop

* [Jcrop](http://deepliquid.com/content/Jcrop.html),
* [źródło](https://github.com/tapmodo/Jcrop) (Github)

Bibliotekę Jcrop zainstalujemy w aplikacji Rails
korzystając z gemu *jcrop-rails-v2* oraz opisu
z [README](https://github.com/maxd/jcrop-rails-v2)
(dopisujemy Jcrop w *application.{css,js}*).

Widok *crop.html.erb*:

```rhtml
<h1>Crop Cover</h1>

<%= image_tag @book.cover_url, id: "jcrop_target" %>

<%= simple_form_for @book, html: { id: "coords" } do |f| %>
<% %w[x y w h].each do |attr| %>
  <%= f.input "crop_#{attr}" %>
<% end %>
  <div class="form-actions">
    <%= f.submit "Crop" %>
  </div>
<% end %>
<p>
<%= link_to 'Show', @book %> | <%= link_to 'Back', books_path %></p>

<%= javascript_include_tag 'crop' %>
```

Dopisujemy do routingu w pliku *routes.rb*:

```ruby
resources :books do
  member do
    get 'crop'
  end
end
```

Dopisujemy do kontrolera *books_controller.rb*
metodę *crop*:

```ruby
class BooksController < ApplicationController
  # dodajemy :crop
  before_action :set_book, only: [:show, :edit, :update, :destroy, :crop]

  # GET /books/1/crop
  def crop
  end
```

Sprawdzamy czy ten kod działa. W tym celu uruchuchomimy przykład
[Hello World!](http://deepliquid.com/projects/Jcrop/demos.php?demo=basic).

Dopisujemy do pliku *crop.js*:

```js
(function() {
  $('#jcrop_target').Jcrop({
    onChange: showCoords,
    onSelect: showCoords,
    minSize: [200, 200]
  });
  function showCoords(c) {
    console.log('x:', c.x, ' y:', c.y, 'w:', c.w, ' h:', c.h);
  };
})();
```

I wchodzimy na stronę, np. *localhost:3000/books/1/crop**.
Po kliknięciu w obrazek i przeciągnięciu myszką, powinnien się pojawić
zaznaczenie prostokątne, a na konsoli powinny być wypisywane aktualne
współrzędne zaznaczenia.


### Dalsze poprawki

W kontrolerze *BooksController.rb* dopisujemy atrybuty
*crop_{x,y,w,h}* do *book_params*:

```ruby
def book_params
  params.require(:book).permit(:author, :title, :isbn, :price,
      :cover, :remove_cover, :cover_cache, :remote_cover_url,
      :crop_x, :crop_y, :crop_w, :crop_h)
end
```

Atrybuty te dopisujemy też w pliku *book.rb* (tzw. wirtualne atrybuty):

```ruby
class Book < ActiveRecord::Base
  mount_uploader :cover, CoverUploader

  attr_accessor :crop_x, :crop_y, :crop_w, :crop_h

  after_update :crop_cover
  def crop_cover
    cover.recreate_versions! if crop_x.present?
  end
end
```

Przy okazji, po przycięciu układki, uaktualniamy wersje obrazków
(w naszej aplikacji wersję *thumb*)
a „przycinanie” obrazków dodajemy do *cover_uploader.rb*:

```ruby
class CoverUploader < CarrierWave::Uploader::Base
  include CarrierWave::RMagick

  version :thumb do
    process :crop
    process :resize_to_fit => [200, 400]
  end

  def crop
    if model.crop_x.present?
      manipulate! do |img|
        x = model.crop_x.to_i
        y = model.crop_y.to_i
        w = model.crop_w.to_i
        h = model.crop_h.to_i
        img.crop!(x, y, w, h)
      end
    end
  end
```

**Ważne:** Wyjaśnić jak to działa!

Do widoku *crop.html.erb* dodajemy formularz z atrybutami
*crop_{x,y,w,h}* użytymi w kodzie powyżej:

```rhtml
<h1>Crop Cover</h1>
<%= image_tag @book.cover_url, id: "jcrop_target" %>

<%= simple_form_for @book, html: { id: "coords" } do |f| %>
<% %w[x y w h].each do |attr| %>
  <%= f.input "crop_#{attr}" %>
<% end %>
  <div class="form-actions">
    <%= f.submit "Crop" %><!-- powinno przekierowywać na stronę główną -->
  </div>
<% end %>

<p><%= link_to 'Show', @book %> | <%= link_to 'Back', books_path %></p>
<%= javascript_include_tag 'crop' %>
```

Na koniec, zmieniamy kod w wykorzystanym powyżej w widoku *crop.js*.
Korzystamy ze zdarzeń *onChange* i *onSelect* aby zapisać współrzędne
w formularzu w elementach *input*:

```js
(function() {
  $('#jcrop_target').Jcrop({
    onChange: showCoords,
    onSelect: showCoords,
    minSize: [200, 200],
    onRelease: clearCoords,
    // http://deepliquid.com/content/Jcrop_Sizing_Issues.html
    boxWidth: 400,
    boxHeight: 400
  });
  function showCoords(c) {
    $('#book_crop_x').val(c.x);
    $('#book_crop_y').val(c.y);
    $('#book_crop_w').val(c.w);
    $('#book_crop_h').val(c.h);
  };
  function clearCoords() {
    $('#coords input').val('');
  };
})();
```

Powinniśmy też na konsoli Rails, przyciąć wszystkie stare wersje obrazków:

```ruby
Book.all.each do |book|
  book.cover.recreate_versions!
end
```

Kończymy zmiany dodaniem linka 'Crop cover' do widoku *show.html.erb*:

```rhtml
<p id="notice"><%= notice %></p>
<div class="cover">
  <%= image_tag @book.cover_url if @book.cover? %>
</div>

<div class="form-actions">
  <% if @book.cover %>
  <%= link_to 'Crop cover', crop_book_path(@book) %>
  <% end %>
</div>
```

KONIEC.


## powinno przekierowywać na stronę główną…

*config/routes.rb*:

```ruby
  resources :books do
    member do
      get 'crop'
      patch 'update_crop'
    end
  end
```

Dopisujemy do kodu kontrolera:

```ruby
class BooksController < ApplicationController
  before_action :set_book, only: [:show, :edit, :update, :destroy, :crop, :update_crop]
  # GET /books/1/crop
  def crop
  end
  # PATCH/PUT /books/1/update_crop
  def update_crop
    respond_to do |format|
      if @book.update(book_params)
        format.html { redirect_to books_url, notice: 'Cover was successfully updated.' }
      else
        format.html { render action: 'edit' }
      end
    end
  end
```

Zmieniamy jeden wiersz w widoku *crop.html.erb*:

```rhtml
<%= simple_form_for @book, url: update_crop_book_path(@book),
      html: { id: "coords" } do |f| %>
```

## Drugi model – Loan (wypożyczenia)

Nie będziemy korzystać z generatora *scaffold*. Dlaczego?

*Loan* krok po kroku:

```sh
rails generate model Loan lender:string returns:date book:references
      create    db/migrate/20140308184718_create_loans.rb
      create    app/models/loan.rb
rake db:migrate
```

Dopisujemy powiązania między modelami:

```ruby
belongs_to :book                       # w modelu Loan
has_many :loans, dependent: :destroy   #          Book
```

Zamieniamy *resources* routing na zagnieżdżony:

```ruby
resources :books do
  resources :loans
  member do
    get 'crop'
    patch 'update_crop'
  end
end
```
i generujemy kontroler dla wypożyczeń:

```sh
rails generate controller Loans
```

Wypożyczenia będziemy dodawać i wypisywać w widoku *articles/show*:

```rhtml
<h2>Wypożyczenia</h2>
<% @book.loans.each do |loan| %>
  <p>
    <%= loan.lender %>, zwrot do <%= loan.returns %>
  </p>
<% end %>

<h3>Nowe wypożyczenie</h3>
<%= simple_form_for([@book, @book.loans.build]) do |f| %>
  <div class="form-inputs">
    <%= f.input :lender %>
    <%= f.input :returns %>
  </div>
  <div class="form-actions">
    <%= f.button :submit %>
  </div>
<% end %>

<%= link_to 'Edit', edit_book_path(@book) %> |
<%= link_to 'Back', books_path %>
```

Po kliknięciu w przycisk *submit* jest wykonywana akcja
*create* kontrolera *LoansController*:

```ruby
class LoansController < ApplicationController
  def create
    @book = Book.find(params[:book_id])
    @loan = @book.loans.create(loan_params)
    redirect_to book_path(@book)
  end
  def destroy
    @book = Book.find(params[:book_id])
    @loan = @book.loans.find(params[:id])
    @loan.destroy
    redirect_to book_path(@book)
  end

  private
  def loan_params
    params.require(:loan).permit(:lender, :returns)
  end
end
```

**Uwagi:**

1. Zagnieżdżenie zasobów ⇒ *book_id* + *:id*.
2. Walidacja jest trikowa! Wyjaśnić dlaczego?


### Obowiązkowa refaktoryzacja

Widok zawiera za dużo kodu. W widoku *articles/show*
wydzielimy dwa widoki częściowe:

```rhtml
<h2>Wypożyczenia</h2>
<%= render @book.loans %>
<h3>Nowe wypożyczenie</h3>
<%= render "loans/form" %>
```

*loans/_loan.html.erb*:

```rhtml
<p>
  <%= loan.lender %>, zwrot do <%= loan.returns %>
</p>
```

i *loans/_form.html.erb*:

```rhtml
<%= simple_form_for([@book, @book.loans.build]) do |f| %>
  <div class="form-inputs">
    <%= f.input :lender %>
    <%= f.input :returns %>
  </div>
  <div class="form-actions">
    <%= f.button :submit %>
  </div>
<% end %>
```

### Usuwanie wypożyczeń

Dopisujemy link *Usuń wypożyczenie* do widoku

```rhtml
<p>
  <%= link_to "Destroy Loan", [loan.book, loan],
        method: :delete, data: { confirm: "Jesteś pewien?" } %>
</p>
```
Metodę *destroy* do kontrolera *LoansControllers*
dodaliśmy wcześniej.


## Basic Authentication

Dopisujemy do obu kontrolerów:

```ruby
class BooksController < ApplicationController
   http_basic_authenticate_with name: "admin", password: "sekret", except: [:index, :show]

class LoansController < ApplicationController
   http_basic_authenticate_with name: "admin", password: "sekret", only: :destroy
```


## ISBN API

Dodać możliwość korzystania z ISBN. Zobacz:

* http://stackoverflow.com/questions/1297700/what-is-the-most-complete-free-isbn-api
* http://isbndb.com, http://isbndb.com/api/v2/docs
* https://developers.google.com/books/docs/getting-started?hl=pl
* https://developers.google.com/books/docs/v1/getting_started?hl=pl


## Odpluskwianie kodu

1. W widoku dodajemy kod podobny do tego:

```rhtml
<div style="clear: both">
  <%= debug @book %>
</div>
```


## Running app in development

* Ruby version: **2.1.0**, gemset: **my_books**

* Rails version: **4.1.0.rc1**

* System dependencies: ImageMagick, SQLite

* Database creation
  ```
  rake db:migrate
  ```
