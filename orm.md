Week 3 Day 2 - ORMs
================

## Introduction

Working directly with SQL in our Ruby code is cumbersome and annoying.
Why? Because in languages like Ruby, we like to works with classes, instances, and their attributes; not tables, rows and columns.

For this reason, many frameworks provide what's called an "ORM", an Object-Relational-Mapping.

Wikipedia has a good [overview of ORMs](http://en.wikipedia.org/wiki/Object-relational_mapping) (read the intro and overview sections only).

The goal in this assignment is to build an mini-ORM of our own, before we use "ActiveRecord", the ORM of choice for Ruby / Rails developers.

## Our Goal

It would definitely be nice to store the contacts for our recently created app onto the disk, instead of just in an array in the program's runtime memory. This way, the contacts won't disappear when you quit the program.

Instead of storing the contacts in CSV (which is a bit of work to implement and yet not very flexible), we should use an RDBMS (Relational Database Management System) like PG (Postgres) to keep track of our contacts.

## Contact class Interface

Given that we already have our app laid out in an OO way, and have a Contact class responsible for managing the contacts, we should ideally just be able to modify the Contact class to talk to the `contacts` table in our database, instead of to using the `@@contacts` array in memory to store the contacts.

Below is a list of methods that should ideally be implemented (_`#` implies instance method and `.` implies class method._):

### `Contact.new(firstname, lastname, email)`

The constructor / initializer. Used to represent a contact instance in memory. Does not talk to the database.

### `Contact#save`

Either inserts or or updates a row in the database, as necessary for the given instance of contact.

_Ask yourself / discuss:_ When `save` is called, how will it know whether to run an `INSERT` or `UPDATE` SQL statement?

### `Contact#destroy`

Executes a `DELETE` SQL command against the database.

_Ask yourself / discuss:_ What will it need to provide the database as part of the `DELETE` SQL statement?

### `Contact.find(id)`

A class method to `SELECT` a contact row from the database by `id` and return a `Contact` instance that represents ("maps to") that row.

### `Contact.find_all_by_lastname(name)`

Another class method, but this one returns an array of all contacts that have the provided last name. If none are found, an empty array should be returned.

It will do an exact string match.

### `Contact.find_all_by_firstname(name)`

Same as `Contact.find_all_by_lastname(name)` but for last name instead.

### `Contact.find_by_email(email)`

Since emails are assumed to be unique, we return only a single record (or `nil`) here. Hence why we use `find_by_` instead of `find_all_by` for this method name.

## Code Walkthrough

Here's an example walkthrough of how the Contact class would be used and how the methods would interact with the database.


### Creating a new record

    contact = Contact.new("Khurram", "Virani", "kv@gmail.com")
    contact.save

`save` would trigger an `INSERT` SQL statement to be sent to the database to add the contact into the the `contacts` table in our postgres database:

    INSERT INTO contacts (firstname, lastname, email)
      VALUES ('Khurram', 'Virani', 'kv@gmail.com')

The record would get created in PG and the resulting `id` column (the "Primary Key" value for the row) would be auto-assigned by the database. The ORM should then store the id in the instance variable `@id`, for later use (eg: to update the record).

    contact.id # => 5

### Updating a record

Continuing on with our example use of the ORM in Ruby land:

    contact.firstname = "K"
    contact.lastname = "V"
    contact.save

Since this contact was already created, the `save` method will know _not to `INSERT` this contact but rather `UPDATE` it_ in the database:

    UPDATE contacts SET firstname='K', lastname='V', email='kv@gmail.com'
      WHERE id=5

The `id=5` part tells the DB which contact to update, and is thus quite crucial (otherwise all the records in the `contacts` table would get updated). Our Contact class (ORM) should use the `@id` previously stored for creating this WHERE clause.

### Loading a record by ID

Later on, if the _class method_ `Contact.find` is used, the ORM will simply need to `SELECT` that record from the DB and create an instance of a `Contact` that represents that row for us, as such:

    same_contact = Contact.find(5)
    puts same_contact.firstname # => 'K'
    puts same_contact.lastname  # => 'V'
    puts same_contact.email # => 'kv@gmail.com'

The `.find` method will perform the following `SELECT` to retrieve the record from the DB and create a new instance of `Contact` with the necessary information to return back to us.

    SELECT c.id, c.firstname, c.lastname, c.email
      FROM contacts AS c
      WHERE c.id = 5

Nice, this can power our `show` action in the REPL. When a user asks to see contact #5, our `Application` can use the `Contact.find` to retrieve the contact!

### Searching records

Using one of the `find_by` class methods, we can also easily search for contacts like so:

    contacts = Contact.find_all_by_lastname('Virani')
    contacts.each do |c|
      puts c
    end

Note how the `find_by` method returns an array instead of a single instance. Ask yourself, _why would we build it this way?_

### Destroying a record

The ORM will also provide us with a `destroy` instance method to `DELETE` that contact row from the contacts table:

    same_contact.destroy

This will execute a SQL statement such as this:

    DELETE FROM contacts WHERE id = 5

The record will be deleted from the database but the object instance will remain in memory in Ruby. This is because method call to an object cannot destroy the object it is called on from memory.

_Note:_ Since `same_contact` will no longer point to a valid, existing record in the database, using ORM methods like `save` and `destroy` (again) will likely cause a postgres error/exception. That's okay for now (we can prevent this by throwing our own exception to the caller instead of attempting to execute an invalid query; but let's leave that for "Bonus")

Attempting to find contact with id 5 now will naturally not yield a contact (since it was just deleted):

    Contact.find(5) # => nil
