---
layout: post
title: "Ajax Pagination With Angularjs and Rails using will_paginate"
date: 2014-11-29 12:59:40 +0530
comments: true
categories: 
---
I use pagination when I need to list a large number of records. In rails, we use [Will Paginate](https://github.com/mislav/will_paginate) gem for accomplishing pagination. While dealing with pagination in a single page application(SPA)s, `will_paginate` along with the angularjs makes the implementations simpler and faster. AngularJS provides a powerful component called [Directives](https://docs.angularjs.org/guide/directive) which can be used for implementation of reusable pagination component.<br>
  <p>In this post we will implement a dynamic, reusable pagination component, which is implemented using will_paginate and angular_directives.</p> 

##### Preview:
   Let us consider a Books listing page. There are a total of 14 books and the screen displays 3 books per page.

![pagination](/images/books_1.png)
![pagination](/images/books_2.png)
![pagination](/images/books_3.png)
</p>
   The circled component is the pagination directive which we are going to implement.     
<!--more--> 
#####Gems used:

``` ruby Gemfile  
  gem 'haml-rails', '~> 0.5.3'
  gem 'angularjs-rails', '~> 1.2.22'
  gem 'angular-rails-templates'  
  gem 'will_paginate', '~> 3.0.7'
  gem 'font-awesome-rails', '~> 4.2.0.0'
```
 1. `will_paginate` is the rails gem for pagiantion
 2. `font-awesome-rails` is rails gem which wraps the beutiful [Font Awesome](http://fortawesome.github.io/Font-Awesome) icons library.
 3. Rest of the gems are discussed in the previous [post](http://sri-sankl.github.io/blog/2014/11/05/connecting-angularjs-resource-with-rails/) 
</p>
Now do `bundle install`<br>

#### Implementation:
 Get the json list of books along with the pagination properties.     
``` ruby app/controllers/books_controller.rb
class BooksController < ApplicationController
  def index
    respond_to do |format|
      format.json do
        page = params[:page].present? ? params[:page] : 1 
        @books = Book.all.paginate(:page => page, :per_page => 5)
        render :json => Paginator.pagination_attributes(@books).merge!(:books => @books)
      end
      format.html
    end
  end
end
```
  1. `paginate()` method injects pagination properties into the result.
  2. `paginate()` method accepts pagination options. `page` is the required attribute and others are optional. 
  3. We define a `Paginator` class whose responsibility is to collect pagination attributes from the given object and returns a hash that contains *current_page, start_index, to_index and total_pages*.

``` ruby lib/paginator.rb
class Paginator
  class << self
    def pagination_attributes(source_obj, data_hash = {})
      data_hash[:total_entries] = source_obj.total_entries
      previous_page = source_obj.previous_page.present? ? source_obj.previous_page : 0
      data_hash[:current_page] = previous_page+1
      data_hash[:to_index] = (data_hash[:current_page] * source_obj.per_page) > source_ obj.total_entries ? source_obj.total_entries : (data_hash[:current_page] * source_obj.per_page)
      data_hash[:from_index] = (previous_page * source_obj.per_page)+1
      data_hash
    end
  end
end
```
<b> *Note:* </b>Since paginator.rb is located in `/lib` directory, we have to load this directory
``` ruby `config/application.rb` 
config.autoload_paths += %W(#{Rails.root}/lib/)
```                                        

#####Bootstrap Angular Application
``` js app/assets/javascripts/ng_initialize.js
var myApp = angular.module("myApp", ['ngResource', 'myTemplates'])
```
  Configuring angularjs and its dependent modules are discussed in [connecting angularjs with rails](http://sri-sankl.github.io/blog/2014/11/05/connecting-angularjs-resource-with-rails/)
<br/>
Write an angular Book service which fetches the list of books
``` js app/assets/services/book.js
(function(angular, app) {
    "use strict";
    app.service("bookService",["$http", function($http) {
        var books = function(page){
            return $http.get("/books.json", {params: {page: page }})
        }
            
        return {
            books : books
        }
    }]);
})(angular, myApp);
```
 
##### Define Angular Controller        
``` js app/controllers/books_controller.js
(function(angular, app) {
    "use strict";
    app.controller("BooksController",["bookService", "$scope",function(bookService, $scope) {
        $scope.getBooks = function(page){
            bookService.books(page)
                .then(function(response){
                    $scope.books = response.data.books
                    $scope.from_index = response.data.from_index
                    $scope.to_index = response.data.to_index                     
                    $scope.total_entries = response.data.total_entries;
                    $scope.current_page = parseInt(response.data.current_page)
                })
        }
    }])
})(angular, myApp);
```
 1. Inject book service into the controller
 2. `getBooks` method queries the `bookService` with the required `page` number.
 3. `books` data as well as the pagination attributes are made available in the scope of `BooksController`
 4. `from_index`, `to_index`, `total_entries`, `current_page` are made to available in its scope

#### Define a reusable directive
##### Directive Usage
``` haml app/views/books/index.html.haml
.col-md-4{:ng_controller => "BooksController", :ng_init => "getBooks(1)"}
  %my-pagination{:from => "from_index", :to => "to_index", :current_page => "current_page", :total=> "total_entries", :action => "getBooks(page)"}
```
  1. The custom element `my-pagination` is a directive and it has the attributes *from, to, current_page, total*.
  2. `from` is assigned with `from_index`, a scoped variable which is under `Bookscontroller`.
  3. `to` is assigned with `to_index`.
  4. `current_page` is assigned with `current_page`.
  5. `total` is assigned with `total_entries`.
  6. `ation` is reffered to the `getBooks()` method, which is defined in `Bookscontroller`.

##### Directive Defination
``` js app/assets/javascripts/directives/pagination.js
(function(angular, app) {
    "use strict";
    app.directive("myPagination", function() {
	return {
	    restrict: 'E',
	    scope: {
		from: '=',
		to: '=',
		total: '=',
                currentPage: '=',
		action: '&'
	    },
	    controller: ["$scope", function($scope){
		$scope.previousPage = function(){
		    $scope.currentPage -= 1
		    $scope.action({page: $scope.currentPage})
		}
		$scope.nextPage = function(){
		    $scope.currentPage += 1
		    $scope.action({page: $scope.currentPage})
		}
	    }],
	    templateUrl: "paginationElements.html"
	}
    });
})(angular, myApp);
```
 1. The element `my_pagination` maps to `myPagination` directive. This mapping has been done by angular [normalization](https://docs.angularjs.org/guide/directive) 
 2. Our directive is restricted to element by defining `restrict` with `E`.
 3. <b>*Scope*</b> creates an *Isolate Scope* to this directive so that we can reuse it any where.
    1. `from: '='` is similar to `from : = from`. It states that `from` property of directive scope is bind with the value assigned to the `from` attribute of `my_pagination` element.
    2. `action: '&'` is similar to `action: &action`. `&` binding is used evalute the expression or function defined on `action` attribute of `my_pagination` element.
 4. <b>*Controller*</b>: Isolated Scoped attributes can access and manipulated by its own controller. 
    1. `previousPage()` method is to show the previous page. It decreases `currentPage` by 1 and invokes the `action` attributes value. `{page: $scope.currentPage}` is passed as the page parameter to the `getBooks(page)` function. 
    2. `nextPage()` method is to show the previous page. It increases `currentPage` by 1 and invokes the `action` attributes value.

 5. <b>*TemplateUrl*</b>: It is path to the html content.
   Here is the code                
{% raw %}       
``` haml app/assets/templates/paginationElements.html.haml
%span
  {{from}} - {{to}} of {{total}}
.btn-group.btn-group-sm
  %button.btn.btn-default{:ng_click => "previousPage()", :ng_class => "{'disabled': from == 1}"}
    %i.fa.fa-chevron-left.fa-lg
  %button.btn.btn-default{:ng_click => "nextPage()", :ng_class => "{'disabled': to == total}"}
    %i.fa.fa-chevron-right.fa-lg
```
{% endraw %}

#####The final index view is
``` haml app/views/books/index.html.haml
.col-md-4{:ng_controller => "BooksController", :ng_init => "getBooks(1)"}
  %h3
    Books
    .pull-right
      %my-pagination{:from => "from_index", :to => "to_index", :current_page => "current_page", :total=> "total_entries", :action => "getBooks(page)"}
  %table.table.table-condensed.table-bordered
    %thead
      %th S.No
      %th ISBN
      %th Name
    %tbody
      %tr{:ng_repeat => "book in books"}
        %td {{from_index+$index}}
        %td {{book.isbn}}
        %td {{book.name}}
```
>Thank you for reading the post. Feel free to comment.
>Happy Coding..:)

        