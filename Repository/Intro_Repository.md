# The Repository
At the moment, we have our database logic in our Activities and our ViewModels. We could improve our architecture by extracting all that logic to a common place, or Repository. This way, all database-related operations can be handled from a single location, while being accessible from anywhere in our app. This is an example of using the [Single Responsibility Principle](https://en.wikipedia.org/wiki/Single_responsibility_principle).

Using a Repository also adds more flexibility to our architecture. Instead of touching code in multiple places whenever we want to change the database, we can instead isolate the code change to one place using a Repository. 
