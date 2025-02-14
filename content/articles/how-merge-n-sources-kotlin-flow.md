+++
title = "How to merge N sources using Kotlin Flow"
date = "2019-10-20"
tags = ["kotlin", "coroutines", "flow"]
draft = true
+++

In an effort to expand the projects that I am working on from being only on the JVM, I've been migrating them to [Kotlin multiplatform](https://kotlinlang.org/docs/reference/multiplatform.html). This year [Flow](https://medium.com/@elizarov/kotlin-flows-and-coroutines-256260fb3bdb) was introduced, the most appropriate replacement for RxJava.

Not all operators that one can find on RxJava are available on Kotlin's Flow, but the goal is not to have a perfect replacement anyway. Due to how coroutines works, operators are much [easier to implement and to maintain](https://medium.com/@elizarov/simple-design-of-kotlin-flow-4725e7398c4c).

Recently I was converting a class that powered a screen that presented a list of offline items, including ones in the process of being downloaded. Naturally, I needed to persist both the metadata as well as the audio files for offline playback. The method to get all book's metadata was defined as:

```kotlin
fun offlineAudiobooks(): Flow<List<Audiobook>>
```

The audio files though, are managed by a 3rd-party vendor, with methods to query the download status individually, per book id. A simplified version is:

```kotlin
fun downloadState(audiobook: Audiobook): Flow<Status>

sealed class Status {
	data class Progress(val percent: Double) : Status()
	data class Error(val throwable: Throwable) : Status()
	object Completed : Status()
	object NotDownloaded : Status()
}
```

The UI layer is reactive and based on an uni-directional data flow that follows the concept introduced by Redux with a class that represents the state of the screen at any time of the application's lifecycle.

To correctly display the downloads statuses one would have to first query what books are offline, to then for each item of the returned list get the up-to-date download state, to finally flat all streams. Since one can have any number of offline items, it does not seem to be intuitive how one can merge N streams.

Luckily there is an [operator](https://github.com/Kotlin/kotlinx.coroutines/blob/f605b26cd08b6b1fbef889310f5aa6c28ade2e72/kotlinx-coroutines-core/common/src/flow/operators/Zip.kt#L286-L301) that accepts an `Iterable` and a function with `FlowCollector` being the receiver. Its simplified prototype is:

```kotlin
fun <T, R> combineTransform(
    flows: Iterable<Flow<T>>,
    transform: suspend FlowCollector<R>.(Array<T>) -> Unit
): Flow<R>
```

The way it fits the solution is:

```kotlin
offlineAudiobooks().flatMapConcat { books ->
  val offlineBooksFlows = books.map { book ->
    downloader.downloadState(book).map {
      OfflineAudiobook(book, it)
    }
  }
  return@flatMapConcat combineTransform(offlineBooksFlows) {
    emit(it.toList())
  }
}
```

Since I'm building on top of Kotlin coroutines, `map {}` calls are easier to understand because they apply for both synchronous as well as asynchronous calls.

Simplicity is relative, and I was pleased to have came up with a complex algorithm such as the one above.
