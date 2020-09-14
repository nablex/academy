# Data modeling

Data is the most valuable commodity most companies have. However, data is only valuable if you can do something with it.
More specifically if you have 5GB of random bits, you may have a lot of data, but not a lot of use for it, it is therefor worthless.

If you have 5GB of structured data however, you can start drawing conclusions, build new applications that extend upon that data,...

Data modeling is the act of designing the structure of the data. What do you need to capture, how does it relate to other data...

It is usually the most important step when building any application as changes to the data model down the line are much harder to do than changes to business logic, for example you can't retroactively capture data you forgot in your initial design.

## Specificity

The most important part of designing your structured data is to be _specific_. 

If you have an "amount" field indicating the amount of items in an order. Can it (technically) be a string? Sure that can work. All numbers can be displayed as string. However, not all valid strings are valid numbers, so defining it as a number is more specific and therefor the better option.

Do you know for a fact it can not have a decimal part? Make it an integer instead of a double. 
Do you know it can never be negative? Set the minInclusive to 0. 
Are you sure it is always filled in? Make it mandatory. 
And so on and so forth. 

The more specific you can be, the better. This will make your application more predictable and less buggy because there will be fewer edge cases. Additionally the end result is more readable.

An important rule here is: you can always make your data field "less specific" in the future in case you want to support new usecases because the accumulated data so far will definitely match the less strict rule. You can almost never make it "more specific" however as the accumulated data is likely to conflict with the more stringent requirements.

Apart from restrictions on a single field, you can also add guarantees so that your data is coherent. For example if your order is linked to a company, you can ensure that the company actually exists at all times, making it impossible to "accidently" delete and hence invalidate the usefullness of the order data.

The specificity also encompasses defining each and every field. It is at this level that most nosql stores fail dramatically. They generally have very lax requirements on the data definitions which means, at runtime you can have two pieces of data that are ideologically the same (e.g. a company object) but because they were created by different application versions (or different applications alltogether) they don't have the same fields.

This leads to _very_ defensive application code where you have to deal with every eventuality and those that haven't occurred yet.
Relational databases provide very strict requirements of the data, ensuring that data is consistent.

## Storage Guarantees

Ideally you have as much of the specificity at the location where you store your data. The alternative is that the data storage does not mandate too many rules or restrictions and you handle this at the application layer. However, such an approach will inevitably lead to problems as the data gets reused by other applications, applications are refactored or built from scratch without proper knowledge of the reasoning...

A basic example of this is the "unique" contraint. You can tell the database that a particular field or set of fields is unique, if you try to insert a second row with the same field(s), the database will simply not allow it. Alternatively you can do a unique constraint at the application level by first selecting on your keys and only inserting if it doesn't exist.

This is a "fragile" design, the database itself should implement as many of the guarantees that it can. This is however, never everything.

## Validation

Nabu supports quite a few validations that allow you to validate your data. These validations stem originally from XML Schema but others (like java bean validation) have been added over time.
In most services there is a checkbox to explicitly validate the input and/or output. For REST services it is generally adviseable to turn these on.

These validations allow you to set patterns (regexes), length requirements on strings, ranges on numbers,... These validations are field-based, cross-field validations are currently not (yet) supported. An example of this is [schematron](https://en.wikipedia.org/wiki/Schematron).

## Applicative Mutation

Mutating data often needs to follow a set of business rules. For example perhaps an order can only be added to a company _if_ said company has been verified by your legal department. Employees can only be removed from the database if HR confirms that he was actually fired etc.

This mutation logic can not really be captured in a database, and is (by necessity) captured in the application. Ideally however, this is a separate layer within the application, idealogically you can think of this as a microservice within your application. Microservices capture the business logic necessary to mutate the underlying data.

## Single Truth

An important part of designing your data structures is to ensure that each particular piece of information is only stored once. Suppose you have the company name present in two fields, and at some point you update one of those two fields. You now have two fields contradicting each other when it comes to the name of the company. Which one is correct?

Duplication is in 99% of the cases a bad idea, but once every so often it _is_ the best solution given certain performance challenges. However, it must be a very well considered, well documented decision, not a key design philosophy.

The same goes for derivative data, if you have an order and it has orderlines. Each orderline has (presumably) a "unitPrice" and an "amount" field. Calculating the total value of an orderline is then simply ``unitPrice * amount``. Do you store this in a third column "total" or do you calculate it on the fly? Same goes for the order: the total of the order is (simplistically) the sum of all the orderline totals.

Do you store it separately and thus create a duplicate that needs to be kept in sync, or do you calculate it at runtime?

This often depends on the required performance, the end result is however indeniably linked to the separate fields, updating those fields without updating the end result will end in a similar inconsistency.

There are alternatives like database "views". These provide different views on existing tables and (by default) are calculated on the fly. In case the calculations are too complex, you can opt for ``materialized views`` which store the end result until you force it to recalculate. You still risk accessing stale data, but at least updating the data to the latest state is trivial (recompute the view). In the case of the orderlines you need to know the business logic behind the data to recalculate it correctly.

A new feature that is likely coming to nabu soon is computed fields!

## Data Integrity

When you are modifying complex datasets, you want an "all or nothing" approach. If you need to create an order and some orderlines, you want to be sure that both the order and the lines exist in case of success and in case of some failure that neither exists. You don't want to end up with a partial state where for example the order was persisted but the orderlines were not.

To this end you can use transactions. In nabu transactions are largely automated which means you generally don't have to care about it. If those few cases that it _does_ matter you can take control over the transactions in a number of ways.
Interesting read: [ACID](https://en.wikipedia.org/wiki/ACID): atomicity, consistency, isolation, durability.

A transaction generally has a set of statements that are "staged", followed by a final "commit" or "rollback" depending on the success or failure of the end result.

To preserve data integrity across systems, there is a [two-phase commit](https://en.wikipedia.org/wiki/Two-phase_commit_protocol) concept where the first commit primes the data for the final commit, the final commit is only done if all systems agree that the first commit was a success.

In my experience 2PC are rarely used as they have added complexity while still leaving open the possibility of inconsistent end results. It just makes it statistically less likely rather than impossible.

Note that transactions lead to locking depending on the [transaction isolation|https://en.wikipedia.org/wiki/Isolation_(database_systems)].

## Primary Keys

Of particular interest in data models are primary keys: as a general rule of thumb every record should be identifiable by a single unique field.
This should preferably be an internal id, not an external one. For example a VAT can be considered to be an external id of a company. External ids can get special treatment (e.g. unique restrictions etc) but should not be your primary key.

In general there are two accepted primary keys:

- a sequence: a number that increases every time you create a new record. Usually there is a single sequence per table.
- uuid: a [universal unique identifier](https://en.wikipedia.org/wiki/Universally_unique_identifier) that is mathematically guaranteed to have a _very_ tiny colission chance. There are 4 variants, some of which are less than ideal to use (as they leak internal information), we generally use type 4 securely randomly generated ids.

The added advantage of a uuid is that it is unique _cross_ table and even cross database whereas numeric keys are only unique for a given table.
The downside of uuids is that they take up more space and are difficult to "quickly" reference in a conversation. "Hey, check out order 345" is a lot easier than "Hey, check out order 123e4567-e89b-12d3-a456-426652340000".

Another upside of uuids is that they can be generated without going to the database. This means, at the application level you can generate your id, use it in various contexts, and asynchronously persist it in the database at a later time. Sequences are managed and generated by the database so you need a roundtrip to the database to get the next id.

The downside of uuids in this regard is that they are not sequential, there is no inherent order to uuids. Sequences are guaranteed to be in order of creation (more or less). Note that sequences are guaranteed to be unique and counting upwards, they are _not_ guaranteed to be sequential!

## Sequences

If you want to use a sequence in nabu, you can select your numeric field and set the property `generated` to true. At that point, nabu will no longer send that field (by default) when doing an insert, letting the database fill it in.

Additionally nabu will generate a sequence declaration in the DDL, for example:

```
create sequence seq_pharmacies_id;
create table pharmacies (
	id integer primary key default nextval('seq_pharmacies_id'),
	licensee text not null
);
```

In the SQL services you can fill in the "generated column name" property at the top, at that point the actual number that was generated will appear in the output in the `generatedKeys` section.

# Modeling

Especially when you are familiar with object oriented programming, you will likely create a hierarchical data model. For example if we were to model a very simple "company", it could look like this:

[:.resources/hierarchical-company.png]

When you are using NOSQL databases which act like document stores, this setup could be enough to persist the company. You use a key that is the id, the content of the company as value and persist it. However, relational databases are still the industry standard (and preferred) method of storing data because they offer much better data quality guarantees.

At that point we need to model it as a relation of "flat" documents however, using ids to refer to one another with foreign keys:

[:.resources/relational-company.png]

Most relations between entities are either ``1-1`` (a belgian resident can have exactly one pasport and one pasport belongs to exactly one resident) or ``1-many`` where for example a person can have multiple cars but each car has exactly one owner. In that case the element that is "many" will generally have the id of the singular element as part of its definition.

A special type of relation is however the ``many-many``, for example you can like multiple songs on spotify, but multiple people can like the same song. At this point it is no longer enough to have the id in one of the two tables, instead you need a third table to model this:

[:.resources/many-to-many.png]

## Tools

For a long time we used UML (more specifically with argouml) to model the data. After a long development freeze however, that particular project is now officially dead. Luckily UML is exported as a largely standard XMI file, apart from some specific extensions for the specific modeling tool.

That XMI can be given to nabu who can generate the necessary datatypes from it. It can generate both hierarchical and relational views, with our without extensions and some other options.

We are however no longer using UML in our new projects for a number of reasons:

- the tool we were using to model is dead
- as a general practice, the goal for Nabu is to offer a single tool, both for the developer and the server. The underlying goal is to make use of nabu as simple as possible. If, besides developer, you need 5 other tools, custom build pipelines etc, it quickly becomes unfeasible to set up for non-technical people. UML was a holdout from the early days, data modeling in nabu has since caught up (and in a lot of cases gone beyond) what you can do in UML.
- flexibility: apart from features that you have in nabu that you can't really express in UML, nabu also has to make a number of assumptions when importing the UML. Designing in nabu gives you much more flexibility to make decisions on a structure basis rather than a whole model basis.

Apart from UML and nabu structures, nabu also understands other data models like xml schema, json schema, java beans,...

## Extensions

When designing data models, extensions are an interesting concept to use.

For example suppose you are modelling a "pharmacy" and a "retailer". Both are separate types of entities with specific fields. However, they also share a lot of data which we can capture as "organisation". Every pharmacy is an organisation, every retailer is an organisation.

You can do this with extensions: pharmacy can extend organisation which gives it all the fields organisation has plus some specific ones for pharmacy.

Once you start with extensions at the data model level, you can choose (at the database level) how you express those:

In this particular example do we have three tables? An organisation table with all the common data and two extension tables?
Or do we have simply 2 tables that each have all the fields for their respective organisation type?

The first design has the benefit of being able to do searches on the organisation without specifically needing to know (or care) whether it is a pharmacy or a retailer.
The second design has the benefit that selects on pharmacies for example are not impacted by the retailers. This can be especially important for high volume data.

However, in reality, the first design (the three table approach) is almost always the correct answer.

Most analysts will map such an extension like this:

[:.resources/classic-extension.png]

Where the pharmacy (the one who extends) refers to the organisation (the one who is extended) by a separate field with a foreign key. This is also absolutely necessary if you are working with sequences.
However, once you switch to uuids there is another option where the id of the pharmacy is simply a copy (and foreign key) to the id of the organisation:

[:.resources/custom-extension.png]

We have been doing this for quite a few years now and it makes things a _lot_ easier.

## Nabu

In nabu, any data type can (more or less) be exported to the database. However, once you explicitly fill in the "collection name" property of a structure, nabu knows that you specifically want that document as a collection in the database. This gives it more awareness of how you want to handle the table. It also determines the final table name if filled in.

Contrary to "popular" naming conventions, we use the plural for table names, this specifically to have fewer naming colissions (there are a lot of reserved singular words).

[^.resources/extension-modeling.mp4]

In this example we model the pharmacy as an extension of organisation. Initially nabu assumes you want to store the entire data model in a single table. Once you mark organisation as being a table in its own right, nabu will treat it as such and split off pharmacy into a new table. To keep the id field and (more importantly) keep it in sync, you can use the duplicate attribute. In "normal" situations, a field name should be unique across the entire pharmacy. However, for storage you may want to duplicate "some" values, id is a particular example, but also fields like "created" or "modified" might want to be persisted in each table. They are more for meta annotation than structural data.

When generating selects, Nabu also allows you to switch between the "all fields in one table" vs "multiple tables":

[^.resources/join-select.mp4]

## Model vs Emodel

In the UML artifact you can choose to generate either the separate tables view or the merged tables view, this leads to us often having a "model" (separate tables) and an "emodel" (merged tables view) where the model is often used to to atomary actions on a single table (insert, update, delete) whereas the merged emodel view is used for selections cross table.

However, this dates from an earlier period, nabu has been updated to have much more robust generic support for those atomary statements, if you now pass a merged table view (e.g. the pharmacy in the above) to the generic ``nabu.services.jdbc.Services.insert``, it will do two insertions (one in organisation and one in pharmacy) and in the correct order to guarantee foreign keys etc. This means we are slowly migrating to "emodel-only" views though this will take time.

## Restrictions

Nabu allows you to extend a data structure and then restrict it. In the background there are multiple types of restrictions, but from a developer point of the view the most important and visible one is via the restrict property:

[^.resources/restrict.mp4]

Why would you need this? We generally try to use the data model as much as possible throughout our application, including our REST services that feed our frontend. However, in most cases you don't want the frontend to see _all_ the data, just specific fields. Or in case of insert or update you may only allow some fields to be filled in by the user.

You could make a new structure that only has the fields you want, but this "duplication" needs to be manually updated if the data model is updated. By extending, then restricting, all the data types are automatically updated when a data model change occurs.

Additionally, from a type perspective, the two types are now related as one extends the other, rather than totally unrelated types that happen to look similar.

## Evolving the data model

If you look at some of our oldest projects you will see that changing the data model is...painful. Simply adding a field to the data model requires application level changes in a ton of places: selects, rest services, business mappings... 

Over time we have evolved new features and conventions to streamline this that a change to the database model is largely transparent for every level of the application. 

For example suppose you have this select:

```sql
select
	o.id,
	o.vat,
	p.licensee
from ~organisations o join ~pharmacies p on p.id = o.id
```

The output of the select is the extended model, which derives directly from the data model. If you add a field to the pharmacy, e.g. `publicName`, the output will be updated (as it directly references the model), but the SQL statement will not. In this particular example, you can write this to make it "change-proof":

```sql
select *
from ~organisations o join ~pharmacies p on p.id = o.id
```

At runtime, Nabu will expand the star into the correct columns based on the table information it has.

Some things to keep in mind:

- use the generic insert/update/delete/merge services ``nabu.services.jdbc.Services`` rather than dedicated insert/update/... jdbc services. Not only is this more intelligent (e.g. inserts into multiple tables), it is also more future proof as you don't have hardcoded insert statements.
- as mentioned in the example above: write selects using the * syntax rather than writing out the fields. This does rely on a correctly annotated output document.
- use extensions and restrictions rather than duplicate document structures
- map documents at the root whenever possible rather than seperate lines

[^.resources/root-mapping.mp4]

The separate lines suffer the same problem: if you add a new field, it will not have a line. By mapping at the root level however, any additional field is automatically picked up. This includes changing your mind about a particular restriction.

If two types are related through extension, you can draw a regular line as Nabu Developer knows they are related. If they are unrelated, you can draw a masked line. You can drag the line as you would normally and hold control. At that point the line becomes red when you drop it. This allows completely unrelated types to be mapped automatically by name. Where possible, nabu will also do automatic conversion.

## CRUD

By far the most common actions on data can be described as CRUD: create, read, update, delete. In sql terminology: insert, select, update, delete (ISUD just sounds crappier).

With the conventions laid out above, you can do CRUD actions that don't break the minute you update the data model. All these conventions are also captured in the CRUD provider. This is a type of artifact where you hand it a data model and it will generate services that work in the background according to the conventions.

You can add restrictions by checking the fields you want to hide, add filters that will create dynamic select statements when needed and the resulting services that are available in the tree can be called directly or can be added to a web application as they are also valid REST services:

[:.resources/crud.png]

## Deployment

You can generate DDL for a particular data structure or a folder of multiple data structures by ``Right Clicking -> SQL -> Create DDL -> <Database>`` where the database should be the type you will be using as the create syntax is generally specific per database.

However, the easiest way to create database tables _and_ automatically deploy them is to use the synchronize feature: ``Right Clicking -> SQL -> Synchronize -> To Database -> <Database>``. At this point the table will be created if it doesn't exist. When you deploy the code for the first time, the table will automatically be created in the target environment. When you change the model (e.g. add a column), the alter table scripts will be automatically run in every environment (upon deployment).

If you want to deploy DML (so actual data rather than table definitions), you can use deployment actions. When creating a deployment, the source service will be run to gather all the necessary data. After deployment the target service will be run which can then persist the necessary data. These deployment actions can be used for other things than DML though that is their primary use today.

For CMS related items, there are existing deployment actions that you can plug into your project, these automatically deploy things like masterdata, security settings, translations...

## Change Tracking

The SQL services and the generic services have a change tracker input field. You can fill in the implementation service of the change tracker specification. If you are using the CMS, you can use the default change tracker implementation: nabu.cms.core.providers.misc.changeTracker

When you enable change tracking on a particular insert, update or delete statement, Nabu will calculate _exactly_ what changed. For example if you run an update statement with 10 fields and only one actually differs from the one in the database, only that field is flagged. 

These changes are then sent to the change tracker who can choose to persist them. The CMS does this in a generic table where you can view all the changes that occurred to a table over time.

## Translations

If you fill in the language input for an update statement, Nabu will check if the update is in the "default" language. If it is the default language, the update is performed on the table as per usual. If it is not the default language however, Nabu will calculate what is different from this data as compared to the data in the actual table at that point and store these changes in a translation table.

Upon select, you can then fill in the language you want to select the data in, Nabu will automatically fill in the correct values where applicable.

## Affixes

In the generated statements you might have noticed that table names are generally prefixed with "~". This can actually occur anywhere in the table name but is generally put in the front. In most cases, the sign is removed before the SQL is executed. However, if you fill in the affix configuration in the JDBC connection that is being used, this can change. 

Suppose for example you want to use the task framework which has pretty generic table names like "tasks". These table names might conflict with your own already existing tables. At that point you could fill in the affix to for example "nabu_". From that point on, the task framework will automatically use the table "nabu_tasks", avoiding naming conflicts.

## Communication

To build web applications we need to send data back and forth, as mentioned above, for our own REST services we want to keep the data types as much as possible in sync with the original data model. This makes it a lot easier. For example if you reference masterdata in your pharmacy, we send the id of the masterdata to the frontend, not the name. The frontend has enough tooling to support dynamic lookups to resolve the ids into sensible values.

When building API's for consumption by third parties however, we generally choose to create new data types that have resolved values in them for things like masterdata.
