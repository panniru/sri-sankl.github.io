---
layout: post
title: "Metaprogramming In Ruby: Implementing Class Extension Mixins"
date: 2016-01-26 19:38:53 +0530
comments: true
categories: 
---
> Metaprogramming: Code that writes code
 

##### Problem statement: 
  In Indian payroll system, many compements are based on the `basic`. These components varies from company to company. Below `payslip_breakups` table shows the different components and their value from `basic` in percentages. 

| component_code              |  criteria     |
|-----------------------------|:-------------:|
|    hra                      |     50        |          
| city_compensatory_allowance |     10        |          
| employer_pf_contribution    |     12        |          
| bonus_payment               |     15        |          
   
 These components are dynamic and has to be evaluated at run time.

<!--more--> 
##### Hook Methods:
 Whenever we write code in the object oriented programming, we will face several situations such as inheriting classes, mixing modules into classes and defining, undefining and removing methods. Metaprogramming helps us to define some methods to handle these events and do some workaround when these events fired. These methods are known as the **Hook Methods**. 


``` ruby payslip_breakup_mixin.rb
module PayslipBreakupMixin
  def self.included(base)
    puts "PayslipBreakupMixin was mixed into #{base}"
  end   
end
```
``` ruby payslip.rb
class Payslip
  include PayslipBreakupMixin
end
```
 `included()` is the hook method which will be invoked when the module is included in other class and it will accept the including class as it's parameter. We can override this method and implement our own logic.
<br/>
Eg: `extend_object()`, `method_added()`, `method_removed()`, `method_undefined` are some hook methods

##### Implementing hook method and using class macros

``` ruby  payslip_breakup_mixin.rb
module PayslipBreakupMixin
  attr_reader :basic
  def self.included(base)
    base.extend(ClassMethods)
  end   
  
  module ClassMethods
    def attr_on_basic(*params)
      params.each do |param|
        define_method param.to_sym do
          ((component_criterias[param.to_sym]/100)*basic.to_i)
        end
      end
    end

    private
    def component_criterias
      @component_criterias ||= PayslipBreakUp.belongs_to_salary.map{|break_up| [break_up.component_code.to_sym, break_up.criteria] }.to_h
    end
  end  
end
```
 In the above mixin, `included()` method will be called by Ruby, when an inclusor(which included this module into it) includes the mixin. The `extend()` method will add the `ClassMethods` module's methods into the inclusor's Eigen class. 
<br>
Eigen class is known as Object's own class. Every object has its own class and class itself is an object of the class `Class`. 

###### ClassMethods Explained
 1. `attr_on_basic()` method accepts parameteters to which we need to calculate the value based on basic. 
 2. `define_method()` creates a dynamic method for each parameter.
 3. `component_criterias()` fetches all the breakup component criterias and creates a map.
 4. `basic` should be an instance valriable in the included class and it can be red by it's accessor.

``` ruby payslip.rb
class Payslip
  include PayslipBreakupMixin
end
```

When we include the `PayslipBreakupMixin` into the payslip class. These events will be triggered:

 1. Ruby calls a Hook Method: the `included()` method
 2. The hook turns back to the including class (which is sometimes called the inclusor, or the base in this case) and extends it with the ClassMethods module.
 3. The `extend()` method includes the methods from ClassMethods in the inclusor's eigenclass.

##### Class Macros

``` ruby payslip.rb
class Payslip
  attr_accessor :basic
  include PayslipBreakupMixin
  attr_on_basic :hra, :city_compensatory_allowance, :employer_pf_contribution, :bonus_payment 
  
  def initialize(basic)
    @basic = basic
  end
end
```

  **Class macros** are the class level methods which creates some dynamic code for the given attributes

 1. `attr_on_basic()` is a class macro and it takes some arguments and defines the given methods in it's respected eagen class.
 2. `attr_accessor()`, `attr_reader()`, `attr_writer()` etc.. are the class macros provided by ruby.  

 `attr_on_basic()` method defines a method for each `:hra`, `:city_compensatory_allowance`, `:employer_pf_contribution`, `:bonus_payment` component. These component definations will fetch their respected value percentages from  `payslip_breakups` model and it will calculate it's absolute value by using `basic` and will return the final result.

>Thank you for reading the post. Feel free to comment.
>Happy Coding..:)

######References:
 1. Metaprogramming Ruby-Paolo Perrotta
