
# Android-Architecture-Components
To introduce the terminology, here is a short introduction to the Architecture Components and how they work together. Each component is explained more in their respective sections.

This diagram shows a basic form of this architecture.
![Basic AAC Diagram](/images/basic_aac_diagram)

*Entity:* When working with Architecture Components, the entity is an annotated class that describes a database table.

*SQLite database:* On the device, data is stored in a SQLite database. The Room persistence library creates and maintains this database for you.

*DAO:* Data access object. A mapping of SQL queries to functions. You used to have to define these queries in a helper class, such as `SQLiteOpenHelper`. When you use a DAO, you call the methods, and the components take care of the rest.

*Room database:* Database layer on top of SQLite database that takes care of mundane tasks that you used to handle with a helper class, such as `SQLiteOpenHelper`. Provides easier local data storage. The Room database uses the DAO to issue queries to the SQLite database.

*Repository:*  A class that you create for managing multiple data sources, for example using the WordRepository class.

*ViewModel:* Provides data to the UI and acts as a communication center between the Repository and the UI. Hides the backend from the UI. `ViewModel` instances survive device configuration changes.

*LiveData:* A data holder class that follows the observer pattern, which means that it can be observed. Always holds/caches latest version of data. Notifies its observers when the data has changed. `LiveData` is lifecycle aware. UI components observe relevant data. `LiveData`
automatically manages stopping and resuming observation, because it's aware of the relevant lifecycle status changes.
