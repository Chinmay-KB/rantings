# An easier way to work with Firebase Database in Flutter

> Disclaimer - This article is not about how to set up flutter. [firebase.flutter.dev](https://firebase.flutter.dev/) is a good place to learn about that. This article is about an issue I face personally while using Firebase Database, and what I do to make life a little easier!

## The Problem

Assuming you are a Flutter developer, you love dart for being strongly typed. But when we are dealing with Firebase Database, it returns a `Map<String, dynamic>` object. There are quite a few downsides to it.

1. We lose type safety- We may push data of an unintended data type
2. No code completion- You need to type all the keys of the map. Life's hell if the map is too nested :/

## The Solution

We add some order to the chaos! We map this response to an object, simple! Like how we treat an api response, similarly.

## The Demonstration

I have set up a firebase database with a single document, with fairly nested data.

![Data for demo](https://i.ibb.co/N659mMg/Screenshot-2021-06-28-214942.png)

From this document we want to fetch the `data` field of the first `achievement` of the first `friend`, and also the `name` of the first `friend`.

### Handle data the usual way

``` dart
Future<Map<String, dynamic>?> getDataTheUsualWay(String document) async {
final _data =
    (await firestoreInstance.collection('users').doc(document).get())
        .data();
return _data;
}
```
Inside `FutureBuilder`, this is handled in the following manner.
``` dart
...
FutureBuilder(
    future: getDataTheUsualWay('ENYMl6A76UTENuQ9fnOi'),
    builder:(BuildContext context,
        AsyncSnapshot<Map<String, dynamic>?> snapshot){
            return snapshot.connectionState != ConnectionState.done
        ? Center(child: CircularProgressIndicator())
        : Center(
            child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                crossAxisAlignment: CrossAxisAlignment.center,
                mainAxisSize: MainAxisSize.max,
                children: [
                Text(
                    'Data handled as how all other tutorials say '),
                Text(
                    'Name is ${snapshot.data!['people']['friends'][0]['name']}'),
                Text(snapshot.data!['people']['friends'][0]
                    ['achievements'][0]['data']),
                Text(snapshot.data!['people']['friends'][0]
                    ['achievements'][0]['type']),
                ],
            ),
            );
        }
)
```

You will be getting no help from the IDE whatsoever while typing `snapshot.data!['people'['friends'][0]['achievements'][0]['data']`. Any typos, or a wrong data type will result in an error. Because of which you will have to keep referring back to the database, hence slowing you down considerably.

### Handle data the better way.

Firebase has a method called `.withConverter()`. What this does is that instead of dealing in `Map<String, dynamic>`, this method will deal in terms of an object of a serializable class. Didn't get it? Just follow along!

First, we need a json of the document. We can get that by doing this 

``` dart
var firestoreInstance = FirebaseFirestore.instance;
final _data = await firestoreInstance
    .collection('users')
    .doc('*replae with document id*') 
    .get();
print(json.encode(_data.data()));
```
For my example the output was this json.

```json
{
   "achievements":[
      {
         "data":"Strong meme game",
         "type":"non-academic"
      },
      {
         "data":"Cleared multiple subjects without submitting assignments",
         "type":"academic"
      }
   ],
   "alive":true,
   "name":"Chinmay Kabi",
   "people":{
      "friends":[
         {
            "achievements":[
               {
                  "data":"Funny name",
                  "type":"non-academic"
               }
            ],
            "name":"Mr Thingumbob"
         }
      ]
   },
   "age":69
}
```

Now open [Quicktype](https://app.quicktype.io/). This website helps you generate serializable classes from json, such that you pass the json to a method and it returns an object such that the keys of the map are the properties of this object! This is how it will look
![quicktype screenshot](https://i.ibb.co/2527fFg/Screenshot-2021-06-28-221239.png)

Select dart from the options and it will generate the required code. Copy this code into a class, `user.dart` in my case.

After doing this, we can simply do this

```dart
Future<User?>? getDataTheBetterWay(String document) async {
    final _data = await firestoreInstance
        .collection('users')
        .doc(document)
        .withConverter<User>(
            fromFirestore: (snapshot, _) =>
                userFromJson(json.encode(snapshot.data())),
            toFirestore: (model, _) => json.decode(userToJson(model)))
        .get();
    return _data.data();
}
```

This returns a `User` object. Getting the name of this user is as simple as `user.name`, which the IDE will give you code suggestions for the moment you have typed `user.n`. We can display the same data we had shown before in this way.

```dart
FutureBuilder(
    future: getDataTheBetterWay('ENYMl6A76UTENuQ9fnOi'),
    builder: (BuildContext context, AsyncSnapshot<User?> snapshot) {
    return snapshot.connectionState != ConnectionState.done
        ? Center(child: CircularProgressIndicator())
        : Center(
            child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                crossAxisAlignment: CrossAxisAlignment.center,
                mainAxisSize: MainAxisSize.max,
                children: [
                Text('Data handled using my method'),
                Text(
                    'Name is ${snapshot.data!.people.friends[0].name}'),
                Text(snapshot
                    .data!.people.friends[0].achievements[0].data),
                Text(snapshot
                    .data!.people.friends[0].achievements[0].type),
                ],
            ),
            );
    },
),
```

Typing `snapshot.data!.people.friends[0].achievements[0].data` will be effortless for you because code suggestions! Also this ensures type safety, so a string cannot magically turn into a number(sorry JS people!). Such an arrangement is helpful when you have to refer a lot of fields of a document. But it only gets better!

### Updating data the usual way

Updating the values of a top level field is easy, but what about a nested field? We will be updating the `type` field of the first `achievements` of the first `friends`

```dart
  Future<void> updateDataTheUsualWay(String document, String data) async {
    final userRef = firestoreInstance.collection('users').doc(document);
    final userData = (await userRef.get()).data();
    var friendsList = userData!['people']['friends'];
    friendsList[0]['achievements'][0]['type'] = data;
    await userRef.update({
      "people": {"friends": friendsList}
    });
    print(data);
  }
```

There is no way to update a particular index of an array and push the update, we need to use the whole array, and pick out the fields we want to update. Then we push an update for the whole array. If that array is nested inside another field, or a map, or (god forbid) another list, we need to pass the reference for those nested fields, so that Firebase knows what to update. This is done my typing the map with the properties manually, maintaining the structure until we hit the array we have mutated, then we pass the mutated array. In this case, we have mutated `friends` array. We have a copy of the mutated array. But we cannot pass it directly. Because it is nested inside `people`, we need to pass this map to the `update()`.

```json
{
    "people":{
        "friends":friendsList
    }
}
```

This is just for a single level nesting. Imaging having multi level nesting, and updating multiple fields in multiple levels. Thankfully, you don't have to!

## Updating data the better way

We will be updating the same fields.

```dart
  Future<void> updateDataTheBetterWay(String document, User user) async {
    final userRef = firestoreInstance
        .collection('users')
        .doc(document)
        .withConverter<User>(
            fromFirestore: (snapshot, _) =>
                userFromJson(json.encode(snapshot.data())),
            toFirestore: (model, _) => model.toJson());
    await userRef.set(user);
  }
```

While calling this function, we can mutate the values of the properties as we feel like. So mutating the properties is something like this

``` dart
User? user =await getUserTheBetterWay('ENYMl6A76UTENuQ9fnOi');
user!.people.friends[0].achievements[0].type = data;
updateDataTheBetterWay('ENYMl6A76UTENuQ9fnOi', user);
```

Clean right?
Feel free to mutate as many parameters you want. Everything can be done easily in a type safe way, and code suggestions will help you make coding this a breeze!

Sad thing is that I have not come across `.withConverter()` in any of the tutorials for firebase database. I hope this article acts as a primer for this, and makes working with Firebase Database a bit easier. This is by no means an in-depth article about `.withConverter()`, so feel free to comment about anything you want me to cover more about this in another article!

You can find the sample project I have made with this code [here](https://github.com/Chinmay-KB/better_firebase_flutter).