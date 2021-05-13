---
layout: post
title: "Jetpack DataStore is the new SharedPreferences"
author: "Layale Matta"
categories: blog
tags: [android, kotlin, jetpack, datastore]
image: preferences_datastore.jpg
---

You've definitely used SharedPreferences to store small or simple data sets. But SharedPreferences' API has a series of [downsides](https://android-developers.googleblog.com/2020/09/prefer-storing-data-with-jetpack.html) and luckily the Jetpack DataStore library aims at addressing those issues.

So if you're currently using SharedPreferences, consider migrating to DataStore instead. 
And good news, it's now in Beta 🎉

## What is DataStore? 🔍

Jetpack DataStore is a data storage solution that provides two different implementations: 
Preferences DataStore and Proto DataStore.

**Preferences DataStore** stores and accesses data using keys.
</br>
**Proto DataStore** stores data as instances of a custom data type and requires creating a schema using protocol buffers.

DataStore uses Kotlin coroutines and Flow to store data asynchronously, consistently, and transactionally unlike SharedPreferences.

### Let's dive 🤿

In this article, we will focus on **Preferences DataStore**. 

In this simple [project](https://github.com/yalematta/datastore-demo), we are implementing the _**Remember Me**_ functionality of a Login screen. We are currently using SharedPreferences to store this value and redirect the user to the Welcome screen once it's checked. We will migrate the code to use DataStore.

<img src="../assets/img/preferences_login.png" width="300"/> <img src="../assets/img/preferences_welcome.png" width="300"/>

To get your hands on the code, consider checking the GitHub repo [_datastore-demo_](https://github.com/yalematta/datastore-demo). </br>
The final code is located in the [_preferences_datastore_](https://github.com/yalematta/datastore-demo/tree/preferences_datastore) branch.

### SharedPreferences Limitations 🛑

The biggest downsides of SharedPreferences include: 
- Lack of safety from runtime exceptions
- Lack of a fully asynchronous API
- Lack of main thread safety
- No type safety

Luckily Jetpack DataStore addresses those issues. Since it's powered by Flow, DataStore saves the preferences in a file and performs all data operations on Dispatchers.IO under the hood. And your app won't be freezing while storing data. 

## Let's get started... 🏁

First, add the Preference DataStore dependency in the build.gradle file:

```kotlin
implementation "androidx.datastore:datastore-preferences:1.0.0-beta01"
```

We have also added the Lifecycle dependencies for using ViewModel:
```kotlin
// architecture components
implementation "androidx.core:core-ktx:$coreVersion"
implementation "androidx.lifecycle:lifecycle-runtime-ktx:$lifecycleVersion"
implementation "androidx.lifecycle:lifecycle-viewmodel-ktx:$lifecycleVersion"
implementation "androidx.lifecycle:lifecycle-livedata-ktx:$lifecycleVersion"
```

## DataStore Repository 🗃️
Inside a new package called _**repository**_, create the Kotlin class **DataStoreRepository.kt**. In this class we are going to store all the logic necessary for writing and reading DataStore preferences. We will pass to it a dataStore of type `DataStore<Preferences>` as a parameter. 

```kotlin
class DataStoreRepository(private val dataStore: DataStore<Preferences>) {
    ...
}
```

Let's create a data class called **UserPreferecences**. It will contain the two values we're going to save.

```kotlin
data class UserPreferences(
    val username: String,
    val remember: Boolean
)
```

Unlike SharedPreferences, in DataStore we cannot add a _**key**_ simply as a String. Instead we have to create a `Preferences.Key<String>` object or simply by using the extension function _`stringPreferencesKey`_ as follows:

```kotlin
class DataStoreRepository(private val dataStore: DataStore<Preferences>) {

    private object PreferencesKeys {
        val USERNAME = stringPreferencesKey("username")
        val REMEMBER = booleanPreferencesKey("remember")
    }

 }
```
### Write to DataStore 📝

In order to save to DataStore, we use the _`dataStore.edit`_ method using the keys we created above.

```kotlin
suspend fun saveToDataStore(username: String, remember: Boolean) {
    dataStore.edit { preference ->
        preference[USERNAME] = username
        preference[REMEMBER] = remember
    }
}
```

You may have noticed that we're using a suspend function here. This is because _`dataStore.edit`_ uses Coroutines. This function accepts a `transform` block of code that allows us to transactionally update the state in DataStore.  It can also throw an IOException if an error was encountered while reading or writing to disk.

### Read from DataStore 📋

To read our data, we will retrieve it using _`dataStore.data`_ as a `Flow<UserPreferences>`.
Later, we are going to convert this Flow emitted value to LiveData in our ViewModel. 

```kotlin
val readFromDataStore : Flow<UserPreferences> = dataStore.data
    .catch { exception ->
        if (exception is IOException) {
            Log.d("DataStoreRepository", exception.message.toString())
            emit(emptyPreferences())
        } else {
            throw exception
        }
    }
    .map { preference ->
        val username = preference[USERNAME] ?: ""
        val remember = preference[REMEMBER] ?: false
        UserPreferences(username, remember)
    }
```
Make sure to handle the IOExceptions, that are thrown when an error occurs while reading data. Do this by using _`catch()`_ before _`map()`_  and emitting _`emptyPreferences()`_.

### Clear DataStore 🆑

To clear data, we can either clear the preferences all together or clear a specific preference by its key.

```kotlin
suspend fun clearDataStore() {
    dataStore.edit { preferences ->
        preferences.clear()
    }
}

suspend fun removeUsername() {
    dataStore.edit { preference ->
        preference.remove(USERNAME)
    }
}
```

## Call it from the ViewModel 🤙🏼

In another _**viewmodel**_ package, create the **LoginViewModel** class. 

```kotlin
class LoginViewModel(private val dataStoreRepository: DataStoreRepository)
    : ViewModel() {

    val userPreferences = dataStoreRepository.readFromDataStore.asLiveData()

    fun saveUserPreferences(username: String, remember: Boolean) {
        viewModelScope.launch(Dispatchers.IO) {
            dataStoreRepository.saveToDataStore(username, remember)
        }
    }

    fun clearUserPreferences() {
        viewModelScope.launch(Dispatchers.IO) {
            dataStoreRepository.clearDataStore()
        }
    }
}

class LoginViewModelFactory(
    private val dataStoreRepository: DataStoreRepository) 
    : ViewModelProvider.Factory {

    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        if (modelClass.isAssignableFrom(LoginViewModel::class.java)) {
            @Suppress("UNCHECKED_CAST")
            return LoginViewModel(dataStoreRepository) as T
        }
        throw IllegalArgumentException("Unknown ViewModel class")
    }
}
```

We're retrieving _userPreferences_ and converting the Flow into LiveData in order to observe it in our Activity. And since _`saveToDataStore`_ and _`clearDataStore`_  are suspended functions, we need to run them from inside a coroutine scope, which is the viewmodel scope in this case.

**LoginViewModelFactory** is a `ViewModelProvider.Factory` that is responsible to create our instance of **LoginViewModel** later in our Activity. We will pass to it the **DataStoreRepository** which is need in **LoginViewModel**'s constructor.

## Create DataStore 🗄️

```kotlin
private const val USER_PREFERENCES_NAME = "user_preferences"

val Context.dataStore by preferencesDataStore(
    name = USER_PREFERENCES_NAME
)
```

### Migrate from SharedPreferences 📦

If we are migrating our existing data from the SharedPreferences, when creating our DataStore, we should add a migration based on the SharedPreferences name. DataStore will be able to migrate from SharedPreferences to DataStore automatically, for us. 

```kotlin
private const val USER_PREFERENCES_NAME = "user_preferences"

private val Context.dataStore by preferencesDataStore(
    name = USER_PREFERENCES_NAME,
    produceMigrations = { context ->
        listOf(SharedPreferencesMigration(context, USER_PREFERENCES_NAME))
    }
)
```

## Observe it in the Activity 🔬

In our activity, we first observe our userPreferences as liveData from our ViewModel.

```kotlin
class LoginActivity : AppCompatActivity() {

    private lateinit var binding: ActivityLoginBinding
    private lateinit var viewModel: LoginViewModel

    private var rememberMe = false
    private lateinit var username: String

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityLoginBinding.inflate(layoutInflater)
        val view = binding.root
        setContentView(view)

        viewModel = ViewModelProvider(this,
            LoginViewModelFactory(DataStoreRepository(dataStore)))
                .get(LoginViewModel::class.java)

        viewModel.userPreferences.observe(this, { userPreferences ->
            rememberMe = userPreferences.remember
            username = userPreferences.username
            if (rememberMe) {
                startActivity(Intent(this, WelcomeActivity::class.java))
            }
        })

        binding.login.setOnClickListener {
            if (binding.remember.isChecked) {
                val name = binding.username.text.toString()
                viewModel.saveUserPreferences(name, true)
            }
            startActivity(Intent(this, WelcomeActivity::class.java))
        }

        binding.remember.setOnCheckedChangeListener { 
                compoundButton: CompoundButton, b: Boolean ->
            if (!compoundButton.isChecked) {
                viewModel.clearUserPreferences()
            }
        }

    }
}
```
Whenever _**Remember Me**_ is observed as checked, we redirect the user to the Welcome screen. Whenever we click the login button, if our checkbox is checked we update our userPreferences, otherwise we clear our saved user preferences.

For the simplicity of our application, we will use the same ViewModel in our **WelcomeActivity** as well. We observe the _username_ and display it whenever it is not empty. And once we log out we clear our saved userPreferences. 

```kotlin
class WelcomeActivity : AppCompatActivity() {

    private lateinit var binding: ActivityWelcomeBinding
    private lateinit var viewModel: LoginViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityWelcomeBinding.inflate(layoutInflater)
        val view = binding.root
        setContentView(view)

        viewModel = ViewModelProvider(this,
            LoginViewModelFactory(DataStoreRepository(dataStore)))
                .get(LoginViewModel::class.java)

        viewModel.userPreferences.observe(this, { userPreferences ->
            val username = userPreferences.username
            if (username.isNotEmpty()) {
                binding.welcome.text = 
                    String.format(getString(R.string.welcome_user), username)
            }
        })

        binding.logout.setOnClickListener {
            viewModel.clearUserPreferences()
            startActivity(Intent(this, LoginActivity::class.java))
        }
    }
}
```
## Key Takeaways 💡

Now that we migrated to Preferences DataStore let's recap! 

**DataStore**:
- is a replacement for SharedPreferences addressing most of its downsides
- has a fully asynchronous API using Kotlin coroutines and Flow
- guarantees data consistency
- handles data migration
- handles data corruption

## Up next ⏭

Join me in the next post to learn how to use **Proto DataStore**.

</br>
If this post was of any help to you, or if you think it requires further explanation, I'd love to know! </br>
Drop me a DM on Twitter [@yalematta](https://twitter.com/yalematta) ✌🏼