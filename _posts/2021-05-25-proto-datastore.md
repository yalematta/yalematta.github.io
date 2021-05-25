---
layout: post
title: "Persisting Typed Objects with DataStore"
author: "Layale Matta"
categories: blog
tags: [android, kotlin, jetpack, datastore]
image: proto_datastore.jpg
---

In a [previous post](https://yalematta.dev/blog/preferences-datastore.html) we have explored how to use **Preferences DataStore** to migrate our **SharedPreferences**. And in this post we will focus on **Proto DataStore**.

## What makes **Proto DataStore** different? 🤔

While both **Preferences** and **Proto DataStore** allow saving data, they do this in different ways. One important feature is applied only when using **Proto DataStore**, and it is **Type Safety**. 

With **Proto DataStore** we don't have to use keys. Instead we define a schema using Protocol Buffers. And with Protocol Buffers we can persist **strongly typed data**.

One of the downsides of **SharedPreferences** and **Preferences DataStore** is that there is no way to ensure that keys are accessed with the correct type. Using **Proto DataStore**, we know what types are stored and we will just provide them.

## What's Protocol Buffers? 🔍

Protocol Buffers were developed by Google for storing and interchanging all kinds of structured information. They are used for **serializing structured data**. It is similar to XML and other similar data formats, but **it's smaller, faster and simpler**.

If you don't know what serialization is, it's the process of translating a data structure or object state into a format that can be stored, like for example, a file or a memory data buffer.

### How Protocol Buffers work?👷🏻

We need to define how we want our data to be structured once, and the compiler will generate the source code for us to easily read and write the structured data. 

That said, with **Proto Datastore** we will serialize and save our custom data structure using Protocol Buffers and of course deserialize and read the values whenever we need to.

### Let's dive 🤿

In this simple [project](https://github.com/yalematta/datastore-demo), we are implementing the same _**Remember Me**_ functionality as in the previous post. We are currently using **SharedPreferences** to store this value and redirect the user to the Welcome screen once it's checked. We will migrate the code to use **Proto DataStore**.

<img src="../assets/img/proto_login.png" width="300"/> <img src="../assets/img/proto_welcome.png" width="300"/>

To get your hands on the code, consider checking this [GitHub repo](https://github.com/yalematta/datastore-demo). <br>
The final code is located in the [_proto_datastore_](https://github.com/yalematta/datastore-demo/tree/proto_datastore) branch.

## Define Protobuf schema 🏷️

In order to use **Proto Datastore** we will need to define our Protobuf schema in a new `.proto` file. So instead of creating a model class in Kotlin, we will define it in a Protobuf schema instead. And there's a **new syntax** that we need to follow when writing this schema.

We will install a plugin in Android Studio called **Protocol Buffer Editor** that will help us write the Proto file. Make sure to install it before creating your proto files.

<img src="../assets/img/protobuf_plugin.png" width="600"/> 

Once installed, we switch to our **Project View** and under **`app/src/main`** we create a new directory called **proto**. Inside this directory we create a new file **`user_prefs.proto`** where we define our Protocol Buffer schema as follows:

<script src="https://gist.github.com/yalematta/cfa7afec0e8f9a72c08fe213b359182f.js"></script>

```kotlin
syntax = "proto3";

option java_package= "com.yalematta.datastore_demo";
option java_multiple_files = true;

message UserPreferences {
  bool remember = 1;
  string username = 2;
  int32 luckyNumber = 3;
}
```

Let me walk you through this syntax:

### 1️⃣ Syntax

There are 2 versions for the Protobuff syntax: proto2 and proto3. You can check the [documention](https://developers.google.com/protocol-buffers/docs/proto3) for more info regarding these two different versions. In our case, we are going to use **proto3**.

### 2️⃣ Options

Then, we are going to write 2 options:
First, our **`java_package`** name. We need it in order to tell our compiler where to generate our classes from this protocol buffer. 
The second option is **`java_multiple_files`**. We will set it to true and this means that we need to create a separate file for each top level **message** object of this proto file.

### 3️⃣ Message

The **message** keyword defines the data structure. And inside it, we define the members of this structure. As you may have noticed we have different primitive types  in this syntax. 

<img src="../assets/img/protobuf_types.png" width="600"/> 

By taking a look at the [documentation](https://developers.google.com/protocol-buffers/docs/proto3), we can learn that: int32 in Java represents an Integer, int64 a Long and bool is a Boolean.

We create an object **UserPreferences** with 2 member fields: a bool, a string and an int32. Don't be confused when you see these 1, 2 and 3. These are not actual values but unique numbers to identify our fields in the _message binary format_ and they **should not be changed** once our message object is in use.

## Dependencies 🔌

Before we continue let's place this **plugin** at the top of our build.gradle file.

<script src="https://gist.github.com/yalematta/b7bfcee4e0dae16c35fd55530c32b527.js"></script>

```
plugins {
    id "com.google.protobuf" version "0.8.12"
}
```

Then we need to add two dependencies, one for Protobuf and one for Proto DataStore.

<script src="https://gist.github.com/yalematta/dfcf0b37b4c885a207626f5bb25238a1.js"></script>

```
// proto dataStore
implementation  "androidx.datastore:datastore:1.0.0-beta01"
implementation  "com.google.protobuf:protobuf-javalite:3.10.0"
```

And finally at the end of our build.gradle file we configure Protobuf and we **sync our project**.

<script src="https://gist.github.com/yalematta/bbece7f997ba217e93676b6e64038afb.js"></script>

```
protobuf {
    protoc {
        artifact = "com.google.protobuf:protoc:3.10.0"
    }

    // Generates the java Protobuf-lite code for the Protobufs in this project. See
    // https://github.com/google/protobuf-gradle-plugin#customizing-protobuf-compilation
    // for more information.
    generateProtoTasks {
        all().each { task ->
            task.builtins {
                java {
                    option 'lite'
                }
            }
        }
    }
}
```

Now that we have added this plugin, we should be able to see the automatically generated files by this plugin from our **`user_prefs.proto`** file. 

Rebuild the project to see those files inside the java (generated) folder. We find a new **UserPrefs** folder that represents our proto file,  and a **UserPreferences** class that represents our message object. Inside it we have java code that implements some getters and setters for this UserPreferences message object.

## Serializer ↪️

To tell DataStore how to read and write the data type we defined in the proto file, we need to implement a Serializer. The Serializer defines also the default value to be returned if there's no data saved yet.

Back in our project, we create a class called **UserPreferencesSerializer** which extends Serializer<UserPreferences>. We implement its two methods readFrom and writeTo. In these methods we define how we want to read and write this object into our DataStore.

<script src="https://gist.github.com/yalematta/de4e14ae3e6c154cca432eecb31cd8e1.js"></script>

```
object UserPreferencesSerializer : Serializer<UserPreferences> {
    override val defaultValue: UserPreferences = UserPreferences.getDefaultInstance()
    override suspend fun readFrom(input: InputStream): UserPreferences {
        try {
            return UserPreferences.parseFrom(input)
        } catch (exception: InvalidProtocolBufferException) {
            throw CorruptionException("Cannot read proto.", exception)
        }
    }

    override suspend fun writeTo(t: UserPreferences, output: OutputStream) = 
        t.writeTo(output)
}
```

## DataStore Repository 🗃️

Next we create our Repository which we call **UserPreferencesRepository**.

<script src="https://gist.github.com/yalematta/dc5d09a3f1fe116b673bc1b0e6c49db0.js"></script>

```
class UserPreferencesRepository(private val userPreferencesStore: DataStore<UserPreferences>){
    ...
}
```

### Read from DataStore 📋

We create a new variable called userPreferencesFlow of type Flow<UserPreferences>. We use our dataStore to read the data and catch exceptions if there is any and emit the default instance of UserPreferences in that case.

<script src="https://gist.github.com/yalematta/0a71f274d6ad057bdddc2c9087fc9434.js"></script>

```
    val userPreferencesFlow: Flow<UserPreferences> = userPreferencesStore.data
        .catch { exception ->
            // dataStore.data throws an IOException when an error is encountered when reading data
            if (exception is IOException) {
                Log.e(TAG, "Error reading sort order preferences.", exception)
                emit(UserPreferences.getDefaultInstance())
            } else {
                throw exception
            }
        }
```

### Write to DataStore 📝

We create the suspend updateUsername function which will update one field from our UserPreferences member values. We will call **`preference.toBuilder().`** and we choose the setter method that we need from our generated class.

<script src="https://gist.github.com/yalematta/8df794025887336597365c5558a820d7.js"></script>

```
    suspend fun updateUsername(username: String) {
        userPreferencesStore.updateData { preferences ->
            preferences.toBuilder().setUsername(username).build()
        }
    }
```

P.S: Don't forget to create a method to update each field.

### Clear DataStore 🆑

To clear data, we can either clear the preferences all together or clear a specific preference by its method from our generated class.

<script src="https://gist.github.com/yalematta/47c77317f29dee5b1a047bee985898c8.js"></script>

```
    suspend fun clearDataStore() {
        userPreferencesStore.updateData { preferences ->
            preferences.toBuilder().clear().build()
        }
    }

    suspend fun clearUsername() {
        userPreferencesStore.updateData { preferences ->
            preferences.toBuilder().clearUsername().build()
        }
    }
```

## Call it from the ViewModel 🤙🏼

In our **LoginViewModel**, we create a variable for our **UserPreferences**, read its data from our DataStore as a Flow and then convert it to LiveData.

Next we create a new function named **saveUserPreferences** and we pass to it the values that we want to update. We call viewModel scope and run the following code inside a coroutine since our update functions in our Repository are using Kotlin Coroutines.

<script src="https://gist.github.com/yalematta/0ca185c7acc4ef45b170cfa476946d5f.js"></script>

```
class LoginViewModel(private val userPreferencesRepository: UserPreferencesRepository) : ViewModel() {

    val userPreferencesFlow = userPreferencesRepository.userPreferencesFlow.asLiveData()

    fun saveUserPreferences(remember: Boolean, username: String, luckyNumber: Int) {
        viewModelScope.launch(Dispatchers.IO) {
            userPreferencesRepository.updateRemember(remember)
            userPreferencesRepository.updateUsername(username)
            userPreferencesRepository.updateLuckyNumber(luckyNumber)
        }
    }

    fun clearUserPreferences() {
        viewModelScope.launch(Dispatchers.IO) {
            userPreferencesRepository.clearDataStore()
        }
    }
}

class LoginViewModelFactory(
    private val userPreferencesRepository: UserPreferencesRepository
) : ViewModelProvider.Factory {

    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        if (modelClass.isAssignableFrom(LoginViewModel::class.java)) {
            @Suppress("UNCHECKED_CAST")
            return LoginViewModel(userPreferencesRepository) as T
        }
        throw IllegalArgumentException("Unknown ViewModel class")
    }
}
```
**LoginViewModelFactory** is a ViewModelProvider.Factory that is responsible to create our instance of **LoginViewModel** later in our Activity. We will pass to it the **DataStoreRepository** which is need in **LoginViewModel**'s constructor.

## Observe it in the Activity 🔬

### Create DataStore 🗄️

In our Activity, we first create our userPreferencesDataStore and we initialize it and pass to it a file name as well as our Serializer class. 

<script src="https://gist.github.com/yalematta/55129c3f89abfc16731669495395347c.js"></script>

```kotlin
private const val DATA_STORE_FILE_NAME = "user_prefs.pb"

val Context.userPreferencesStore: DataStore<UserPreferences> by dataStore(
    fileName = DATA_STORE_FILE_NAME,
    serializer = UserPreferencesSerializer
)
```

### Migrate from SharedPreferences 📦

If we are migrating our existing data from the SharedPreferences, when creating our DataStore, we should add a migration based on the SharedPreferences name. 

And when creating the dataStore we need to update the DataStore builder and assign to the migrations parameter a new list that contains an instance of our SharedPreferencesMigration.

Define the mapping logic from SharedPreferences to UserPreferences inside your SharedPreferencesMigration.

DataStore will be able to migrate from SharedPreferences to DataStore automatically, for us. 

<script src="https://gist.github.com/yalematta/6c975819f7a3e1535f24863c9b20d4a0.js"></script>

```kotlin
const val USER_PREFERENCES_NAME = "user_preferences"
private const val DATA_STORE_FILE_NAME = "user_prefs.pb"

val Context.userPreferencesStore: DataStore<UserPreferences> by dataStore(
    fileName = DATA_STORE_FILE_NAME,
    serializer = UserPreferencesSerializer,
    produceMigrations = { context ->
        listOf(sharedPrefsMigration(context))
    }
)

fun sharedPrefsMigration(context: Context) = SharedPreferencesMigration(
    context, USER_PREFERENCES_NAME) { sharedPrefs: SharedPreferencesView, currentData: UserPreferences ->
    // Define the mapping from SharedPreferences to UserPreferences
    currentData
}
```

Inside our onCreate function, we initialize our ViewModel and we observe our fields' values, so that whenever this data changes we will update it in its corresponding text field. 

And whenever we click our login button, we store the value from our editText and checkBox field and update it in our DataStore using the saveUserPreferences function.

<script src="https://gist.github.com/yalematta/3c9193d6c6e1108c641d8a728f883472.js"></script>

```kotlin
class LoginActivity : AppCompatActivity() {

    private lateinit var binding: ActivityLoginBinding
    private lateinit var viewModel: LoginViewModel

    private var rememberMe = false
    private var luckyNumber = 0
    private lateinit var username: String

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityLoginBinding.inflate(layoutInflater)
        val view = binding.root
        setContentView(view)

        viewModel = ViewModelProvider(
            this,
            LoginViewModelFactory(UserPreferencesRepository(userPreferencesStore))
        ).get(LoginViewModel::class.java)

        viewModel.userPreferencesFlow.observe(this, { userPreferences ->
            rememberMe = userPreferences.remember
            username = userPreferences.username
            luckyNumber = userPreferences.luckyNumber
            if (rememberMe) {
                startActivity(Intent(this, WelcomeActivity::class.java))
            }
        })

        binding.login.setOnClickListener {
            if (binding.remember.isChecked) {
                val name = binding.username.text.toString()
                var number = luckyNumber
                if (binding.luckyNumber.text.toString().isNotEmpty()) {
                    number = binding.luckyNumber.text.toString().toInt()
                }
                viewModel.saveUserPreferences(true, name, number)
            }
            startActivity(Intent(this, WelcomeActivity::class.java))
        }

        binding.remember.setOnCheckedChangeListener { compoundButton: CompoundButton, b: Boolean ->
            if (!compoundButton.isChecked) {
                viewModel.clearUserPreferences()
            }
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

DataStore has 2 different implementations: Preferences DataStore and Proto DataStore.

[**Preferences DataStore**](https://yalematta.dev/blog/preferences-datastore.html):
- stores and accesses data using keys

**Proto DataStore**:
- ensures Type Safety
- requires defining a schema using Protocol Buffers

<br>
If this post was of any help to you, or if you'd like me to write about another specific Android related topics, let me know! <br>
Drop me a DM on Twitter [@yalematta](https://twitter.com/yalematta) ✌🏼