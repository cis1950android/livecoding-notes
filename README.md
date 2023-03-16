# Connecting to the Internet

## 0. Add dependencies

**NOTE: all dependencies mentioned in this tutorial are already included in your starter code, except the ones in section 7**

```kotlin
// in app-level build.gradle file
   implementation "com.squareup.retrofit2:retrofit:2.9.0"
   implementation "com.squareup.retrofit2:converter-scalars:2.9.0"
```

## 1. Create API interface

### 1.1 - In our case, let's call it `JokesApiService.kt`, this will be our endpoint to instantiate and use `Retrofit`

```
Right click on the main package -> new -> Kotlin Class/File
```

### 1.2 - Create an instance of `Retrofit`

To do so, we first need to provide the base URL of the API

```kotlin
private const val BASE_URL = "https://official-joke-api.appspot.com/"
```

Next, use `Retrofit.Builder` to create the Retrofit object

```kotlin
private val retrofit = Retrofit.Builder()
    .addConverterFactory(ScalarsConverterFactory.create())
    .baseUrl(BASE_URL)
    .build()
```

Note that we use `ScalarsConverterFactory` for starters, but we will need to provide a JSON converter factory later on. For now, we just want to display raw JSON as text.

### 1.3 - Create an interface to define API calls

```kotlin
interface JokesApiService {
    @GET("random_ten")
    fun getTenJokes():
            Call<String>
}
```

### 1.4 - Create a public object to expose the Retrofit service to the rest of the app

```kotlin
object JokesApi {
    val retrofitService : JokesApiService by lazy {
        retrofit.create(JokesApiService::class.java)
    }
}
```

## 2. Use the retrofit object in `ViewModel`

### 2.1 - Create a function that uses `JokesApi` to retrieve the jokes
```kotlin
//JokeViewModel.kt
private fun getJokes() {
    JokesApi.retrofitService.getTenJokes().enqueue(object: Callback<String> {
        override fun onResponse(call: Call<String>, response: Response<String>) {
            TODO("Not yet implemented")
        }

        override fun onFailure(call: Call<String>, t: Throwable) {
            TODO("Not yet implemented")
        }

    })
}
```

Then call the function inside the `init` block

```kotlin
//JokeViewModel.kt
init {
    getJokes()
}
```

### 2.2 - Populate the livedata with the API response

```kotlin
//in the body of the two overridden methods
override fun onResponse(call: Call<String>, response: Response<String>) {
        _response.value = response.body()
    }

override fun onFailure(call: Call<String>, t: Throwable) {
        _response.value = "ERROR: " + t.message
    }
```

## 3. Add Permissions

Add the following to `AndroidManifset.xml` right above the `<application>` tag

```kotlin
    <uses-permission android:name="android.permission.INTERNET" />
```

## 4. Receive the data in the UI Controller

```kotlin
val viewModel: JokesViewModel by viewModels()
```
```kotlin
//MainActivity.kt
viewModel.response.observe(this, Observer {
    binding.dataHolderText.text = it
})
```

## 5. Parsing the JSON response

Now that we were able to see the data on the app, it is time to get rid of the `ScalarsConverterFactory` and use a converter factory that parses JSON data. For this, we will use a library called `Moshi`.

### 5.1 - Add dependencies

```kotlin
//app-level gradle
    implementation "com.squareup.moshi:moshi:1.13.0"
    implementation "com.squareup.moshi:moshi-kotlin:1.13.0"
    implementation "com.squareup.retrofit2:converter-moshi:2.9.0"
```

### 5.2 - Replace ScalarsConverterFactory

In `JokesApiService.kt`, replace this: 
```kotlin
private val retrofit = Retrofit.Builder()
    .addConverterFactory(ScalarsConverterFactory.create())
    .baseUrl(BASE_URL)
    .build()
```
with this:

```kotlin
 private val moshi = Moshi.Builder()
       .add(KotlinJsonAdapterFactory())
       .build()

private val retrofit = Retrofit.Builder()
    .addConverterFactory(MoshiConverterFactory.create(moshi))
    .baseUrl(BASE_URL)
    .build()
```

### 5.3 - Create a data class to store the JSON response as an object

For this step, you have two options:

1. Create the class manually by looking at the JSON response and creating appropriate class fields
2. Use a cool extension called to generate it automatically (with Moshi annotations!). 

If you want to use the extension, you can find the docs [here](https://plugins.jetbrains.com/plugin/10054-generate-kotlin-data-classes-from-json). It is highly recommended, saves a lot of time. 

Install it through: `File -> Settings (for MacOS: Preferences)` then type "plugins" in the search bar. Next, search for the plugin by typing "Generate Kotlin Data Classes From JSON" and install it. Restart your Android Studio and follow the steps in the docs above to use it.

Here is the class we will be using for this tutorial: 

```kotlin
data class Joke(@Json(name = "punchline")
                val punchline: String = "",
                @Json(name = "setup")
                val setup: String = "",
                @Json(name = "id")
                val id: Int = 0,
                @Json(name = "type")
                val type: String = "")
```

### 5.4 - Use the data class as datatype for API call

In `JokesApiService.kt`, replace this: 
```kotlin
fun getTenJokes():
    Call<String>
```
with this:

```kotlin
fun getTenJokes():
    Call<List<Joke>>
```

In `JokeViewModel.kt`, change the datatype to `List<Joke>` as well

replace this:
```kotlin
    private fun getJokes() {
        JokesApi.retrofitService.getTenJokes().enqueue(object: Callback<String> {
            override fun onResponse(call: Call<String>, response: Response<String>) {
                _response.value = response.body()
            }

            override fun onFailure(call: Call<String>, t: Throwable) {
                _response.value = "ERROR: " + t.message
            }

        })
    }
```

with this: 

```kotlin
    private fun getJokes() {
        JokesApi.retrofitService.getTenJokes().enqueue(object: Callback<List<Joke>> {
            override fun onResponse(call: Call<List<Joke>>, response: Response<List<Joke>>) {
                _response.value = response.body()
            }

            override fun onFailure(call: Call<List<Joke>>, t: Throwable) {
                Log.i("API", "ERROR: " + t.message)
            }

        })
    }
```

Then, since `response.body()` is now of type `List<Joke>` and not `String`, we need to modify the LiveData variables `_response` and `response` accordingly. Notice how we used Log statements to handle the error above for the same reason.

Replace this:

```kotlin
    private val _response = MutableLiveData<String>()
    val response : LiveData<String>
        get() = _response
```

with this: 

```kotlin
    private val _response = MutableLiveData<List<Joke>>()
    val response : LiveData<List<Joke>>
        get() = _response
```

Lastly, we need to modify the last part in this propagated chain of changes: the UI controller!. In `MainActivity.kt`,

replace this:

```kotlin
viewModel.response.observe(this, Observer {
    binding.dataHolderText.text = it
})
```

with this:

```kotlin
viewModel.response.observe(this, Observer {
    binding.dataHolderText.text = it.size.toString()
})
```

Notice that for now we are only displaying the size of the list. The next step will involve displaying the entire list in a `RecyclerView`


## 6. Displaying the list in a RecyclerView

Follow the steps from lecture 7 to add a recyclerview

Here is the code ready to use for the purposes of this tutorial:

1. Create `JokeAdapter.kt`:
```kotlin
//JokeAdapter.kt
class JokeAdapter (private val jokes: MutableList<Joke>): RecyclerView.Adapter<JokeAdapter.JokeViewHolder>() {

    inner class JokeViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        // Your holder should contain and initialize a member variable
        // for any view that will be set as you render a row
        val jokeBody: TextView = itemView.findViewById(R.id.joke_body)
        val jokePunchline: TextView = itemView.findViewById(R.id.joke_punchline)
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): JokeAdapter.JokeViewHolder {
        val layoutInflater = LayoutInflater.from(parent.context)
        val view = layoutInflater
            .inflate(R.layout.joke_list_item, parent, false)
        return JokeViewHolder(view)
    }

    override fun getItemCount(): Int {
        return jokes.size
    }

    override fun onBindViewHolder(holder: JokeViewHolder, position: Int) {
        val joke = jokes[position]
        val jokeBody = holder.jokeBody
        val jokePunchline = holder.jokePunchline

        jokeBody.text = joke.setup
        jokePunchline.text = joke.punchline
    }
}
```
2. The Model data class already exists from step 5.3

3. Add a recyclerview in `activity_main.xml` and remove/comment out the TextView

```xml
    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/joke_list_rv"
        android:layout_width="0dp"
        android:layout_height="0dp"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layoutManager="androidx.recyclerview.widget.LinearLayoutManager"/>
```

4. Instantiate the recyclerview and adapter in `MainActivity.kt`

```kotlin
//MainActivity.kt
val recyclerView = binding.jokeListRv

viewModel.response.observe(this, Observer {
    val adapter = JokeAdapter(it)
    recyclerView.adapter = adapter
})
```

**Very Important Note:** The approach we are taking here will reinitialize the RV adapter every time we retrieve the data. Assuming that we have a refresh button or refresh onSwipe behavior, this is far from optimal. If you are interested to learn more about the appropriate behavior, check this [article](https://medium.com/geekculture/android-listadapter-a-better-implementation-for-the-recyclerview-1af1826a7d21) on implementing RV with `ListAdapter` and `DiffUtil` instead of `RecyclerView.Adapter`. What we currently have is fine for our purposes, though.

## 7. Extra: Implement refresh on swipe

### 7.1 - Add dependencies
```kotlin
implementation 'androidx.swiperefreshlayout:swiperefreshlayout:1.1.0'
```
### 7.2 - Replace ConstraintLayout with SwipeRefreshLayout

```xml
<androidx.swiperefreshlayout.widget.SwipeRefreshLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity"
    android:id="@+id/swipeLayout">
```

### 7.3 - Set an `onRefreshListener` in `MainActivity`

```kotlin
binding.swipeLayout.setOnRefreshListener {
    viewModel.getJokes()
    binding.swipeLayout.isRefreshing = false
}
```
and don't forget to make the function `getJokes()` public in `JokeViewModel.kt`
