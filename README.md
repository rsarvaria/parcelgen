Parcelgen
=========

Parcelgen simplifies the process of building objects from JSON descriptions and passing those objects between Android Activities.

What?
-----

Parcelgen is a Python script coupled with a small set of Java utilities (accessible as an Android library project or as a drop-in jar) that generates Java code for basic data objects in an application. Parcelgen generates code for the objects to be created from JSON representations and to read and write thesemselves to Android [Parcels][parcel].

Included is a [complete example application](https://github.com/Yelp/parcelgen/tree/master/parcelgen-example) which fetches businesses from the public [Yelp API](http://www.yelp.com/developers/documentation/v2/search_api) and displays their details, using parcelgen to construct objects from the API response and then to pass the objects between activities.

How?
----

### Describe your data objects in JSON

Create a folder called `parcelables` in the top level of your Android application. For each object you want to represent with Parcelgen, create a json file with the same filename as your Java class name.

This is the parcelable description for (a subset of) a [Business returned by the Yelp API](http://www.yelp.com/developers/documentation/v2/search_api#rValue) as used in [the example app](https://github.com/Yelp/parcelgen/blob/master/parcelgen-example/parcelables/Business.json):

``` javascript
{
    "do_json": true,
    "package": "com.yelp.parcelgen",
    "props": {
        "String": [
            "id", "name", "imageUrl", "url", "mobileUrl",
            "phone", "displayPhone", "ratingImageUrl",
            "ratingImageUrlSmall", "snippetText", "snippetImageUrl"
        ],
        "int": ["reviewCount"],
        "double": ["distance"],
        "Location": ["location"]
    },
    "json_map": {
        "ratingImageUrl": "rating_img_url",
        "ratingImageUrlSmall": "rating_img_url_small"
    }
}
```

The Java files generated by parcelgen can be seen in the example at [_Business.java](https://github.com/Yelp/parcelgen/blob/master/parcelgen-example/src/com/yelp/parcelgen/_Business.java) and [Business.java](https://github.com/Yelp/parcelgen/blob/master/parcelgen-example/src/com/yelp/parcelgen/Business.java). Note that the method `toString()` in Business.java was added after the code was generated.

In the json description of object you want to create: `package` is the Java package the object will be placed in, `props` is a hash of Java type to variable names for instance variables, do_json instructs Parcelgen to generate code to construct the object from JSON, and json_map is for properties that don't map directly to JSON properties; by default Parcelgen will convert Java style CamelCase ivar names to JSON-style under_separated names, but if they don't line up directly json_map can override the default. Parcelgen infers the Java class name from the filename.

### Execute parcelgen.py and pass in your description file

    $ python ~/parcelgen/parcelgen.py parcelables/Business.json src/

(where `src` is the path to your Android app's source directory)

### Examine your shiny new Java classes

When creating a class for the first time, Parcelgen creates two files, `YourClassName.java` and `_YourClassName.java`, in the folder corresponding to the package name specified in the description. `YourClassName.java` is just small class which inherits from `_YourClassName.java` and contains a [`CREATOR`](http://d.android.com/reference/android/os/Parcelable.Creator.html) property as required by [Parcelable][parcelable].

The magic is in the `readFromParcel()` and `readFromJson()` methods parcelgen creates in the superclass. These methods take a `Parcel` or `JSONObject` and set the target's instance variables based on the contents of the provided object.

When you re-run parcelgen if you change the properties of on object, parcelgen will overwrite the class beginning with an `_` but not the subclass. This allows you to add logic methods and any other custom handling to the underscoreless subclass without losing the ability to modify the object.

### Add the parcelgen runtime to your Android project

Parcelgen requires a small Java library with utility methods for reading and writing Parcels and JSON. [Download parcelgen.jar here](https://github.com/downloads/Yelp/parcelgen/parcelgen.jar) and put it in the `libs` directory of your Android project (if `libs` doesn't exist, create it). If you're using Eclipse, you'll also need to add the jar to your build path by right clicking on your project -> Properties -> Java Build Path -> Libraries -> Add Jar -> navigate to your project/libs/parcelgen.jar.

If you'd rather add the library project directly, you can import [parcelgen-runtime](https://github.com/Yelp/parcelgen/tree/master/parcelgen-runtime) into Eclipse and add it to your project as an Android library dependency. If you do this, do not add the jar to your project.

### Pass the objects around

Want to pass an object to a new activity in an Intent? Just use `Intent.putExtra()` ([BusinessesActivity.java](https://github.com/Yelp/parcelgen/blob/master/parcelgen-example/src/com/yelp/parcelgen/BusinessesActivity.java#L31)):

``` java
intent.putExtra(EXTRA_BUSINESS, mBusinesses.get(position));
```

Then, in your [Activity's `onCreate()`](https://github.com/Yelp/parcelgen/blob/master/parcelgen-example/src/com/yelp/parcelgen/BusinessActivity.java):

``` java
Business business = getIntent().getParcelableExtra(EXTRA_BUSINESS);
```

Want to create a list of objects from a [JSON array of dictionaries](http://www.yelp.com/developers/documentation/v2/search_api#sampleResponse)? Check out how the [example does it](https://github.com/Yelp/parcelgen/blob/master/parcelgen-example/src/com/yelp/parcelgen/SearchActivity.java#L47):

``` java
JSONObject response = new JSONObject(result);
List<Business> businesses = JsonUtil.parseJsonList(
    response.getJSONArray("businesses"), Business.CREATOR);
```

Why?
----

Since Activities in Android are designed to be independent entities, Android does not allow passing arbitrary objects in memory between them. There are several ways around this. A popular method is to use some sort of global singleton to hold the object(s) you need to pass. As an application grows, this technique gets cumbersome and messy. Additionally, since an Android application can be killed at any time, this doesn't help the activity restart where it was after the its process is killed. Another technique (and one frequently touted by Google's Android team) is to save objects in an sqlite database and pass keys in the database between activities. While this is an excellent technique from an architectual standpoint, the implementation overhead becomes huge as more types of objects are added, and objects' properties change. Finally objects can be seralized using [Java Serialization][serializable], [JSON][jsonobject], or as Android's native high-speed data transport system: [Parcels][parcel].

While working on [Yelp for Android][yelpdroid] I found myself frequently wanting to pass particular objects or collections of objects (such as Yelp Users or Businesses) between activities in Intents. Java and JSON serialization have a reputation for being particularly slow on Android, so I decided to pass objects to new Activities using [Intent.putExtra(String name, Parcelable value)][putextra]. Additionally, when your primary data objects can be written to parcels it's easy to persist them in a Bundle, like for [onSaveInstanceState()](http://d.android.com/reference/android/app/Activity.html#onSaveInstanceState%28android.os.Bundle%29) when your Activity is suspended.

Further Documentation
---------------------

### How parcelgen decides how to persist object properties

Parcelgen supports writing and reading the following types to a Parcel:

* The native Java types: `byte`, `double`, `float`, `int`, `long`, and `java.lang.String` directly use Parcel's [write*Type*()][parcel] methods.
* `boolean`: Parcels only support arrays of booleans, so Parcelgen will place all of the boolean properties of on object into a boolean array before writing to a Parcel, and unpack them in the same order when reading.
* `java.util.Date`: parcelgen stores the value returned by [Date.getTime()](http://d.android.com/reference/java/util/Date.html#getTime%28%29). When reading from JSON, Parcelgen will convert unix timestamps in seconds since epoch into Java `Date` objects.
* Objects which implement [Serializable][serializable], as specified in the `serializables` property documented in the next section.
* Objects which themselves implement [Parcelable][parcelable]. If parcelgen doesn't know what do with an object, it assumes the object has a CREATOR property and uses that to write and read the object from a Parcel.
* `List`s or `ArrayList`s of any of the above object types. Specify the property as a Java generic type: `List<Business>`, and parcelgen will use the above logic to read and write the contents of the list.

### Properties you can use in a parcelgen json description

* **props**: A dictionary of Java type name to list of instance variable names.
* **package**: The Java package where the generated class should be placed
* **imports**: Additional classes to import, in case a member's type is not in the same package as the object being described. Note that parcelgen handles importing the correct classes for any types officialy supported (such as `Date` and `Uri`).
* **serializables**: A list of any properties which should be read and written using [Serializable][serializable]. Since parcelgen doesn't inspect the Java files, it can't tell whether a class is Serializable or Parcelable.
* **do_json**: Whether to generate the code to read this class from json. Leave this out or set it to false if you won't be using parcelgen's json reading/writing features.
* **json_map**: A dictionary of instance variable to json property for properties that parcelgen cannot guess, such as `{"dateCreated": "time_created"}`.
* **json_blacklist**: A list of properties which shouldn't be read from json. Use this for properties you want to passed in Parcels but not read from json.
* **do_json_writer**: If enabled, Parcelgen will generate code to create a `JSONObject` mirroring what would have been read from JSON. **Note that this is currently experimental**, in particular, List-type properties are not supported.
* **default_values**: An optional dictionary containing the value certain properties should be given by default. For instance useful to defaulting integer values to -1:

``` javascript
"default_values": {
    "geoAccuracy": -1
}
```

### Missing Features and Further Work

Parcelgen does not support properties that are Java array types. This should be straightforward to implement, but hasn't been needed.

Parcelgen does not integrate with any of Android's build tools or Eclipse. I don't know ant, but it should be possible to have parcelgen automatically update generated files when its json descriptions are changed, in which case the classes generated by parcelgen shouldn't be kept in version control

Parcelgen only reads and writes json using the built-in json.org based [JSONObject][jsonobject] interface. Support for other parsers such as [Jackson](http://jackson.codehaus.org/) would be a great addition.


[parcel]: http://d.android.com/reference/android/os/Parcel.html
[parcelable]: http://d.android.com/reference/android/os/Parcelable.html
[yelpdroid]: https://market.android.com/details?id=com.yelp.android&feature=search_result
[serializable]: http://d.android.com/reference/java/io/Serializable.html
[jsonobject]: http://d.android.com/reference/org/json/JSONObject.html
[putextra]: http://d.android.com/reference/android/content/Intent.html#putExtra%28java.lang.String%2C%20android.os.Parcelable%29
