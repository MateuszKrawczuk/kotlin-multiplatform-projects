# Overview

This library borrows a lot of concepts from [Apollo](https://www.apollographql.com/docs).

## HttpFetcher
This library doesn't make any assumptions on how you implement your transport as long as your implementation
conforms with the `HttpFetcher` interface. A `JsonHttpFetcher` is an `HttpFetcher` works with `JSON` payloads only 
and you can provide an implementation if you are using `JSON`. 

For other protocols like `protobuf`, use `HttpFetcher` directly.
 
For example, below is JSON fetcher based on [ktor](https://ktor.io/clients/index.html).

```kotlin
class KtorFetcher : JsonHttpFetcher {
    override fun fetch(
        url: String,
        request: JsonHttpFetchRequest,
        headers: Map<String, String>?,
        handler: (JsonHttpFetchResponse) -> Unit
    ) {
        // launch fetch in background
        GlobalScope.launch {
            val client = HttpClient()
            try {
                val response = client.post<HttpResponse>(url) {
                    this.body = TextContent(request.body ?: "{}", contentType = ContentType.Application.Json)
                }
                handler(JsonHttpFetchResponse(response.readText(), response.status.value, "", response.status.value != 200, null))
            } catch (e: ClientRequestException) {
                val body = e.response.readText(Charset.forName("UTF-8"))
                
                handler(JsonHttpFetchResponse(body, e.response.status.value, e.localizedMessage, true, e.localizedMessage))
            } catch (e: Exception) {
                handler(JsonHttpFetchResponse(null, 0, e.localizedMessage, true, e.localizedMessage))
            }
        }
    }
}
```

## Link

A link takes an operation and returns an observable.
Links are a way to compose subsets of actions to handle
data. Links can be concatenated with each other to form
a chain to carry out an even complex data handling workflow. Conceptually,
it is identical to [Apollo-Link](https://www.apollographql.com/docs/link/overview/).

A link has only one method `execute` which accepts an operation of
type `A` and the next link in the chain. The next link can be `null` if
the current link is a terminating link. For example, the last link
can start a network request and return its response as an observable.

`JsonHttpGraphQlLink` provides a default implementation of a terminating link
which sends a JSON request using the provided `JsonHttpFetcher` implementation.

## Client
A client executes operations against a GraphQl server.
It accepts a chain of `Link` which emits the final response of type `T`.

The client should not concern itself with how to make a network request.
Rather, it should rely on the last `Link` of the `chain` as terminating
link which fetches the response from the server. `JsonHttpGraphQlLink` is
one such link. The client should then take the response as input
and transform it into the correct response type based on the operation.

## Example

```kotlin
fun main() {
    val ktorFetcher = KtorFetcher()
    val httpLink = JsonHttpGraphQlLink(ktorFetcher, "https://countries.trevorblades.com")
    val client = JsonHttpGraphQlClient(httpLink)
    val operation = Operations.Countries() // generated by plugin

    client.execute(operation)
        .subscribe {
            when(it) {
                is Result.Success -> {
                    println(" Successful ${it.value.data}")
                }
                is Result.Failure -> {
                    println("Failed ${it.value.error} ${it.value.response} ${it.cause}")
                }
            }
        }

    runBlocking {
        kotlinx.coroutines.delay(10000)
    }
}
```