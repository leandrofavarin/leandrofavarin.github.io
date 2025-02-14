+++
title = "Drive your UI with SQLDelight’s views"
summary = "SQLDelight’s views can help you create flexible user interfaces with minimal effort."
date = "2019-09-22"
tags = ["sqldelight", "kotlin"]
+++

Users demand more from apps every year and 2019 feels as though this pressure has never been so high. Among many ways we can improve the user experience as developers, driving the interface from a local database is an easy way to achieve robustness and flexibility. This is especially the case with database tools that can broadcast changes in the data set to interested parties.

Suppose you’re writing a database-driven application using [SQLDelight](https://github.com/cashapp/sqldelight), and you need to present the same type of data in slightly different ways. It could be that they differ on how they’re ordered, or just a subset of them.

SQLDelight automatically generates model objects for every query that you write. In many cases, the types it generates are either from a single column (`SELECT id FROM Foo`) or from the star projection (`SELECT * FROM Foo`). The former being simple Kotlin String/Long/Int types, and the latter the type SQLDelight generates from your table definition. What happens, then, if you select a subset of the properties declared on your table?

In this case SQLDelight will create a new type for every query, with its name being the named query:

```sql
bandsOrderedByName:
SELECT id, name
FROM band
ORDER BY name DESC;

bandsOrderedByAge:
SELECT id, name
FROM band
ORDER BY age;
```

A simplified code of what SQLDelight generates is:

```kotlin
data class BandsOrderedByName(id: String, name: String)

data class BandsOrderedByAge(id: String, name: String)
```

If you wanted to transform them *later on*, you would have to write two functions of types `(BandsOrderedByName) -> X` and `(BandsOrderedByAge) -> X`, even though the underlying composition is the same: two `String`s.

While this is a simple exercise, in real-life examples projections tend to be more complex, usually with [join clauses](https://www.sqlite.org/syntax/join-clause.html) and many properties selected. For example:

```sql
SELECT
  band.id, 
  band.name,
  album.*
FROM band
JOIN album ON band.id = album.band_id; 
```

A SQL [View](https://sqlite.org/lang_createview.html) solves this problem elegantly, with our queries now being:

```sql
CREATE VIEW bandWithAlbum AS
SELECT
  band.id, 
  band.name,
  album.*
FROM band
JOIN album ON band.id = album.band_id; 

bandsOrderedByName:
SELECT *
FROM bandWithAlbum
ORDER BY name DESC;

bandsOrderedByAge:
SELECT *
FROM bandWithAlbum
ORDER BY age;
```

SQLDelight will generate `BandWithAlbum` type that can be queried, ordered, and filtered differently, per query. Much of the SELECT boilerplate from the queries is eliminated while still allowing for specific constraints, additional joins for scoping, as well as selecting as a list or as a single item. Moreover, it creates a solid building block for pagination, with:

```sql
count:
SELECT count(*)
FROM bandWithAlbum;

paged:
SELECT *
FROM bandWithAlbum
LIMIT ?
OFFSET ?;
```

Minimal effort is then needed to present items to the user by using only one [RecyclerView.Adapter](https://developer.android.com/reference/androidx/recyclerview/widget/RecyclerView.Adapter).

A combination of any out-of-the-box extension such as for [Kotlin Coroutines](https://github.com/cashapp/sqldelight/tree/master/extensions/coroutines-extensions) or [RxJava](https://github.com/cashapp/sqldelight/tree/master/extensions/rxjava2-extensions) plus [DiffUtil](https://developer.android.com/reference/androidx/recyclerview/widget/DiffUtil) allows the user to easily visualize the data the way they want, because since the model being generated is a [value class](https://kotlinlang.org/docs/reference/data-classes.html), the diff callbacks can be trivially written. One example would be:

```kotlin
object BandItemCallback : ItemCallback<BandWithAlbum>() {
  override fun areItemsTheSame(oldItem: BandWithAlbum, newItem: BandWithAlbum): Boolean {
    return oldItem.id == newItem.id
  }

  override fun areContentsTheSame(oldItem: BandWithAlbum, newItem: BandWithAlbum): Boolean {
    return oldItem == newItem
  }
}
```

Going back to the queries with different sorting, one could use an enum class to model the sort options and use Kotlin’s [`when` expression](https://kotlinlang.org/docs/reference/control-flow.html#when-expression) to write:

```kotlin
enum class Sort { NAME, AGE }

fun bandsSorted(by: Sort): Flow<List<BandWithAlbum>> = when (by) {
  NAME -> db.bandsOrderedByName()
  AGE -> db.bandsOrderedByAge()
}.asFlow().mapToList()
```

and delegate to SQL algorithms that would be [less efficient](https://speakerdeck.com/jakewharton/the-resurgence-of-sql-droidcon-nyc-2017?slide=123) to run if they were executed programmatically.

Building views for the types you want and then querying those views for the data you need is an easy way to attain responsiveness and flexibility with very little code needed. It’s clear that user demands will only get more intense as technology develops. It’s therefore crucial to stay on top of these simple ways to improve the experience without breaking our backs (or keyboards).

---

*Many thanks to [Jake Wharton](https://jakewharton.com) and [Alec Strong](https://www.alecstrong.com) for proofreading this article!*
