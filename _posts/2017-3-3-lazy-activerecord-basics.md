---
layout: post
title: lazy activerecord basics
categories: ruby
tags: [rails, activerecord, sql]
---

One of the sexiest parts of rails is ActiveRecord. It simplifies database calls, and provides an easy channel for new Rubyist to create sophisticated apps. With great power also comes great responsibility, and unchecked power can lead to issues down the road. Where on the road will depend on your database platform, structure, and how you implemented ActiveRecord calls, but to put it simply... it will happen at scale.

## Example
Let's start off with simple a call to a Domain model.

```ruby
domain = Domains.find(1)
#=> <#ActiveRecord:: Id='1',Name='testdomain.com',Registrar='HopefullyNotGoDaddy',NameServers='ns1.domain.com', ResourceRecords=[]>
```

This is a seemingly benign ActiveRecord request, but let's take a look under the hood. This can be represented in SQL as
```sql
select * from Domains where id='1'
```
What's significant here? The query is being generated with the `*` wildcard. This means that regardless of what we need, it will pull all items from that row. 

So that's fine and well, but let's scale this up! We've just imported our customer's data and added 10,000 domains to our database, and we have to build a page that displays all domains and links each to a details page. You might start off by using a call like this. Let's also assume that the ``ResourceRecord`` values have hundreds of entries saved in them (we will discuss best practices on database relations in another post because I'm lazy.).

```ruby
domains = Domains.all
#=> #<ActiveRecord::Relation [#<Domain id: 1, name: "domain1.com", registrar="HopeFullyNotGoDaddy", resourcerecords: []>, #<Domain id: 2, name: "domain2.com", registrar="HopeFullyNotGoDaddy", resourcerecords: []>, #<Domain id: 3, name: "domains3.com", registrar="HopeFullyNotGoDaddy", resourcerecords: []>, #<Domain id: 4, name: "domain4.com", registrar="HopeFullyNotGoDaddy", resourcerecords: []>, #<Domain id: 5, name: "domains5.com", registrar="HopeFullyNotGoDaddy", resourcerecords: []>, #<Domain id: 6, name: "domain6.com", registrar="HopeFullyNotGoDaddy", resourcerecords: []>, #<Domain id: 7, name: "domain7.com", registrar="HopeFullyNotGoDaddy", resourcerecords: []>, ...]> 
```

It's easy when you're first starting to pull every item for every record that you call, hell, when I'm scaffolding a new app, I do too. You never know what you might want of need in a given controller. This works fine, but refactoring an entire app can be tedious.

## Issue
Where you really start to run into trouble is when you now have thousands of these requests being served up by your database, and a page speed starts crawling.

## Solution
So the more efficient approach is to do record scoping before you build your view. In your controller lay out what items you think you will need. In this example, we only need the domain `id` and `name` for our view.

```ruby
domains = Domain.all.select(:id,:name)
#=> #<ActiveRecord::Relation [#<Domain id: 1, name: "domain1.com">, #<Domain id: 2, name: "domain2.com">, #<Domain id: 3, name: "domains3.com">, #<Domain id: 4, name: "domain4.com">, #<Domain id: 5, name: "domains5.com">, #<Domain id: 6, name: "domain6.com">, #<Domain id: 7, name: "domain7.com">, ...]> 
```

## Why
Getting in the habit of limiting the data returned to only the items you need, will ensure your app is making efficient database calls. Although this example is quite simple, it illustrates the ease in which you can correct this issue, and have one less thing to worry about when performance issues inevitably hit down the road as your database grows.

## Tips
* Check out the `.pluck()` method if you can get away with not generating an ActiveRecord model in your call.
* Open up your development/production logs to see the full SQL queries being generated by ActiveRecord.