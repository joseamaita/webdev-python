# Flask

## Overview

Flask is a micro framework. *Micro* refers to the small core of the 
framework, not the ability to create single-file applications. Flask 
basically provides routing and templating, wrapped around a few 
configuration conventions. Its objective is to be flexible and allow the 
user to pick the tools that are best for their project. It provides many 
hooks for customization and extensions.

Flask curiously started as an April Fool's joke, but it's in fact a very
serious framework, heavily tested and extensively documented. It 
features integrated unit testing support and includes a development 
server with a powerful debugger that lets you examine values and step 
through the code using the browser.

Flask is unicode-based and supports the Jinja2 templating engine, which 
is one of the most popular for Python web applications. Though it can be 
used with other template systems, Flask takes advantage of Jinja2's 
unique features, so it's really not advisable to do so.

Flask's routing system is very well-suited for RESTful request 
dispatching, which is really a fancy name for allowing specific routes 
for specific HTTP verbs (methods). This is very useful for building APIs 
and web services.

Other Flask features include sessions with secure cookies, pluggable 
views, and signals (for notifications and subscriptions to them). Flask 
also uses the concept of *blueprints* for making application components.

## Literature

These books will help you to know more about Flask:

* Flask Web Development (O'Reilly - Miguel Grinberg)
* Building Web Applications with Flask (Packt Publishing - Italo Maia)
* Learning Flask Framework (Packt Publishing - Matt Copperwaite, Charles 
Leifer)
* Flask Framework Cookbook (Packt Publishing - Shalabh Aggarwal)
* Instant Flask Web Development (Packt Publishing - Ron DuPlain)
* Flask Blueprints (Packt Publishing - Joël Perras)
