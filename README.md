# Testing dynamic pages with Ruby, Page-Object and Cucumber - An simple strategy

### Introduction
Sometime ago I had to do some BDD on a project which involved some pages with dynamic content, the HTML element on the pages could change with each visit on the page.I found a simple solution using Ruby and Page Object gem. I have put it here in case some one else finds it useful. For example a page that presents varying HTML text input elements on each page visit. As an automation tester you want to test the dynamic element by evaluating the page at run time and take appropriate actions. We will use ruby to come to our rescue, in particular we will use the capability of ruby to add methods to a class dynamically using ```class eval```. We will be aided on the way by the “page-object” gem which helps to dynamically build our pages and follow the Page Object design pattern. Later we will write a cucumber step that will help us test the page using BDD. The same strategy can be used for any type of HTML element.

### Example Page under test

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <body>
    <form action="/payment/confirm" method="post" id="confirm_form">
      <label name="question">
        <strong>Please enter the 2nd &amp; 7th characters of your first car</strong>
      </label>
      <div class="row" name="answer_row">
        <input type="password" name="car__1" maxlength="1" class="character1" required autocomplete="off">
        <input type="password" name="car__2" maxlength="1" class="character1" required autocomplete="off">
      </div>
      <label name="question">
        <strong>Please enter the 9th &amp; 11th characters of your mothers first name</strong>
      </label>
      <div class="row" name="answer_row">
        <input type="password" name="mother__9" maxlength="1" class="character1" required autocomplete="off">
        <input type="password" name="mother__11" maxlength="1" class="character1" required autocomplete="off">
      </div>
      <label name="question">
        <strong>Please enter the 6th &amp; 9th characters of your favourite book</strong>
      </label>
      <div class="row" name="answer_row">
        <input type="password" name="book__6" maxlength="1" class="character1" required autocomplete="off">
        <input type="password" name="book__9" maxlength="1" class="character1" required autocomplete="off">
      </div>
      <button type="submit" id="process_payment">Process payment</button>
    </form>
  </body>
<head>

```
The above page may be part of a secure card payment page that requires the user to answer some questions before the payment can be authorised. The page presents a form with 6 text boxes that allow for user to enter single character. The text boxes have names generated dynamically. For example there may be a question about the person’s place of birth asked, in which case we might have input fields with names birth__3 and birth__8 presented. The challange for the automation tester is to drive the test code to either query of populate these dynamic text boxes.

### Design considerations
As you may have noticed in the simple example above that I have provided enough test hooks in the form of “name” and “id” attributes to make our life easier ahead, this is one of the keys to producing testable pages and code (provide enough test hooks). Developers and testers needs to be considerate to each other!

### The Ruby page object
We are now going to translate the above page into a page object in ruby.

```ruby
class MyPage
  include PageObject
  div(:answer_row1, :name => "answer_row", :index => 0)
  div(:answer_row1, :name => "answer_row", :index => 1)
  div(:answer_row1, :name => "answer_row", :index => 2)
  button(:pay, :id => "process_payment")
end
```
The ```include PageObject``` statement includes the PageObject mixin module that includes a lot of helper methods. In our case we have used the div and the button helper methods to correspond to our html elements. 
> The page class obviously lacks the declarations for the input text boxes as we dont know which questions will be presented at run time, 
this is the problem we will try to solve as we progress below.

To further improve the ```MyPage``` class we need to add some helper methods in the class so that the cucumber step definitions become simple to write later. There are 3 div rows in our form and each of them can have 2 text boxes generated within them. We need a helper method that will give us the name of these elements and also dynamically generate page-object calls for these text boxes (using PageObject modules text-field call).

```ruby
class MyPage

  include PageObject

  #
  # Some static content
  #

  div(:answer_row1, :name => "answer_row", :index => 0)
  div(:answer_row1, :name => "answer_row", :index => 1)
  div(:answer_row1, :name => "answer_row", :index => 2)
  button(:pay, :id => "process_payment")

  #
  # Dynamically get name attributes of text inputs and dynamic page-object calls
  #

  def form_dynamic_textinput_names
    question1_element_name = ""
    question2_element_name = ""
    question3_element_name = ""
    dynamic_names = []
    answer_row1_element.text_field_elements.each do |page_input_element|
      question1_element_name = page_input_element.attribute("name")
      dynamic_names << question1_element_name
      class_eval do
        text_field(question1_element_name.to_sym, :name=>question1_element_name)
      end
    end

    answer_row2_element.text_field_elements.each do |page_input_element|
      question2_element_name = page_input_element.attribute("name")
      dynamic_names << question2_element_name
      class_eval do
        text_field(question2_element_name.to_sym, :name=>question2_element_name)
      end
    end

    answer_row3_element.text_field_elements.each do |page_input_element|
     question3_element_name = page_input_element.attribute("name")
      dynamic_names << question3_element_name
      class_eval do
        text_field(question3_element_name.to_sym, :name=>question3_element_name)
      end
    end
    return dynamic_names
  end
end
```

Here we have added the function ```form_dynamic_textinput_names``` that finds  the text input elements that the page has presented to us. It does two main tasks, first it dynamically adds HTML elements by calling ```class_eval``` on the page instance to add text fields and secondly it returns the name attribute of the text input elements. Also worth notice is the use of ```answer_row1_element.text_field_elements``` method call that is provided by page-object to get the nested elements for the div answer_row1 till answer_row3.
To further elaborate the part where we dynamically add methods to our page object class to add text input elements. We have used the below code in a loop for each of the HTML answer rows.
```ruby
    answer_row3_element.text_field_elements.each do |page_input_element|
     question3_element_name = page_input_element.attribute("name")
      dynamic_names << question3_element_name
      class_eval do
        text_field(question3_element_name.to_sym, :name=>question3_element_name)
      end
    end
```
We get the name attribute of each of the text box element and then use page-object gem to add a text field with same same name in the page object class ```MyPage```.

Next we are going to add a method to populate the text fields using the helper we have written just now

```ruby
  class MyPage

    include PageObject

  #
  # Some static content
  #

  div(:answer_row1, :name => "answer_row", :index => 0)
  div(:answer_row1, :name => "answer_row", :index => 1)
  div(:answer_row1, :name => "answer_row", :index => 2)
  button(:pay, :id => "process_payment")

  #
  # Dynamically get name attributes of text inputs and dynamic page-object calls
  #

  def form_dynamic_textinput_names
    question1_element_name = ""
    question2_element_name = ""
    question3_element_name = ""
    dynamic_names = []
    answer_row1_element.text_field_elements.each do |page_input_element|
      question1_element_name = page_input_element.attribute("name")
      dynamic_names << question1_element_name
      class_eval do
        text_field(question1_element_name.to_sym, :name=>question1_element_name)
      end
    end

    answer_row2_element.text_field_elements.each do |page_input_element|
      question2_element_name = page_input_element.attribute("name")
      dynamic_names << question2_element_name
      class_eval do
        text_field(question2_element_name.to_sym, :name=>question2_element_name)
      end
    end

    answer_row3_element.text_field_elements.each do |page_input_element|
      question3_element_name = page_input_element.attribute("name")
      dynamic_names << question3_element_name
      class_eval do
        text_field(question3_element_name.to_sym, :name=>question3_element_name)
      end
    end
    return dynamic_names
  end

  #
  # Send character “X” into each of the text fields
  #
  def answer_questions_correctly
    form_dynamic_textinput_names.each do |dynamic_element_name|
      element_symbol = dynamic_element_name.to_sym
      self.send("#{element_symbol}=", "X")
    end
  end
end
```

We have now added the method ```answer_security_questions_correctly``` which populates the character “X” into each of the text areas when invoked. Notice the use of the self.send method that sends a message to the MyPage class methods identified by ```"#{element_symbol}="``` which are provided by page object method calls. 

### A Cucumber step

A typical cucumber step using the above page object class might look like
```ruby
Then (/^I answer the security questions correctly$/) do
  on(MyPage).answer_questions_correctly
end
```

### References
1.	Page Object gem intro - http://www.cheezyworld.com/2011/07/29/introducing-page-object-gem/
2.	Page object design pattern - http://code.google.com/p/selenium/wiki/PageObjects
3.	Ruby - https://www.ruby-lang.org/en/
4.	Cucumber - http://cukes.info/


