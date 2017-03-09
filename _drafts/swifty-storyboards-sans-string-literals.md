---
layout: post
title:  "Swifty Storyboards Sans String Literals"
date:   2017-03-09 01:13:30 -0500
categories: storyboard string
---

## Avoiding Literals (But Not Literature)

Personally, I'm quite a fan of Interface Builder and `UIStoryboard`. I appreciate the help visualizing the layouts (especially across device sizes with Xcode 8) and it reduces the average size of my view controller files by putting layout and display properties into a separate XML file that treats them like content.

Of course there are some pitfalls when using Storyboards, especially on teams, but there are strategies one can employ to minimize the dangers. Not the least of these pitfalls is a naïve  reliance on string literals for linking the storyboard files to their associated implementation files. 

Each string literal used for this purpose is a liability—a weakness in the app's maintainability that is prone to human oversight and the occasional incorrectly-handled git merge conflict. Take for example, the Cocoa pattern for instantiating a view controller using a storyboard:

```swift
let loginSB = UIStoryboard(name: "Login", bundle: nil)
let loginVC = loginSB.instantiateViewController(withIdentifier: "LoginViewController")
```

What happens if we change the values behind either of these string literals—either the name of the storyboard (perhaps to `Onboarding`) or the name of the view controller (perhaps to `SignUpViewController`)—but fail to change any one occurrence of the string literal? We'll get a run time crash, and that's not very Swifty of us—tsk tsk tsk. It's also prone to slipping through behavioral testing and making its way into app store releases, especially when it's an alternate route to an obscure view controller deep within the app (trust me, I know from experience).

While Stan Otrovskiy wrote a [great blog post](https://medium.com/ios-os-x-development/xcode-a-better-way-to-deal-with-storyboards-8b6a8b504c06#.jd8svebek) explaining a 1:1 view-controller-to-storyboard ratio, I personally find this to be a little bit extreme and prefer to organize my layouts into a handful of storyboards by "flow". I like to see related view controllers next to each other without having to switch storyboard files. 

What constitutes a "flow" is a somewhat subjective matter in which root-view-controller classes like Tab Bar Controllers and Page View Controllers don't generally fit in with anything else, but I do find it works generally well for separating app components by navigation stack. 

## Don't Solve This With Segues

You might be thinking at this juncture that using segues naturally dodges the need to use `instantiateViewController(withIdentifier:)` and you would be correct. However, if there's any setup that we need to perform from the presenting view controller, we'd have provide an implementation using `prepare(for: UIStoryboardSegue, sender: Any?)` and in doing so we'd have to check the segue's identifier using...wait for it...yup, a **string literal**. This leaves us with a similar problem.

[Stan Otrovskiy](https://medium.com/ios-os-x-development/xcode-a-better-way-to-deal-with-storyboards-8b6a8b504c06#.jd8svebek), mentioned above, has similar feelings:

>* You need to name every segue, that alone is error-prone. Hard-coding the long string names is always a bad programming habit.
>* *PrepareForSegue* method will become ugly and non-readable, when you add a few segues using either “if/else” or “switch” statements.

For my part, I've come to believe that it's best to avoid mixing the use of segues and programmatic view controller presentation within the same project. It gets difficult to remember which has been used where and makes the code paths harder to follow as a reader. (The one exception to this being root- or embed-segues such as for `UINavigationController`s.)

Since I have an obvious bias for using programmatic view controller presentation, I have developed a two-point strategy for dodging this bullet in my projects:

1. extend `UIStoryboard` with class getters to encapsulate a single, necessary use of a string literal, and
2. write each view controller class with a constructor that uses the class's name at runtime to instantiate an instance.

## Extending UIStoryboard

This step is pretty straightforward: create a swift extension file named `UIStoryboard+Storyboards.swift` and write a class getter for each of your storyboard files. When you change a storyboard file's name, update this extension file and the compiler will highlight every occurrence of the old name throughout your application without even needing the Find tool.

Assuming we have a project with three storyboard files; `Login.storyboard`, `MyProfile.storyboard`, and `Chat.storyboard`; our extension could look like this:

```swift
//  UIStoryboard+Storyboards.swift

import UIKit

extension UIStoryboard {
    static var login: UIStoryboard {
        return UIStoryboard(name: "Login", bundle: nil)
    }
    
    static var myProfile: UIStoryboard {
        return UIStoryboard(name: "MyProfile", bundle: nil)
    }
    
    static var chat: UIStoryboard {
        return UIStoryboard(name: "Chat", bundle: nil)
    }
}
```

Now, elsewhere in our project we can use these class properties to access our storyboards without using string literals:

```swift
let loginSB = UIStoryboard.login
let loginVC = loginSB.instantiateViewController(withIdentifier: "LoginViewController")
```

With the added readability, we can more sensibly collapse these two lines into one:

```swift
let loginVC = UIStoryboard.login.instantiateViewController(withIdentifier: "LoginViewController")
```

### Changing a Storyboard's Name

With this extension in place (and used exclusively), we can now readily change the name of our `Login.storyboard` to `Onboarding.storyboard` with only the one encapsulated string literal to update.

```swift
static var onboarding: UIStoryboard {
    return UIStoryboard(name: "Onboarding", bundle: nil)
}
```

And the compiler will highlight our now-incorrect use of `UIStoryboard.login`, allowing us to change it to `UIStoryboard.onboarding`:

```swift
let loginVC = UIStoryboard.onboarding.instantiateViewController(withIdentifier: "LoginViewController")
```

Step one complete.

## View Controller Constructors

**Authorship Note:** *This section is very similar to the pattern in [Stan Otrovskiy's blog post from Sep 2016](https://medium.com/ios-os-x-development/xcode-a-better-way-to-deal-with-storyboards-8b6a8b504c06#.jd8svebek). While he wrote about it before I did, I developed my own pattern with multiple combined storyboards in an Objective-C project at a time previous to his publication date.*

Removing the usage of the view controller class name's string literal is a bit trickier but equally important: it requires us to encapsulate the call of `instantiateViewController(withIdentifier:)` into a class constructor so we can correctly utilize the `self` keyword in conjunction with `String(describing: self)` (or `NSStringFromClass([self class])` from an Objective-C file).

**Note:** *Use a class function that returns a new instance and not a static property. This will emphasize its nature as a constructor, and avoid the disastrous possibility that another programmer might unwittingly change your `static var` to a `static let`.*

Since we should avoid naming this constructor `init()` or `new()`, I prefer to name it `instance()`, and its return type will be the class we're in. In our `LoginViewController` class file, this will look like:

```swift
// LoginViewController.swift

class LoginViewController: UIViewController {
    class func instance() -> LoginViewController {
        return UIStoryboard.onboarding.instantiateViewController(withIdentifier: String(describing: self)) as! LoginViewController
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
    }
}
```

You'll notice that we explicitly cast the resulting `UIViewController` to that of our current class using `as!`. While repetitive, this avoids using a string literal. If we later change the name of the class, the compiler will complain until we update all occurrences of the old class name throughout the project.

#### Presenting Using the Constructor

Now instead of using `instantiateViewController(withIdentifier:)` from the presenting view controller, we can simply call our  `instance()` constructor to get a new instance:

```swift
let loginVC = LoginViewController.instance()
```

We can then setup any properties before presenting it. Alternatively, if we have no setup to perform, then we can pass this short code phrase directly into a presentation method quite succinctly:

```swift
self.present(viewController: LoginViewController.instance(), animated: true)
```
```swift
self.navigationController?.push(viewController: LoginViewController.instance(), animated: true)
```

#### Changing a View Controller's Class Name

With this constructor now employed, if we wish to change the view controller's class name we'll have no string literals to hunt down in our project's implementation files. Make sure you update the storyboard canvas's Class Name **and Storyboard ID** when you do so (along with the `*.swift` filename too), but otherwise the compiler will help us hunt down all the occurrences of the old class name throughout the project.

For example, renaming our `LoginViewController` to `SignUpViewController` simply requires us to edit the class name and constructor like so:

```swift
// SignUpViewController.swift

class SignUpViewController: UIViewController {
    class func instance() -> SignUpViewController {
        return UIStoryboard.onboarding.instantiateViewController(withIdentifier: String(describing: self)) as! SignUpViewController
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
    }
}
```

And then the compiler will highlight the points at which we used the constructor, which can update with the new class name:

```swift
let signUpVC = SignUpViewController.instance()
```
```swift
self.present(viewController: SignUpViewController.instance(), animated: true)
```
```swift
self.navigationController?.push(viewController: SignUpViewController.instance(), animated: true)
```

### Moving View Controllers Between Storyboards

In this multiple combined-storyboard approach, it's also feasible that the app will get rearranged at some point, with some view controllers being moved into different (or altogether new) storyboard files. With this pattern correctly in place, changing the storyboard in the implementation files is as simple as updating which `UIStoryboard` getter is called in the `instance()` constructor.

For example, if we have as part of our Onboarding flow an `AvatarImageSelectorViewController` class for selecting a profile picture:

```swift
// AvatarImageSelectorViewController.swift

class AvatarImageSelectorViewController: UIViewController {
    class func instance() -> AvatarImageSelectorViewController {
        return UIStoryboard.onboarding.instantiateViewController(withIdentifier: String(describing: self)) as! AvatarImageSelectorViewController
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
    }
}
```

If a user experience redesign moves this to an optional step in the MyProfile flow, we can adapt to this quite readily. Once we have moved the storyboard canvas from `Onboarding.storyboard` to `MyProfile.storyboard`, we simply replace the `instance()` constructor's usage of `UIStoryboard.onboarding` with `UIStoryboard.myProfile` and the project should build just fine.

```swift
// AvatarImageSelectorViewController.swift

class AvatarImageSelectorViewController: UIViewController {
    class func instance() -> AvatarImageSelectorViewController {
        return UIStoryboard.myProfile.instantiateViewController(withIdentifier: String(describing: self)) as! AvatarImageSelectorViewController
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
    }
}
```

Once the presentation logic is updated appropriately, the migration is done.

## Summary

Employing these two patterns throughout our project allows us to avoid the use of string literals in every file except for the one `UIStoryboard+Storyboards.swift` extension. Any changes to our storyboard file names or view controller class names will be enforced by the compiler. The result of this will be a more maintainable codebase that is resilient both to human oversight and to incorrectly resolved git merge conflicts. And finally, it improves the readability of our code by reducing the instantiation logic to a simple code phrase containing the view controller's class name and the name of our constructor method `instance()`.