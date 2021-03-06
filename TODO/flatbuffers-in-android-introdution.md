> * 原文链接 : [FlatBuffers in Android - introduction – froger_mcs dev blog – Coding with love {❤️}](http://frogermcs.github.io/flatbuffers-in-android-introdution/)
* 原文作者 : [froger_mcs dev blog](http://frogermcs.github.io/)
* 译文出自 : [掘金翻译计划](https://github.com/xitu/gold-miner)
* 译者 : 
* 校对者: 
* 状态 :  待定





JSON - probably everyone knows this lightweight data format used in almost all modern servers. It weights less, is more human-readable and in general is more dev-friendly than old-fashined, horrible xml. JSON is language-independend data format but parsing data and transforming it to e.g. Java objects costs us time and memory resources.  
Several days ago Facebook announced big performance improvement in data handling in its Android app. It was connected with dropping JSON format and replacing it with FlatBuffers in almost entire app. Please check [this article](https://code.facebook.com/posts/872547912839369/improving-facebook-s-performance-on-android-with-flatbuffers/)to get some basic knowledge about FlatBuffers and results of transition to it from JSON.

While the results are very promising, at the first glance the implementation isn’t too obvious. Also facebook didn’t say too much. That’s why in this post I’d like to show how we can start our work with FlatBuffers.

## FlatBuffers

In short, [FlatBuffers](https://github.com/google/flatbuffers) is a cross-platform serialization library from Google, created specifically for game development and, as Facebook showed, to follow the [16ms rule](https://www.youtube.com/watch?v=CaMTIgxCSqU) of smooth and responsive UI in Android.

_But hey, before you throw everything to migrate all your data to FlatBuffers, just make sure that you need this. Sometimes the impact on performance will be imperceptible and sometimes [data safety](https://publicobject.com/2014/06/18/im-not-switching-to-flatbuffers/) will be more important than a tens of milliseconds difference in computation speed._

What makes FlatBuffers so effective?

*   Serialized data is accessed without parsing because of flat binary buffer, even for hierarchical data. Thanks to this we don’t need to initialize parsers (what means to build complicated field mappings) and parse this data, which also takes time.
*   FlatBuffers data doesn’t need to allocate more memory than it’s used by buffer itself. We don’t need to allocate extra objects for whole hierarchy of parsed data like it’s done in JSON.

For the real numbers just check again [facebook article](https://code.facebook.com/posts/872547912839369/improving-facebook-s-performance-on-android-with-flatbuffers/) about migrating to FlatBuffers or [Google documentation](http://google.github.io/flatbuffers/) itself.

## Implementation

This article will cover the simplest way of using FlatBuffers in Android app:

*   JSON data is converted to FlatBuffer format _somewhere_ outside the app (e.g. bin ary file is delivered as a file or returned directly from API)
*   Data model (Java classes) is generated by hand, with **flatc** (FlatBuffer compiler)
*   There are some limitations for JSON file (null fields cannot be used, Date format is parsed as a String)

Probably in the future we’ll prepare more complex solution.

## FlatBuffers compiler

At the beginning we have to get **flatc** - FlatBuffers compiler. It can be built from source code hosted in Google’s [flatbuffers repository](https://github.com/google/flatbuffers). Let’s download/clone it. Whole build process is described on [FlatBuffers Building](https://google.github.io/flatbuffers/md__building.html) documentation. If you are Mac user all you have to do is:

1.  Open downloaded source code on `\{extract directory}\build\XcodeFlatBuffers.xcodeproj`
2.  Run **flatc** scheme (should be selected by default) by pressing **Play** button or `⌘ + R`
3.  **flatc** executable will appear in project root directory.

Now we’re able to [use schema compiler](https://google.github.io/flatbuffers/md__compiler.html) which among the others can generate model classes for given schema (in Java, C#, Python, GO and C++) or convert JSON to FlatBuffer binary file.

## Schema file

Now we have to prepare schema file which defines data structures we want to de-/serialize. This schema will be used with flatc to create Java models and to transform JSON into Flatbuffer binary file.

Here is a part of our JSON file:

    {
      "repos": [
        {
          "id": 27149168,
          "name": "acai",
          "full_name": "google/acai",
          "owner": {
            "login": "google",
            "id": 1342004,
            ...
            "type": "Organization",
            "site_admin": false
          },
          "private": false,
          "html_url": "https://github.com/google/acai",
          "description": "Testing library for JUnit4 and Guice.",
          ...
          "watchers": 21,
          "default_branch": "master"
        },
        ...
      ]
    }

Full version is available [here](https://github.com/frogermcs/FlatBuffs/blob/master/flatbuffers/repos_json.json). It’s a bit modified version of data which can be taken from Github API call: [https://api.github.com/users/google/repos](https://api.github.com/users/google/repos).

Writing a FlatBuffer schema is very well [documented](https://google.github.io/flatbuffers/md__schemas.html), so I won’t delve into this. Also in our case schema won’t be very complicated. All we have to do is to create 3 tables: `ReposList`, `Repo` and `User`, and define `root_type`. Here is the important part of this schema:

    table ReposList {
        repos : [Repo];
    }

    table Repo {
        id : long;
        name : string;
        full_name : string;
        owner : User;
        //...
        labels_url : string (deprecated);
        releases_url : string (deprecated);
    }

    table User {
        login : string;
        id : long;
        avatar_url : string;
        gravatar_id : string;
        //...
        site_admin : bool;
    }

    root_type ReposList;

Full schema file is [available here](https://github.com/frogermcs/FlatBuffs/blob/master/flatbuffers/repos_schema.fbs).

## FlatBuffers data file

Great, now all we have to do is to convert `repos_json.json` to FlatBuffers binary file and generate Java models which will be able to represent our data in Java-friendly style (all files required in this operation are [available](https://github.com/frogermcs/FlatBuffs/tree/master/flatbuffers) in our repository):

`$ ./flatc -j -b repos_schema.fbs repos_json.json`

If everything goes well, here is a list of generated files:

*   repos_json.bin (we’ll rename it to repos_flat.bin)
*   Repos/Repo.java
*   Repos/ReposList.java
*   Repos/User.java

## Android app

Now let’s create our example app to check how FlatBuffers format works in practice. Here is the screenshot of it:

![ScreenShot](http://frogermcs.github.io/images/17/screenshot.png "ScreenShot")

ProgressBar will be used only to show how incorrect data handling (in UI thread) can affect smoothness of user interface.

`app/build.gradle` file of our app will look like this:

    apply plugin: 'com.android.application'
    apply plugin: 'com.jakewharton.hugo'

    android {
        compileSdkVersion 22
        buildToolsVersion "23.0.0 rc2"

        defaultConfig {
            applicationId "frogermcs.io.flatbuffs"
            minSdkVersion 15
            targetSdkVersion 22
            versionCode 1
            versionName "1.0"
        }
        buildTypes {
            release {
                minifyEnabled false
                proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            }
        }
    }

    dependencies {
        compile fileTree(dir: 'libs', include: ['*.jar'])
        compile 'com.android.support:appcompat-v7:22.2.1'
        compile 'com.google.code.gson:gson:2.3.1'
        compile 'com.jakewharton:butterknife:7.0.1'
        compile 'io.reactivex:rxjava:1.0.10'
        compile 'io.reactivex:rxandroid:1.0.0'
    }

Of course it’s not necessary to use Rx or ButterKnife in our example, but why not to make this app a bit nicer 😉 ?

Let’s put repos_flat.bin and repos_json.json files to `res/raw/` directory.

Here is [RawDataReader](https://github.com/frogermcs/FlatBuffs/blob/master/app/src/main/java/frogermcs/io/flatbuffs/utils/RawDataReader.java) util which helps us to read raw files in Android app.

At the end put `Repo`, `ReposList` and `User` somewhere in project’s source code.

### FlatBuffers library

FlatBuffers provides java library to handle this data format directly in java. Here is [flatbuffers-java-1.2.0-SNAPSHOT.jar](https://github.com/frogermcs/FlatBuffs/blob/master/app/libs/flatbuffers-java-1.2.0-SNAPSHOT.jar) file. If you want to generate it by hand you have to move back to downloaded FlatBuffers source code, go to `java/` directory and use Maven to generate this library:

`$ mvn install`

Now put .jar file to your Android project, into `app/libs/` directory.

Great, now all we have to do is to implement `MainActivity` class. Here is the full source code of it:

    public class MainActivity extends AppCompatActivity {

        @Bind(R.id.tvFlat)
        TextView tvFlat;
        @Bind(R.id.tvJson)
        TextView tvJson;

        private RawDataReader rawDataReader;

        private ReposListJson reposListJson;
        private ReposList reposListFlat;

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);
            ButterKnife.bind(this);
            rawDataReader = new RawDataReader(this);
        }

        @OnClick(R.id.btnJson)
        public void onJsonClick() {
            rawDataReader.loadJsonString(R.raw.repos_json).subscribe(new SimpleObserver() {
                @Override
                public void onNext(String reposStr) {
                    parseReposListJson(reposStr);
                }
            });
        }

        private void parseReposListJson(String reposStr) {
            long startTime = System.currentTimeMillis();
            reposListJson = new Gson().fromJson(reposStr, ReposListJson.class);
            for (int i = 0; i < reposListJson.repos.size(); i++) {
                RepoJson repo = reposListJson.repos.get(i);
                Log.d("FlatBuffers", "Repo #" + i + ", id: " + repo.id);
            }
            long endTime = System.currentTimeMillis() - startTime;
            tvJson.setText("Elements: " + reposListJson.repos.size() + ": load time: " + endTime + "ms");
        }

        @OnClick(R.id.btnFlatBuffers)
        public void onFlatBuffersClick() {
            rawDataReader.loadBytes(R.raw.repos_flat).subscribe(new SimpleObserver() {
                @Override
                public void onNext(byte[] bytes) {
                    loadFlatBuffer(bytes);
                }
            });
        }

        private void loadFlatBuffer(byte[] bytes) {
            long startTime = System.currentTimeMillis();
            ByteBuffer bb = ByteBuffer.wrap(bytes);
            reposListFlat = frogermcs.io.flatbuffs.model.flat.ReposList.getRootAsReposList(bb);
            for (int i = 0; i < reposListFlat.reposLength(); i++) {
                Repo repos = reposListFlat.repos(i);
                Log.d("FlatBuffers", "Repo #" + i + ", id: " + repos.id());
            }
            long endTime = System.currentTimeMillis() - startTime;
            tvFlat.setText("Elements: " + reposListFlat.reposLength() + ": load time: " + endTime + "ms");

        }
    }

Methods which should interest us the most:

*   `parseReposListJson(String reposStr)` - this method initializes Gson parser and convert json String to Java objects.
*   `loadFlatBuffer(byte[] bytes)` - this method converts bytes (our repos_flat.bin file) to Java objects.

## Results

Now let’s visualize differences between JSON and FlatBuffers loading time and consumed resources. Tests are made on Nexus 5 with Android M (beta) installed.

## Loading time

Measured operation is conversion to Java files and iteration over all (90) elements.

JSON - 200ms (range: 180ms - 250ms) - average loading time of our JSON file (weight: 478kB) FlatBuffers - 5ms (range: 3ms - 10ms) - average loading time of FlatBuffers binary file (weight: 362kB)

Remember our [16ms rule](https://www.youtube.com/watch?v=CaMTIgxCSqU) ? We’re calling those method with a reason in UI thread. Take a look at how our interface would behave in this case:

### JSON loading

![JSON](http://frogermcs.github.io/images/17/json.gif "JSON")

### FlatBuffer loading

![FlatBuffers](http://frogermcs.github.io/images/17/flatbuffers.gif "FlatBuffers")

See the difference? Json loading freezes ProgressBar for a while, making our interface unpleasant (operation takes more than 16ms).

### Allocations, CPU etc.

Whant to measure more? Maybe it’s a good time to give a try to [Android Studio 1.3](http://android-developers.blogspot.com/2015/07/get-your-hands-on-android-studio-13.html) and new features like Allocation Tracker, Memory Viewer and Method Tracer.

## Source code

Full source code of described project is available on Github [repository](https://github.com/frogermcs/FlatBuffs). You don’t need to deal with FlatBuffers project - all you need is in `flatbuffers/` directory.

[Miroslaw Stanek](http://about.me/froger_mcs)  
_Head of Mobile Development_ @ [Azimo Money Transfer](https://azimo.com)

If you liked this post, you can [share it with your followers](https://twitter.com/intent/tweet?url=http://frogermcs.github.io/flatbuffers-in-android-introdution/&text=FlatBuffers%20in%20Android%20-%20introduction&via=froger_mcs) or [follow me on Twitter](https://twitter.com/froger_mcs)!



