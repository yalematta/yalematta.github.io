---
layout: post
title: "DataStore is the new SharedPreferences"
author: "Layale Matta"
categories: blog
tags: [android, kotlin, jetpack, datastore]
image: preferences_datastore.jpg
---

You've definitely used SharedPreferences to store small or simple data sets. But SharedPreferences' API has a series of [downsides](https://android-developers.googleblog.com/2020/09/prefer-storing-data-with-jetpack.html) and luckily the Jetpack DataStore library aims at addressing those issues.

So if you're currently using SharedPreferences, consider migrating to DataStore instead. 
And good news, it's now in Beta 🎉

## 🔍 What is DataStore? 

Jetpack DataStore is a data storage solution that provides two different implementations: 
Preferences DataStore and Proto DataStore.

**Preferences DataStore** stores and accesses data using keys.
<br>
**Proto DataStore** stores data as instances of a custom data type and requires creating a schema using protocol buffers.

DataStore uses Kotlin coroutines and Flow to store data asynchronously, consistently, and transactionally unlike SharedPreferences.

### 🤿 Let's dive 

In this article, we will focus on **Preferences DataStore**. 

In this simple [project](https://github.com/yalematta/datastore-demo), we are implementing the _**Remember Me**_ functionality of a Login screen. We are currently using SharedPreferences to store this value and redirect the user to the Welcome screen once it's checked. We will migrate the code to use DataStore.

<img src="../assets/img/preferences_login.png" width="300"/> <img src="../assets/img/preferences_welcome.png" width="300"/>

To get your hands on the code, consider checking this [GitHub repo](https://github.com/yalematta/datastore-demo). <br>
The final code is located in the [_preferences_datastore_](https://github.com/yalematta/datastore-demo/tree/preferences_datastore) branch.

### 🛑 SharedPreferences Limitations 

The biggest downsides of SharedPreferences include: 
- Lack of safety from runtime exceptions
- Lack of a fully asynchronous API
- Lack of main thread safety
- No type safety

Luckily Jetpack DataStore addresses those issues. Since it's powered by Flow, DataStore saves the preferences in a file and performs all data operations on Dispatchers.IO under the hood. And your app won't be freezing while storing data. 

## 🏁 Let's get started... 

First, add the Preference DataStore dependency in the build.gradle file:

<script src="https://gist.github.com/yalematta/348c8c97a8e97ecb17dffb8081d499de.js"></script>

We have also added the Lifecycle dependencies for using ViewModel:

<script src="https://gist.github.com/yalematta/8b915f66235df897bef53dcf01d2637c.js"></script>

## 🗃️ DataStore Repository 
Inside a new package called _**repository**_, create the Kotlin class **DataStoreRepository.kt**. In this class we are going to store all the logic necessary for writing and reading DataStore preferences. We will pass to it a dataStore of type `DataStore<Preferences>` as a parameter. 

<script src="https://gist.github.com/yalematta/bb26e69e192bb33b170ead7249ad97ac.js"></script>

Let's create a data class called **UserPreferecences**. It will contain the two values we're going to save.

<script src="https://gist.github.com/yalematta/34049de665faff336330d137a4905d1b.js"></script>

Unlike SharedPreferences, in DataStore we cannot add a _**key**_ simply as a String. Instead we have to create a `Preferences.Key<String>` object or simply by using the extension function _`stringPreferencesKey`_ as follows:

<script src="https://gist.github.com/yalematta/08866644d645bc2f3e1a556a510f76db.js"></script>

### 📝 Write to DataStore 

In order to save to DataStore, we use the _`dataStore.edit`_ method using the keys we created above.

<script src="https://gist.github.com/yalematta/95f5f58529aa52397daab93616ab542b.js"></script>

You may have noticed that we're using a suspend function here. This is because _`dataStore.edit`_ uses Coroutines. This function accepts a `transform` block of code that allows us to transactionally update the state in DataStore.  It can also throw an IOException if an error was encountered while reading or writing to disk.

### 📋 Read from DataStore 

To read our data, we will retrieve it using _`dataStore.data`_ as a `Flow<UserPreferences>`.
Later, we are going to convert this Flow emitted value to LiveData in our ViewModel. 

<script src="https://gist.github.com/yalematta/9a521534d54821199014aec796346c4e.js"></script>

Make sure to handle the IOExceptions, that are thrown when an error occurs while reading data. Do this by using _`catch()`_ before _`map()`_  and emitting _`emptyPreferences()`_.

### 🆑 Clear DataStore 

To clear data, we can either clear the preferences all together or clear a specific preference by its key.

<script src="https://gist.github.com/yalematta/0fe8c6aa5e43b070803be431a3099864.js"></script>

## 🤙🏼 Call it from the ViewModel 

In another _**viewmodel**_ package, create the **LoginViewModel** class. 

<script src="https://gist.github.com/yalematta/4d0150753d3dbe9a2432b1a4ce832042.js"></script>

We're retrieving _userPreferences_ and converting the Flow into LiveData in order to observe it in our Activity. And since _`saveToDataStore`_ and _`clearDataStore`_  are suspended functions, we need to run them from inside a coroutine scope, which is the viewmodel scope in this case.

**LoginViewModelFactory** is a `ViewModelProvider.Factory` that is responsible to create our instance of **LoginViewModel** later in our Activity. We will pass to it the **DataStoreRepository** which is need in **LoginViewModel**'s constructor.

## 🗄️ Create DataStore 

<script src="https://gist.github.com/yalematta/7bfd2c23a821f87f2344af2fa9fbf9c8.js"></script>

### 📦 Migrate from SharedPreferences 

If we are migrating our existing data from the SharedPreferences, when creating our DataStore, we should add a migration based on the SharedPreferences name. DataStore will be able to migrate from SharedPreferences to DataStore automatically, for us. 

<script src="https://gist.github.com/yalematta/2b265802c8afe71049e343e1e802c854.js"></script>

## 🔬 Observe it in the Activity 

In our activity, we first observe our userPreferences as liveData from our ViewModel.

<script src="https://gist.github.com/yalematta/ec554c854cfdc8be3fc9b819aa25db8f.js"></script>

Whenever _**Remember Me**_ is observed as checked, we redirect the user to the Welcome screen. Whenever we click the login button, if our checkbox is checked we update our userPreferences, otherwise we clear our saved user preferences.

For the simplicity of our application, we will use the same ViewModel in our **WelcomeActivity** as well. We observe the _username_ and display it whenever it is not empty. And once we log out we clear our saved userPreferences. 

<script src="https://gist.github.com/yalematta/e8f38328cd5a6e8debe8ef88f8b429f8.js"></script>

## 💡 Key Takeaways 

Now that we migrated to Preferences DataStore let's recap! 

**DataStore**:
- is a replacement for SharedPreferences
- has a fully asynchronous API using Kotlin coroutines and Flow
- guarantees data consistency
- handles data migration
- handles data corruption

## ⏭ Up next 

Join me in the next post to learn how to use [Proto DataStore](https://yalematta.dev/blog/proto-datastore.html).

<br>
If this post was of any help to you, or if you think it requires further explanation, I'd love to know! Drop me a DM on Twitter [@yalematta](https://twitter.com/yalematta) ✌🏼