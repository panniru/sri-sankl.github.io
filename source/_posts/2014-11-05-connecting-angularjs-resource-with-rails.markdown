---
layout: post
title: "Connecting AngularJs Resource with Rails"
date: 2014-11-05 21:59:12 +0530
comments: true

---
RAILS [RESTful](http://en.wikipedia.org/wiki/Representational_state_transfer) web services have made my web application development much simpler and faster. There are several javascript frameworks which provides RESTful services. [Angular.js](https://angularjs.org/) is one of the best and popular javasript framework among them. It provides RESTful services using its `ng_resouce` API.
##Implementing a sample app using Rails, *ng, ng_resouce and ng_routes*.

####Gems used:
``` ruby Gemfile  
  gem 'haml-rails', '~> 0.5.3'
  gem 'angularjs-rails', '~> 1.2.22'
  gem 'angular-rails-templates'  
```
<!--more-->
 1. [Haml](http://haml.info) is a markup language used to write a well-intented and a clear structured  HTML. [haml-rails](https://github.com/indirect/haml-rails) is a templating engine which converts haml into html.
 2. [angularjs-rails](https://rubygems.org/gems/angularjs-rails) is the rails gem for angular.js library.
 3. [angular-rails-templates](https://github.com/pitr/angular-rails-templates) is used to load  angular templates as part of asset pipeline.
    - Angular engine cannot convert haml templates into html while rendering. Hence the rendered content will be shown as a text in the web page.
    -  `angular-rails-templates` converts the haml files into html by using the `haml` gem. These converted html files will be loaded with the assets.
    -  Since templates are cached, we can avoid the ajax calls required by angular engine to render the templates.
       <br>
        *Create a separate directory for templates
           <br/> 
         `$ mkdir app/assets/templates`*
    - configure angular-rails-templates.      
``` sh config/application.rb                 
     config.angular_templates.module_name    = 'myTemplates'
     config.angular_templates.ignore_prefix  = 'templates/'
     config.angular_templates.markups        = %w(erb haml)       
```
   
Yeah..! We have done with the set up. Now lets get into action.
#### A Book Model
Let us take the example of a Book resource and use RESTful API to manipulate its state. Consider the attributes ISBN, Name and Damaged(Boolean indicating whether the book is damaged or not).
  <br/>
#####Generate ruby model and controller.

```ruby Generating Ruby Model and Controller
$ rails g model book isbn:string name:string damaged:boolean
$ rails g controller books
```
#####Configure Book at rails routes

```ruby config/routes.rb
resources :books        
```

#####Include required javasript libraries 

``` javascript app/assets/javascripts/application.js
//= require angular
//= require angular-resource
//= require angular-route.min
//= require angular-rails-templates
//= require ng_initialize
//= require_tree ../templates
//= require_tree .
    
```
 `require_tree ../templates` loads all the haml files inside its `templates` directory and its sub directories.

######*Note: Angular.js library should be loaded before its dependent libraries are loaded.*

####Bootstraping Angular App
We can bootstrap angular app in two ways. *Automatic initialization* and *Manual initialization*
#####Automatic Initialization
``` haml app/view/layaouts/application.html.haml
     !!!
     %html{:ng_app => 'myApp'}    
```
 1. `ng_app` on a html element bootstraps angular engine automatically.
 2.  All the siblings of `html` tag will be under the scope of `ng_app`.   
 3. When the angular.js library is loaded, it will search for the `:ng_app` attribute on any of the DOM element and [bootstraps](https://docs.angularjs.org/guide/bootstrap) it.

##### Manual Initialization
  We can get more control on angular module initilization using [manual bootstrap](https://docs.angularjs.org/guide/bootstrap)
``` js
angular.module('myApp', [])
      .controller('MyController', ['$scope', function ($scope) {
        $scope.greetMe = 'World';
      }]);
angular.element(document).ready(function() {
      angular.bootstrap(document, ['myApp']);
    });
```
 We will go with automatic initialization in this post.
            
#### Inject dependent modules into Angular application

``` js app/assets/javascripts/ng_initialize.js
var myApp = angular.module("myApp", ['ngResource', 'ngRoute', 'myTemplates'])
```
 1. `angularjs-rails` gem comes with `ng_resource`. But we need to inject into it explicitly. 
 2. `ngRoute` is used for angular routes.
 3. `myTemplates` is `module_name` of rails angular templates, which was configured at *application.rb*.

####Define Resource

*Group [services/factories](https://docs.angularjs.org/guide/providers) into a services directory.*


``` js app/assets/javascripts/services/book.js
(function(angular, app) {
    "use strict";
    app.service("Book",["$resource", function($resource) {
        return $resource('/books/:id.json', {id:'@id'})
    }]);
})(angular, myApp);

```
  The URL of the resource has an attribute `id` prefixed with `:`. It acts as a request parameter and this value will be replaced with respective attribute of the parameter hash.

 1. If the parameter hash is `{id: 2}` then the url becomes `/books/2.json`.
 2. If the parameter hash is `{id:2, name: 'OS'}` then the url becomes `/books/2.json?name=OS`. Here the extra keys in hash will be added as parameters to the url after `?`
 3. `@` symbol on any attribute will fetch the corresponding values from its data. If a *book* object is having the state as `{id: 2, name: 'OS', isbn:'121-340121'}` and the resource is `$resource('/books/:id.json', {id:'@id', isbn:'@isbn'})`. Here {id} and {isbn} would be respctive attributes of *book*.
 4. The sufix of the url is a response format(`.json`). We can also make this format dynamic by changing the resource to `('/books/:id.:json', {id:'@id', json: '@json'})`
<br>
*We define an anonymous [module](http://www.adequatelygood.com/JavaScript-Module-Pattern-In-Depth.html) and pass global variables `angular` and `myApp` to it, so that global scope can be avoided* 
######*Note: Make sure that initilization of `myApp` has been done before loading this service.*
*$resource* provides the following functions and their respective request types
 
``` js
{ 'get':    {method:'GET'}, 
  'save':   {method:'POST'},
  'query':  {method:'GET', isArray:true},
  'remove': {method:'DELETE'},
  'delete': {method:'DELETE'} 
};
```
 1. `Book.get({id: 1})` method makes a `GET /books/1.json` request.
 2. `Book.query()` method makes a `GET /books.json` request.       
 3. `true` value of `isArray` indicates response data is in the form of Array.
 4. `false` value of `isArray` indicates response data is in the form of Hash.         

We can call `$method()` on any object(book.$save(), book.$delete()..etc). These `$method()s` returns the [promises](https://docs.angularjs.org/api/ng/service/$q).

####Books Listing :

#####Definie `index` action in `books_controller.rb`.

``` ruby app/controllers/books_controller.rb
class BooksController < ApplicationController
  def index
    respond_to do |format|
      format.json do
        render :json => Book.all
      end
      format.html
    end
  end
end    
```
This index action gives an array of books as json data.

#####Define angular route to navigate to the listing page.

``` javascript app/assets/javascripts/router.js
(function(angular, app) {
    "use strict";
    app.config(["$routeProvider", function($routeProvider) {
        $routeProvider
            .when('/books', {
                templateUrl: 'books/index.html',
                resolve: {
                    books: function(Book){
                        return Book.query()
                    }
                },
                controller: "BooksIndexController"
            })
    }]);
})(angular, myApp);

```
1. [$routeProvider](https://docs.angularjs.org/api/ngRoute/provider/$routeProvider) is used to create an angular routes.
2. `when` adds a new route to route service
3. `templateUrl` is the path to the file to be rendered after action. Angular templates engine adds `templates/` to `books/index.html` and the final file path will be `templates/books/index.html`.
4. [`resolve`](https://docs.angularjs.org/api/ngRoute/provider/$routeProvider) executes before the initialization of *BooksIndexController*. The returned object can be injected into controller as a dependency.
  <br/>
`Book.query()` makes a get request, resolves the promise and returns all the `books`.

``` javascript app/assets/javascripts/controllers/books_controller.js
(function(angular, app) {
    "use strict";
    app.controller("BooksIndexController",["books", "$scope", function(books, $scope) {
        $scope.books = books;
    }]);                     
})(angular, myApp);
```
`books` returned from resolve of route is injected into to the controller scope.

{% raw %}
``` haml app/assets/templetes/books/index.html.haml
  %a{:href => "#/new", :class => "btn btn-primary btn-sm"} New Book
  
  %table.table.table-condensed.table-bordered
    %thead
      %th ISBN
      %th Name
      %th Is Damaged
      %th
    %tbody
      %tr{:ng_repeat => "book in books"}
        %td {{book.isbn}}
        %td {{book.name}}
        %td {{book.damaged}}
        %td 
          %a{:href => "#/{{book.id}}/edit", :class => "btn btn-warning btn-sm "} Edit
          %a{:href => "", :ng_click => "destroy(book)", :class => "btn btn-danger btn-sm "} Delete
          %a{:href => "", :ng_click => "markAsDamaged(book)", :class => "btn btn-info btn-sm" , :ng_hide => "book.damaged"} Marks as damaged

```
{% endraw %}

####Define New Book and Save
Write `create` action in `books_controller.rb` 
``` ruby app/controllers/books_controller.rb
class BooksController < ApplicationController
  def create
    respond_to do |format|
      format.json do
        book = Book.new(book_params)
        render :json => book.save
      end
  end
end    
```
The Angular router needs to be updated now
``` javascript app/assets/javascripts/router.js
(function(angular, app) {
    "use strict";
    app.config(["$routeProvider", "$locationProvider", function($routeProvider, $locationProvider) {
        $routeProvider
            .when('/books', {
                 ......
            })
            .when('/new', {
                templateUrl: 'books/new.html',
                resolve: {
                    newBook: function(Book){
                        return new Book({isbn: null , name: null, damaged: false})
                    }
                },
                controller: "BooksNewController"
            })
    }]);
})(angular, myApp);
```
`#/new` route creates an empty book object and passes it to the *BooksNewController*

``` javascript app/assets/javascripts/controllers/books_controller.js
(function(angular, app) {
    "use strict";
    ......    
    app.controller("BooksNewController",["$scope", "$location", "book", function($scope, $location, book) {
        $scope.book = book
    }]);            
    
})(angular, myApp);

```
{% raw %}

Here is the form defining creation of a new book 

``` haml app/assets/templetes/books/new.html.haml
%form{:method => "post", :class => "form-horizantal", :ng_submit => "submit()"}
  .form-groupe
    %label.control-label ISBN
    %input{:type => 'text', :ng_model => "book.isbn"}
  .form-groupe
    %label.control-label Name
    %input{:type => 'text', :ng_model => "book.name"}
  .form-groupe
    %button{:type => "submit"} Save
```
{% endraw %}
 1. new.html.haml is under the *BooksNewController*'s scope
 2. All the book attributes are bounded with `ng_model` of form inputs. When we submit the form it invokes `submit()` of `ng_submit` attribute.
   Let us add submit function in *BooksNewController*
``` javascript app/assets/javascripts/controllers/books_controller.js
(function(angular, app) {
    "use strict";
    ......
    app.controller("BooksNewController",["$scope", "$location", "book", function($scope, $location, book) {
        $scope.book = book
        $scope.submit = function(){
            $scope.book.$save()
                .then(function(value) {
                    $location.path("/books")
                }, function(reason) {
                    // handle failure
                });
        }
    }]);
    
})(angular, myApp);
```
{% raw %}
On successfull creation of book, [$location](https://docs.angularjs.org/api/ng/service/$location) redirects to `/books` route.          
####Edit and Update Book
Let us update `books_controller.rb` to include functionality to update a book 

``` ruby app/controllers/books_controller.rb
class BooksController < ApplicationController
  def show
    respond_to do |format|
      format.json do
        book = Book.find(params[:id])
        render :json => book
      end
    end
  end

  def update
    respond_to do |format|
      format.json do
        book = Book.find(params[:id])
        render :json => book.update(book_params)
      end
    end
  end
end
```
#### Define *update* action in *resource*
Default actions of `$resource` will not include Update action. We have to configure it.
``` javascript app/assets/javascripts/services/book.js
(function(angular, app) {
    "use strict";
    app.service("Book",["$resource", function($resource) {
        return $resource('/books/:id.json', {id:'@id'},
                         {
                             "update": { method: "PUT"}
                         }
                        );
    }]);
})(angular, myApp);
```

####Update angular router for edit action
   
``` javascript app/assets/javascripts/router.js
(function(angular, app) {
    "use strict";
    app.config(["$routeProvider", "$locationProvider", function($routeProvider, $locationProvider) {
        $routeProvider
            .when('/books', {
                 ......
            })
            .when('/new', {
                 ......            
            })
            .when('/:id/edit', {
                templateUrl: 'books/new.html',
                resolve: {
                    book: function(Book, $route){
                        return Book.get({id: $route.current.params.id})
                    }
                },
                controller: "BooksEditController"
            })
    }]);
})(angular, myApp);

```
1. Get request parameters using [`$route`](https://docs.angularjs.org/api/ngRoute/service/$route).
2. Use `$route.current.params`  instead of [$routeParams](https://docs.angularjs.org/api/ngRoute/service/$routeParams) to access the new current route parameters.
3. `Book.get({id: $route.current.params.id})` makes the GET request with url like "/books/1.json". 
4. Now the *Book* state is `"book"=>{"id"=>"1", "isbn"=>"81-7596-067-1", "name"=>"Old Book Name", "damaged"=>false}`

#### Handle update in angular controller
``` javascript app/assets/javascripts/controllers/books_controller.js
(function(angular, app) {
    "use strict";
    ......
    app.controller("BooksEditController",["book", "$scope","$location", function(book, $scope, $location) {
        $scope.book = book
        $scope.submit = function(){
            $scope.book.$update()
                .then(function(value) {
                    $location.path("/books")
                }, function(reason) {
                    // handle failure
                });
        }
    }]);
    
})(angular, myApp);

```
1. The submitted form here calls the `$update()` function on book.
2. It makes the PUT `/books/1.json` and params will be `"book"=>{"id"=>"1", "isbn"=>"81-7596-067-1", "name"=>"New Book Name", "damaged"=>false}`

###Delete Book
Since all the books are rendered under the  *BooksIndexController* scope, let us define `destroy()` method in the same controller.
``` javascript app/assets/javascripts/controllers/books_controller.js
(function(angular, app) {
    "use strict";
    app.controller("BooksIndexController",["books", "$scope", "$route",function(books, $scope, $route) {
        $scope.books = books;
        $scope.destroy = function(book){
            if(confirm("Are You Sure?")){
                book.$delete()
                    .then(function(response){
                        $route.reload();
                    }, function(reason){
                       // handle failure   
                    });
            }
        }
    }]);
})(angular, myApp);

```
 1. Call <code>$delete()</code> function on a book to destroy it. It creates DELETE `/books/1.json` request
 2. If the book is deleted successfully, `$route.reload()` reloads the current page.

###Custom Actions

We have used CRUD action on the resource so far. We can also make custom calls on the resource
``` javascript app/assets/javascripts/services/book.js
(function(angular, app) {
    "use strict";
    app.service("Book",["$resource", function($resource) {
        return $resource('/books/:id.json', {id:'@id'},
                         {
                             "update": { method: "PUT"},
                             "markAsDamaged":{url: "/books/:id/mark_as_damaged.json", method: "PUT"}
                             "damagedBooks":{url: "/books/damaged_books.json", method: "GET", isArray: true}
                         }
                        );
    }]);
})(angular, myApp);
```
These custom actions needs complete Url. 

1. `markAsDamaged` is called on book object to changes its status to damaged. `book.$markAsDamaged()` create a `PUT` request and url as `/books/1/mark_as_damaged.json` 
2. `damagedBooks` gets an array of damaged books. Book.damaged_books() makes a `GET` request and url as `books/damaged_books.json`

####Configure custom action in routes.rb
```ruby config/routes.rb
  resources :books do
    member do
      put "mark_as_damaged"
    end
    collection do
      get "damaged_books"
    end
  end
```
update books_controller.rb

``` ruby app/controllers/books_controller.rb
class BooksController < ApplicationController
  def mark_as_damaged
    respond_to do |format|
      format.json do
        book = Book.find(params[:id])
        book.update(:damaged => true)
        render :json => book
      end
    end
  end
end
```

Define `markAsDamaged()` function in *BooksIndexController*
``` javascript app/assets/javascripts/controllers/books_controller.js
(function(angular, app) {
    "use strict";
    app.controller("BooksIndexController",["books", "$scope", "$location",function(books, $scope, $location) {
        $scope.books = books;

        $scope.destroy = function(book){
                 .......      
        }

        $scope.markAsDamaged = function(book){
            book.$markAsDamaged()
                .then(function(response){
                    book = response.data;
                }, function(reason){
                   // handle error
                });
        }
        
    }]);
})(angular, myApp);
```

I hope by now you would know how to define `damaged_books()` handler in the *BooksIndexController*.
<br/>

You can find this git hub repo at [ng_resouce_with_rails](https://github.com/sri-sankl/ng_resouce_with_rails)
>Thank you for reading the post. Feel free to comment.
>Happy Coding..:)
