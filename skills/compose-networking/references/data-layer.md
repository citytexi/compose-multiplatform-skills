# Data Layer: DTOs, Mappers, Services, Repositories

How network data becomes domain data. Client construction is in [client-setup.md](client-setup.md); error wrapping is in [error-handling.md](error-handling.md).

## DTOs

DTOs mirror the API contract exactly and hold no business logic. Every DTO is `@Serializable`, uses `@SerialName` when a JSON key doesn't match the Kotlin property name (snake_case wire format is the common case), and gives optional fields a default so `coerceInputValues`/missing keys don't crash decoding.

```kotlin
@Serializable
data class ItemListDto(
    val items: List<ItemDto>,
    val total: Int,
    @SerialName("next_page") val nextPage: String? = null,
)

@Serializable
data class ItemDto(
    val id: String,
    val name: String,
    val status: StatusDto = StatusDto.ACTIVE,
    @SerialName("created_at") val createdAt: Long,
)

@Serializable
enum class StatusDto {
    @SerialName("active") ACTIVE,
    @SerialName("archived") ARCHIVED,
}
```

## Mappers

Domain models carry no serialization annotations. Mapping happens at the repository boundary, so nothing above the data layer ever sees a DTO — screens, view models, and use cases only ever work with `Item` and `ItemStatus`.

```kotlin
data class Item(val id: String, val name: String, val status: ItemStatus, val createdAt: Long)
enum class ItemStatus { ACTIVE, ARCHIVED }

fun ItemDto.toDomain() = Item(
    id = id,
    name = name,
    status = ItemStatus.valueOf(status.name),
    createdAt = createdAt,
)

fun List<ItemDto>.toDomain() = map { it.toDomain() }
```

## API Service

The service wraps `HttpClient` in typed methods and returns DTOs — it does no mapping. Mapping is the repository's job. Each method opts its own request into `expectSuccess = true`, overriding the shared client's `expectSuccess = false` (see [client-setup.md](client-setup.md)) for that one call only: a non-2xx response then raises `ClientRequestException` / `ServerResponseException` instead of being silently decoded as if it were the success DTO. That matters because the repository below wraps this service with `safeCall` (from [error-handling.md](error-handling.md)), which classifies *thrown* exceptions, not status codes — see the `safeCall` section there for why a service used this way must throw on failure.

```kotlin
class ItemApi(private val client: HttpClient) {

    suspend fun getItems(page: Int = 1, limit: Int = 20): ItemListDto {
        return client.get("items") {
            expectSuccess = true
            parameter("page", page)
            parameter("limit", limit)
        }.body()
    }

    suspend fun getItem(id: String): ItemDto =
        client.get("items/$id") { expectSuccess = true }.body()

    suspend fun createItem(request: CreateItemRequest): ItemDto {
        return client.post("items") {
            expectSuccess = true
            contentType(ContentType.Application.Json)
            setBody(request)
        }.body()
    }

    suspend fun deleteItem(id: String) {
        client.delete("items/$id") { expectSuccess = true }
    }
}

@Serializable
data class CreateItemRequest(val name: String)
```

## Repository

The repository is the network boundary: it calls the API service, maps DTOs to domain models, and returns every result wrapped in `NetworkResult<T>` (defined in [error-handling.md](error-handling.md)) instead of a raw domain type or a Kotlin `Result`. Two wrappers get you there, and which one applies depends on how the call is shaped: `safeRequest` is an `HttpClient` extension for a repository that builds the request itself and wants status-code granularity; `safeCall` runs any suspending block — including a call into `ItemApi` — through the same failure classification, from the thrown-exception side. `ItemRepositoryImpl` below goes through `ItemApi` rather than building requests itself, so it uses `safeCall`.

```kotlin
interface ItemRepository {
    suspend fun getItems(): NetworkResult<List<Item>>
    suspend fun getItem(id: String): NetworkResult<Item>
}

class ItemRepositoryImpl(private val api: ItemApi) : ItemRepository {

    override suspend fun getItems(): NetworkResult<List<Item>> =
        safeCall { api.getItems().items.toDomain() }

    override suspend fun getItem(id: String): NetworkResult<Item> =
        safeCall { api.getItem(id).toDomain() }
}
```

Turning a `NetworkResult` into MVI screen state is covered by the compose-architecture skill; exposing a repository result as a `Flow` for collection is covered by the compose-async skill. Neither concern belongs in this file.

### Offline-first shape

Some repositories treat local storage as the source of truth: the repository refreshes local storage from the network, and the UI observes the local `Flow` instead of reacting to individual network calls.

```kotlin
class OfflineFirstItemRepository(
    private val api: ItemApi,
    private val dao: ItemDao,
) {
    val items: Flow<List<Item>> = dao.observeAll().map { entities -> entities.map { it.toDomain() } }

    suspend fun refresh(): NetworkResult<Unit> = safeCall {
        val remote = api.getItems().items
        dao.replaceAll(remote.map { it.toEntity() })
    }
}
```

`refresh()` goes through `safeCall` for the same reason `ItemRepositoryImpl` does above: `api.getItems()` throws on a non-2xx response (see the API Service section), and a raw call here would bypass classification entirely — the `Flow` above keeps serving the last-known-good rows from `dao`, but the caller still needs a typed signal that the refresh itself failed (e.g. an MVI screen showing a "couldn't refresh" banner over the cached list) rather than an unhandled exception. Local persistence (`ItemDao`, entities) is out of scope for this skill and belongs to a future persistence skill — it is shown here only to place the network boundary: `refresh()` is where a remote DTO list turns into rows an offline-first UI can observe.

## Type-Safe Resources

The Ktor Resources plugin maps `@Resource`-annotated classes to HTTP paths, replacing hand-built path/query strings with compile-time-checked types. Add `ktor-client-resources` to the version catalog and `implementation(libs.ktor.client.resources)`, then `install(Resources)` alongside `ContentNegotiation` on the client from [client-setup.md](client-setup.md). Reference: [Ktor type-safe requests](https://ktor.io/docs/client-resources.html).

```kotlin
import io.ktor.resources.*
import kotlinx.serialization.Serializable

@Serializable
@Resource("/articles")
class Articles {
    @Serializable
    @Resource("{id}")
    class ById(val parent: Articles = Articles(), val id: Int)

    @Serializable
    @Resource("search")
    class Search(val parent: Articles = Articles(), val query: String, val page: Int = 1)
}

// Nested paths and query params resolve from the resource tree, e.g. /articles/42, /articles/search?query=compose&page=1
val articles: List<ArticleDto> = client.get(Articles()).body()
val article: ArticleDto = client.get(Articles.ById(id = 42)).body()
val results: ArticleListDto = client.get(Articles.Search(query = "compose")).body()
val created: ArticleDto = client.post(Articles()) {
    contentType(ContentType.Application.Json)
    setBody(CreateArticleRequest(title = "New Article"))
}.body()
client.delete(Articles.ById(id = 42))
```
