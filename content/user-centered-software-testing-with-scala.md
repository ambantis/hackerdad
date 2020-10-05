+++
title = "User-Centered Software Testing"
date = 2013-10-12
path = "blog/2013/10/12/user-centered-software-testing-with-scala"
+++

Today I want to talk a bit about how I reoriented my tests from a file-center to a user-centered perspective. I’ve been spending a lot of time thinking about testing lately.

My code base is still relatively small, just under 2k, but the past week or so I haven’t added any new features while I’ve been changing my focus from we’re-a-startup-and-there’s-no-time-to-test to a more sane approach. After all, building an application from scratch is both a sprint and a marathon.

I’ve settled on two testing styles that I like within the ScalaTest framework: FunSpec and FeatureSpec. FunSpec is for my unit testing and FeatureSpec is for my integration testing. Here’s what a simple Funspec looks like:

```scala
@DoNotDiscover
class DbUserTest extends FunSpec with Matchers with EitherValues
    with DbUserJson {

  describe("A DbUser") {
    describe("Should be instantiated from valid json") {
      validUserJson.validate[DbUser].left.get.email should be(
          "bart@simpsons.com")
    }
  }
}
```

The `@DoNotDiscover` annotation is because all of my tests are organized into suites, which helps keep the test output organized and clean, important since the tests are an important source of documentation and specification for my MagicNotebook app.

The integration testing … well, this morning I was looking at how I was organizing my tests, and they really were file-centric rather than user-centric. Fundamentally, the tests were asking whether isolated parts of the application (database, cache, & models, for example) were working together as expected. But the tests were organized around different files and mimicked the application file/folder structure. Here’s what it looked like when I woke up this morning:

```
 % tree test
test
├── integration
│   ├── IntegrationSuites.scala
│   └── models
│       ├── db
│       │   └── DbUserSpec.scala
│       └── google
│           └── GFileSpec.scala
├── resources
├── unit
│   ├── models
│   │   ├── db
│   │   │   └── DbUserTest.scala
│   │   └── google
│   │       └── GFolderTest.scala
│   └── UnitSuites.scala
└── utils
    ├── db
    │   ├── DbSetup.scala
    │   └── DbUserJson.scala
    ├── google
    └── MagicNotebookData.scala
```

So I decided to reorganize it around actual specifications that a user would understand. Here’s what it looks like now:

```
 % tree test
test
├── feature
│   ├── admin
│   │   └── AccountFeatures.scala
│   ├── explorer
│   │   └── ViewDriveFeatures.scala
│   └── FeatureSuites.scala
├── resources
├── unit
│   ├── models
│   │   ├── db
│   │   │   └── DbUserTest.scala
│   │   └── google
│   │       └── GFolderTest.scala
│   └── UnitSuites.scala
└── utils
    ├── db
    │   ├── DbSetup.scala
    │   └── DbUserJson.scala
    ├── google
    └── MagicNotebookData.scala
```

As you can see, the Integration tests are now organized around features, such as account management and views Google Drive file features. So, I’m on the cusp of breaking the piddling 2k loc metric, but I’m not in a rush to grow the code base because I want it to grow into a beautiful greenfield that will stay green and healthy, because I’m in it for the long haul, and I hope you are too!
