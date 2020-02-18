# The iOS Style Guide - Proposal

## Creating new screens in App

We use MVVM-C (Model View View Model with Coordinators) as our architecture with our own Coordinator implementation (which you can find documentation for in the main project's README)

There are no Storyboards in App projects - everything is done via code. While people have mixed opinions about this, we believe that view coding is the best option as it allows multiple people to code in the same screen at the same time, merge conflicts are simple and you gain more control over your screens, not to mention that you don't have to wait 10 minutes for Xcode to load the visual representation of your screen - everything is just regular code.

In addition, this allows use to use dependency injection in a straight forward manner. You'll see that there are no singletons in Rapiddo - everything relevant to a class must be directly sent to it. This allows us to write better tests and make sure that classes can only do what they are supposed to.

This section details how to create a new screen called `MyScreen`. Please note that we have a Sourcery template in the `/ClassGen` folder that you can use to generate all the files needed to create a new screen using our architecture.

In normal conditions, `MyScreen` would consist of:

### `MyScreenCoordinator`

In MVVM-C, The Coordinator is the object responsible for handling screen transitions. It retains its inner `UIViewController` and delegates it in order to know when to transition to another Coordinator.

```swift
import UIKit
import RapiddoCore

final class MyScreenCoordinator: Coordinator {
    init(client: HTTPClient, persistence: Persistence) {
        let viewModel = MyScreenViewModel(client: client, persistence: persistence)
        let viewController = MyScreenViewController(viewModel: viewModel)
        super.init(rootViewController: viewController)
        viewController.delegate = self
    }
}

extension MyScreenCoordinator: MyScreenViewControllerDelegate {
    func continue() {
        let coordinator = NextCoordinator()
        push(coordinator, animated: true)
    }
}
```

### `MyScreenViewModel`

The ViewModel handles a ViewController's business logic. Ideally, this is where API calls happen and where the data source is retained.

```swift
import RapiddoCore

final class MyScreenViewModel {

    let client: HTTPClient
    let persistence: Persistence
    
    var myData = [MyData]()

    init(client: HTTPClient, persistence: Persistence) {
        self.client = client
        self.persistence = persistence
    }
    
    func getMyData() {
        //some request
        //myData = the result
    }
}

```

### `MyScreenView`

The View is where everything regarding the visual aspect of the ViewController should be built and retained. Note that we are not in an `UIViewController`!. Instead of adding views to the ViewController, we use a separate view file and override the `UIViewController`'s default `view` property by overriding `loadView()`, as you will see below in the `MyScreenViewController` explanation.

```swift
import UIKit
import Cartography

protocol MyScreenViewDelegate: class {
    func somethingHappened()
}

final class MyScreenView: UIView {

    weak var delegate: MyScreenViewDelegate?

    private let aView: UIView = {
        //View setup
    }()

    private let anotherView: UIView = {
        //View setup
    }()

    override init(frame: CGRect) {
        super.init(frame: frame)
        setup()
    }

    required init?(coder aDecoder: NSCoder) {
        fatalError()
    }

    private func setup() {
        setupAView()
        setupAnotherView()
    }

    private func setupAView() {
        addSubview(aView)
        //Constraints
    }

    private func setupAnotherView() {
        addSubview(anotherView)
        //Constraints
    }
    
    func render() {
        //Update the view
    }
}
````

### `MyScreenViewController`

Finally, the ViewController wraps together the previous three classes. Usually, the ViewController doesn't do anything besides routing information between the parties (the exception being `UITableView` delegates).

Note that we use a protocol named `SmartViewController` in order to allow the ViewController to access the inner `MyScreenView` through a `smartView` property. You can read more about `loadView()` [here.](https://swiftrocks.com/writing-cleaner-view-code-by-overriding-loadview.html)

```swift
import UIKit
import AppCore

protocol MyScreenViewControllerDelegate: class {
    func continue()
}

final class MyScreenViewController: CoordenableViewController {
    weak var delegate: MyScreenViewControllerDelegate?

    let viewModel: MyScreenViewModel

    init(viewModel: MyScreenViewModel) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
        title = Localization.myScreenTitle
    }

    required init?(coder aDecoder: NSCoder) {
        fatalError()
    }

    override func loadView() {
        let myScreenView = MyScreenView()
        view = myScreenView
        myScreenView.delegate = self
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        viewModel.getSomething()
        smartView.render()
    }
}

extension MyScreenViewController: MyScreenViewDelegate {
    func someButtonTouched() {
        delegate?.continue()
    }
}

extension MyScreenViewController: SmartViewController {
    typealias SmartView = MyScreenView
}
```

# Style Guide

In order to maintain our status as a highly-scalable superapp, all Rapiddo code must follow these guidelines. This is not a complete list by any means, so feel free to make your own suggestions!

## Regarding Frameworks

Please do not import frameworks "just because". Try to do things natively, only importing frameworks if it means a massive improvement for every provider (such as `Cartography` and `PromiseKit`). Remember that this is not a regular app - if we imported frameworks for every little thing our binary would be in the gigabytes.

## Naming

Using descriptive names makes code easier to read and understand. Use the Swift naming conventions described in the [API Design Guidelines](https://swift.org/documentation/api-design-guidelines/). Also refer to the Clean Code's chapter on naming for more examples. Remember what was said at the Comment's section and be aware that property/parameter/method names should be enough documentation. Make sure that their purpose can be fully understood purely by reading its name.

# Views

The views on the project follow a very simple structure.

### Setup

The initial setup of a view should happen inside a `setup()` method called inside the `UIView`'s init.

```swift
override init(frame: CGRect) {}
    super.init(frame: frame)
    setup()
}
```

Every view should be built purely by code, with constraints handled by the using the `Cartography` framework. To setup the subviews of a view, additional  `setup()` methods should be implemented and called on the main `setup()` method of the view.

```swift
private func setupTableView() {
    addSubview(tableView)
    constrain(tableView, self) { view, superview in
        view.edges == superview.edges
    }
}
```

The constraints of each subview should be added inside the setup of that subview. It’s important to always remember to add the subview to it’s superview before adding any constraints and pay attention to the order on which the setup methods are called.

```swift
private func setup() {
    setupTableView()
    setupEmptyState()
    setupLoadingView()
    setupSegmentedControl()
}
```
Actions of buttons and other views are configured in the setup method of that view.

### Subview Creation

All subviews should be created and configured using closures. Any setup that doesn’t depend on state or dynamic information must be done inside the closure.

```swift
let confirmButton: RapiddoButton = {
    let button = RapiddoButton(style: .positive)
    button.setTitle(Localization.confirm, for: .normal)
    button.isEnabled = false
    return button
}()
```

### Rendering content on Views

To display updated information, a view should implement a `render(_:)` method that will be called by that view’s superview (observe that this could be, and often will be, a `ViewController`). Views that displays dynamic information (e.g. loading states related to server requests)  should have a nested `State` enum.

```swift
final class MyView: UIView {
    enum State {
        case loading
        case updated
        case error(Error)
    }
}
```

The render method should then handle these states. Here is an example:

```swift
func render(state: State) {
    switch state {
    case .loading:
        renderLoading()
    case .updated:
        renderUpdated()
    case let .error(error):
        renderError(error)
    }
}
```
# TableView & CollectionView

### Registering and Dequeueing

Rapiddo posesses abstracted versions of common cell methods for both `UITableViews` and `UICollectionViews`.

To register a cell type, all you have to do is call the `register(_:)` method:

```swift
tableView.register(MyTableViewCell.self)
// or
collectionView.register(MyCollectionView.self)
```
Registering of cells must also be done on the closure used to create the view.

In a similar fashion, dequeueing of cells is also just a matter of calling the respective methods:
```swift
tableView.dequeue(type: MyTableViewCell.self)
// or
collectionView.dequeue(type: MyCollectionView.self, for: indexPath)
```

### TableView/CollectionView Delegates

The delegate methods of both Collection and Table views must be implemented on the `ViewController` that has that view as a subview.

## Delegation Patterns

One of the most important and frequent patterns used in the the project is the `delegate` pattern. The communication between the different layers of the project (Views, ViewControllers, ViewModels and Coordinators) is made mostly by them. When naming delegate methods, keep in mind the following recommendations:
- If the method represent an action of the user on a component make this explicit on the name of the method, e.g.`userDidSelect(name: String)`.
- If the delegate belongs to a view that might be used with more than one instance of it at the same superview, then it’s ok to add a parameter identifying the view. E.g. `emptyStateView(_ view: EmptyStateView didSelectButton button: RapiddoButton)`. Otherwise, prefer not to.
- Use `touched` instead of `tapped` or `clicked` when referring to touch events.

## Components & Styles

Some specific components are used throughout the project. Let’s see them.

### RapiddoButton

All the buttons used on the project must be of type `RapiddoButton` in order to make sure that it follows the designated button styles of the app. Creating a new button is just a matter of choosing a style: 

```swift
let button = AppButton(style: .positive)
```

At the moment, `AppButton.Style` can be either `positve` or `neutral`. You should use `positive`  style when you want to draw the attention of the user to the action performed by the button and `neutral` when that is not the case. In extreme cases where the button does not match any of the current styles, you can pass a `nil` style and configure the button manually.

### AppLabel / Fonts

In order to make sure that the correct fonts are used througout App, every label in the project must be a `AppLabel`. Just like `AppButtons`, creating a new label is just a matter of picking a style:

```swift
let label = AppLabel(style: .title2)
```

`AppLabel.Style` is an enum covering all font sizes and weights used in Rapiddo. If you need to use a font outside the context of an `UILabel`, use the `defaultFont` property of the `AppLabel.Style` like shown below:

```swift
button.titleLabel?.font = AppLabel.Style.title2.defaultFont
```

In the extreme case where you are required to use a font that is not part of our pre-determined styles, you can use the `custom` family of styles.

```swift
AppLabel.Style.customBold(17).defaultFont
```

However, try to first talk to the designer to see if it's not possible to adapt such font to one of our pre-determined ones.

### EmptyStateView

App's `EmptyStateView`  has two main uses in the project. Display a friendly message on screens that exhibit data that doesn’t exist yet and display error messages related to failed requests. Alongside a message, the view may also have a button and/or an image.

The empty state view works with an `EmptyStateMode`. The mode defines all the visual information that will be displayed by the view. To create a new mode, extend `EmptyStateMode` and define a new static method. Here is an example:

```swift
static func noOrders() -> EmptyStateMode {
    return EmptyStateMode(image: nil, text: Localization.noOrdersEmptyState, hidesButton: true)
}
```

To handle the action of the button, conform to the `EmptyStateViewDelegate` protocol and implement the `emptyStateViewButtonTouched(for mode: EmptyStateMode)` method. If your view is capable of displaying several types of `EmptyStateModes`, you should use the `mode` property to tell them apart.

To display errors in general, you should use the global `EmptyStateMode.error(Error)` `EmptyStateMode`.

### Colors and Themes

To keep things organized and easy to maintain, we keep all colors, margins and some key dimensions on the `Style` struct.
`Style` has four nested structs:
- `Colors`: Contains all the colors used by the app.
- `Theme`: The theme struct works just like the colors one, but it tends to define concepts like `tintColor` and `darkBackground` instead. This is mostly used by the Providers that are also used outside of the app.
- `Margins`: The horizontal and vertical margins used by the app.
- `Dimensions`: Key dimensions, such as button heights.

All those components are part of the `RapiddoUtils` framework. Always use them if possible. If a color or margin is not available inside these structs, consider talking to the designer to see if it was an oversight. In some extreme cases, we can resort to hardcoded values.

## Assets & Strings

You should always use `SwiftGen` when referencing images and strings.

Rapiddo and most Provider's subprojects already contain a Run Script phase to generate reference files, so most of the times all you have to do is merely compile the project in order to update the reference files.

### Images

After adding an image to it's Provider's `.xcassets` and compiling, you can access it on the `Asset` struct.

```swift
imageView.image = Asset.emptyStatePlaceholder.image
```

### Strings

On a fast growing project like Rapiddo, keeping track of all the text displayed on the app can be quite challenging. Especially for i18n, it’s easy to have some lost strings inside the project. For that reason all the strings in the project are kept in the `Localizable.strings` file. Even though we currently support only the Portuguese language doing this from the beginning make things much easier when we decide to support other languages.

Another benefit of this is that we can use `SwiftGen` to create a static references to those strings, making the code cleaner and easier to maintain. Every time a new text should be added to the project, create a new entry on the `Localizable` file, e.g. `"PLEASE_TRY_AGAIN" = "Por favor tente novamente.";` and, just like with images, compile the project in order to generate a static refenrece. The property will be inside the `Localization` type of that project. Then all you have to do to use it is:

```swift
let message = Localization.pleaseTryAgain
```

One thing to keep in mind is that not all of the text that is displayed on the app is kept inside the app. A lot of them are sent to the app by the server.

Another thing is that the `SwiftGen` scripts are defined by each Provider! If you're trying to add a string to Marmotex, you have to run Marmotex's example project.

### Error Handling

The way errors are presented to users is a big deal to any mobile project. In App, this is no different. There are three ways to display errors in Rapiddo:

- Empty States
- Toast
- SmartMessages

While the error state of an `EmptyStateView` should be rendered directly on its view, all other types of errors should be presented by calling the current Coordinator's `presentError()` method.

### EmptyStates

As mentioned in the `EmptyState` section, you can use an `EmptyStateView` in order to block access to a view, either to warn that there's nothing there or to display an error.

### Toasts

If the error you recieved is not an `APIError` with an underlying `SmartMessage`, calling `presentError()` at your Coordinator will briefly display a small message view at the top of the screen. This is good when you need to display an error that doesn't need to block the user's screen, but note that if you're already rendering an `EmptyStateView` alongside this error, then displaying a toast in unnecessary. In these cases, you should call `presentError(onlyDisplaySmartMessages: true)` instead.

### SmartMessages

Application errors that require an action by the user or a explicit acknowledgment are displayed on special popups as `SmartMessages`. These errors get mapped as `APIErrors` and are returned by the server as a response of a request. Normally smart messages carry at least one action represented by an action that should be handled by conforming your Coordinator to the `ConditionalActionHandler` protocol. See `RapiddoCore`'s `Action` and `RapiddoUtils`' `ActionHandler`/`ConditionalActionHandler` documentation for more details.

## Protocol Conformance

In particular, when adding protocol conformance to a model or view, prefer adding a separate extension for the protocol methods. This keeps the related methods grouped together with the protocol and can simplify instructions to add a protocol to a class.

```swift
extention MyModel: SomeProtocol {
    func someProtocolRequiredMethod() -> Int {
        return 10
    }
}
```

## Unused Code

Unused (dead) code should be removed. Don't worry about losing stuff, that's what Git is for :smile:

## Comments and Documentation

Use comments only to explain intent. Don't use comments to explain things that are already obvious, such as a protocol conformance. Code should be as self-documenting as possible, so if you feel the need to use comments to explain what the method itself is doing, consider refactoring it into something more clearer.

Bad comment:
```swift
//The tableView delegate
extension MyViewController: UITableViewDelegate {}
```

Good comment:
```swift
func loadData() {
    //We need to add a test header
    //because of a backend limitation.
    //They will fix this in the next release.
    client.add(header: "test", value: "true")
}
```

However, we do have an exception when it comes to documentation. In general, if the class you're building is supposed to be abstracted upon (which is the case of most `RapiddoCore` classes), then you should ignore these rules and document your code just like if you were building a framework (which is the case of `AppCore`! :smile: ) by using Swift's documentation formats.

If the class is not supposed to be abstracted upon, we think that using clear names is enough. There are exceptions, so talk to your team and see what they think about it.

## Closure Expressions

Use trailing closure syntax only if there's a single closure expression parameter at the end of the argument list. Give the closure parameters descriptive names.

When defining a closure that captures `self`, the unwrapped property should be named `strongSelf`.

```swift
foo.bar { [weak self] in
    guard let strongSelf = self else {
        return
    }
}
```

## Final

All classes or members of a class that are not meant to be overriden should be marked as `final`.

# Swift Style Guide

Make sure to read [Apple's API Design Guidelines](https://swift.org/documentation/api-design-guidelines/).

Specifics from these guidelines + additional remarks are mentioned below.

This guide was last updated for Swift 5.1 on February 17, 2020.

## Table Of Contents

- [Swift Style Guide](#swift-style-guide)
    - [1. Code Formatting](#1-code-formatting)
    - [2. Naming](#2-naming)
    - [3. Coding Style](#3-coding-style)
        - [3.1 General](#31-general)
        - [3.2 Access Modifiers](#32-access-modifiers)
        - [3.3 Custom Operators](#33-custom-operators)
        - [3.4 Switch Statements and `enum`s](#34-switch-statements-and-enums)
        - [3.5 Optionals](#35-optionals)
        - [3.6 Protocols](#36-protocols)
        - [3.7 Properties](#37-properties)
        - [3.8 Closures](#38-closures)
        - [3.9 Arrays](#39-arrays)
        - [3.10 Error Handling](#310-error-handling)
        - [3.11 Using `guard` Statements](#311-using-guard-statements)
    - [4. Documentation/Comments](#4-documentationcomments)
        - [4.1 Documentation](#41-documentation)
        - [4.2 Other Commenting Guidelines](#42-other-commenting-guidelines)

## 1. Code Formatting

* **1.1** Use 4 spaces for tabs.
* **1.2** Avoid uncomfortably long lines with a hard maximum of 160 characters per line (Xcode->Preferences->Text Editing->Page guide at column: 160 is helpful for this)
* **1.3** Ensure that there is a newline at the end of every file.
* **1.4** Ensure that there is no trailing whitespace anywhere (Xcode->Preferences->Text Editing->Automatically trim trailing whitespace + Including whitespace-only lines).
* **1.5** Do not place opening braces on new lines - we use the [1TBS style](https://en.m.wikipedia.org/wiki/Indentation_style#1TBS).

```swift
class SomeClass {
    func someMethod() {
        if x == y {
            /* ... */
        } else if x == z {
            /* ... */
        } else {
            /* ... */
        }
    }

    /* ... */
}
```

* **1.6** When writing a type for a property, constant, variable, a key for a dictionary, a function argument, a protocol conformance, or a superclass, don't add a space before the colon.

```swift
// specifying type
let pirateViewController: PirateViewController

// dictionary syntax (note that we left-align as opposed to aligning colons)
let ninjaDictionary: [String: AnyObject] = [
    "fightLikeDairyFarmer": false,
    "disgusting": true
]

// declaring a function
func myFunction<T, U: SomeProtocol>(firstArgument: U, secondArgument: T) where T.RelatedType == U {
    /* ... */
}

// calling a function
someFunction(someArgument: "Kitten")

// superclasses
class PirateViewController: UIViewController {
    /* ... */
}

// protocols
extension PirateViewController: UITableViewDataSource {
    /* ... */
}
```

* **1.7** In general, there should be a space following a comma.

```swift
let myArray = [1, 2, 3, 4, 5]
```

* **1.8** There should be a space before and after a binary operator such as `+`, `==`, or `->`. There should also not be a space after a `(` and before a `)`.

```swift
let myValue = 20 + (30 / 2) * 3
if 1 + 1 == 3 {
    fatalError("The universe is broken.")
}
func pancake(with syrup: Syrup) -> Pancake {
    /* ... */
}
```

* **1.9** We follow Xcode's recommended indentation style (i.e. your code should not change if CTRL-I is pressed). When declaring a function that spans multiple lines, prefer using that syntax to which Xcode, as of version 7.3, defaults.

```swift
// Xcode indentation for a function declaration that spans multiple lines
func myFunctionWithManyParameters(parameterOne: String,
                                  parameterTwo: String,
                                  parameterThree: String) {
    // Xcode indents to here for this kind of statement
    print("\(parameterOne) \(parameterTwo) \(parameterThree)")
}

// Xcode indentation for a multi-line `if` statement
if myFirstValue > (mySecondValue + myThirdValue)
    && myFourthValue == .someEnumValue {

    // Xcode indents to here for this kind of statement
    print("Hello, World!")
}
```

* **1.10** When calling a function that has many parameters, put each argument on a separate line with a single extra indentation.

```swift
someFunctionWithManyArguments(
    firstArgument: "Hello, I am a string",
    secondArgument: resultFromSomeFunction(),
    thirdArgument: someOtherLocalProperty)
```

* **1.11** When dealing with an implicit array or dictionary large enough to warrant splitting it into multiple lines, treat the `[` and `]` as if they were braces in a method, `if` statement, etc. Closures in a method should be treated similarly.

```swift
someFunctionWithABunchOfArguments(
    someStringArgument: "hello I am a string",
    someArrayArgument: [
        "dadada daaaa daaaa dadada daaaa daaaa dadada daaaa daaaa",
        "string one is crazy - what is it thinking?"
    ],
    someDictionaryArgument: [
        "dictionary key 1": "some value 1, but also some more text here",
        "dictionary key 2": "some value 2"
    ],
    someClosure: { parameter1 in
        print(parameter1)
    })
```

* **1.12** Prefer using local constants or other mitigation techniques to avoid multi-line predicates where possible.

```swift
// PREFERRED
let firstCondition = x == firstReallyReallyLongPredicateFunction()
let secondCondition = y == secondReallyReallyLongPredicateFunction()
let thirdCondition = z == thirdReallyReallyLongPredicateFunction()
if firstCondition && secondCondition && thirdCondition {
    // do something
}

// NOT PREFERRED
if x == firstReallyReallyLongPredicateFunction()
    && y == secondReallyReallyLongPredicateFunction()
    && z == thirdReallyReallyLongPredicateFunction() {
    // do something
}
```

## 2. Naming

* **2.1** There is no need for Objective-C style prefixing in Swift (e.g. use just `GuybrushThreepwood` instead of `LIGuybrushThreepwood`).

* **2.2** Use `PascalCase` for type names (e.g. `struct`, `enum`, `class`, `typedef`, `associatedtype`, etc.).

* **2.3** Use `camelCase` (initial lowercase letter) for function, method, property, constant, variable, argument names, enum cases, etc.

* **2.4** When dealing with an acronym or other name that is usually written in all caps, actually use all caps in any names that use this in code. The exception is if this word is at the start of a name that needs to start with lowercase - in this case, use all lowercase for the acronym.

```swift
// "HTML" is at the start of a constant name, so we use lowercase "html"
let htmlBodyContent: String = "<p>Hello, World!</p>"
// Prefer using ID to Id
let profileID: Int = 1
// Prefer URLFinder to UrlFinder
class URLFinder {
    /* ... */
}
```

* **2.5** All constants that are instance-independent should be `static`. All such `static` constants should be placed in a marked section of their `class`, `struct`, or `enum`. For classes with many constants, you should group constants that have similar or the same prefixes, suffixes and/or use cases.

```swift
// PREFERRED    
class MyClassName {
    // MARK: - Constants
    static let buttonPadding: CGFloat = 20.0
    static let indianaPi = 3
    static let shared = MyClassName()
}

// NOT PREFERRED
class MyClassName {
    // Don't use `k`-prefix
    static let kButtonPadding: CGFloat = 20.0

    // Don't namespace constants
    enum Constant {
        static let indianaPi = 3
    }
}
```

* **2.6** For generics and associated types, use a `PascalCase` word that describes the generic. If this word clashes with a protocol that it conforms to or a superclass that it subclasses, you can append a `Type` suffix to the associated type or generic name.

```swift
class SomeClass<Model> { /* ... */ }
protocol Modelable {
    associatedtype Model
}
protocol Sequence {
    associatedtype IteratorType: Iterator
}
```

* **2.7** Names should be descriptive and unambiguous.

```swift
// PREFERRED
class RoundAnimatingButton: UIButton { /* ... */ }

// NOT PREFERRED
class CustomButton: UIButton { /* ... */ }
```

* **2.8** Do not abbreviate, use shortened names, or single letter names.

```swift
// PREFERRED
class RoundAnimatingButton: UIButton {
    let animationDuration: NSTimeInterval

    func startAnimating() {
        let firstSubview = subviews.first
    }

}

// NOT PREFERRED
class RoundAnimating: UIButton {
    let aniDur: NSTimeInterval

    func srtAnmating() {
        let v = subviews.first
    }
}
```

* **2.9** Include type information in constant or variable names when it is not obvious otherwise.

```swift
// PREFERRED
class ConnectionTableViewCell: UITableViewCell {
    let personImageView: UIImageView

    let animationDuration: TimeInterval

    // it is ok not to include string in the ivar name here because it's obvious
    // that it's a string from the property name
    let firstName: String

    // though not preferred, it is OK to use `Controller` instead of `ViewController`
    let popupController: UIViewController
    let popupViewController: UIViewController

    // when working with a subclass of `UIViewController` such as a table view
    // controller, collection view controller, split view controller, etc.,
    // fully indicate the type in the name.
    let popupTableViewController: UITableViewController

    // when working with outlets, make sure to specify the outlet type in the
    // property name.
    @IBOutlet weak var submitButton: UIButton!
    @IBOutlet weak var emailTextField: UITextField!
    @IBOutlet weak var nameLabel: UILabel!

}

// NOT PREFERRED
class ConnectionTableViewCell: UITableViewCell {
    // this isn't a `UIImage`, so shouldn't be called image
    // use personImageView instead
    let personImage: UIImageView

    // this isn't a `String`, so it should be `textLabel`
    let text: UILabel

    // `animation` is not clearly a time interval
    // use `animationDuration` or `animationTimeInterval` instead
    let animation: TimeInterval

    // this is not obviously a `String`
    // use `transitionText` or `transitionString` instead
    let transition: String

    // this is a view controller - not a view
    let popupView: UIViewController

    // as mentioned previously, we don't want to use abbreviations, so don't use
    // `VC` instead of `ViewController`
    let popupVC: UIViewController

    // even though this is still technically a `UIViewController`, this property
    // should indicate that we are working with a *Table* View Controller
    let popupViewController: UITableViewController

    // for the sake of consistency, we should put the type name at the end of the
    // property name and not at the start
    @IBOutlet weak var btnSubmit: UIButton!
    @IBOutlet weak var buttonSubmit: UIButton!

    // we should always have a type in the property name when dealing with outlets
    // for example, here, we should have `firstNameLabel` instead
    @IBOutlet weak var firstName: UILabel!
}
```

* **2.10** When naming function arguments, make sure that the function can be read easily to understand the purpose of each argument.

* **2.11** As per [Apple's API Design Guidelines](https://swift.org/documentation/api-design-guidelines/), a `protocol` should be named as nouns if they describe what something is doing (e.g. `Collection`) and using the suffixes `able`, `ible`, or `ing` if it describes a capability (e.g. `Equatable`, `ProgressReporting`). If neither of those options makes sense for your use case, you can add a `Protocol` suffix to the protocol's name as well. Some example `protocol`s are below.

```swift
// here, the name is a noun that describes what the protocol does
protocol TableViewSectionProvider {
    func rowHeight(at row: Int) -> CGFloat
    var numberOfRows: Int { get }
    /* ... */
}

// here, the protocol is a capability, and we name it appropriately
protocol Loggable {
    func logCurrentState()
    /* ... */
}

// suppose we have an `InputTextView` class, but we also want a protocol
// to generalize some of the functionality - it might be appropriate to
// use the `Protocol` suffix here
protocol InputTextViewProtocol {
    func sendTrackingEvent()
    func inputText() -> String
    /* ... */
}
```

## 3. Coding Style

### 3.1 General

* **3.1.1** Prefer `let` to `var` whenever possible.

* **3.1.2** Prefer the composition of `map`, `filter`, `reduce`, etc. over iterating when transforming from one collection to another. Make sure to avoid using closures that have side effects when using these methods.

```swift
// PREFERRED
let stringOfInts = [1, 2, 3].flatMap { String($0) }
// ["1", "2", "3"]

// NOT PREFERRED
var stringOfInts: [String] = []
for integer in [1, 2, 3] {
    stringOfInts.append(String(integer))
}

// PREFERRED
let evenNumbers = [4, 8, 15, 16, 23, 42].filter { $0 % 2 == 0 }
// [4, 8, 16, 42]

// NOT PREFERRED
var evenNumbers: [Int] = []
for integer in [4, 8, 15, 16, 23, 42] {
    if integer % 2 == 0 {
        evenNumbers.append(integer)
    }
}
```

* **3.1.3** Prefer not declaring types for constants or variables if they can be inferred anyway.

* **3.1.4** If a function returns multiple values, prefer returning a tuple to using `inout` arguments (it’s best to use labeled tuples for clarity on what you’re returning if it is not otherwise obvious). If you use a certain tuple more than once, consider using a `typealias`. If you’re returning 3 or more items in a tuple, consider using a `struct` or `class` instead.

```swift
func pirateName() -> (firstName: String, lastName: String) {
    return ("Guybrush", "Threepwood")
}

let name = pirateName()
let firstName = name.firstName
let lastName = name.lastName
```

* **3.1.5** Be wary of retain cycles when creating delegates/protocols for your classes; typically, these properties should be declared `weak`.

* **3.1.6** Be careful when calling `self` directly from an escaping closure as this can cause a retain cycle - use a [capture list](https://developer.apple.com/library/ios/documentation/swift/conceptual/Swift_Programming_Language/Closures.html#//apple_ref/doc/uid/TP40014097-CH11-XID_163) when this might be the case:

```swift
myFunctionWithEscapingClosure() { [weak self] (error) -> Void in
    // you can do this

    self?.doSomething()

    // or you can do this

    guard let strongSelf = self else {
        return
    }

    strongSelf.doSomething()
}
```

* **3.1.7** Don't use labeled breaks.

* **3.1.8** Don't place parentheses around control flow predicates.

```swift
// PREFERRED
if x == y {
    /* ... */
}

// NOT PREFERRED
if (x == y) {
    /* ... */
}
```

* **3.1.9** Avoid writing out an `enum` type where possible - use shorthand.

```swift
// PREFERRED
imageView.setImageWithURL(url, type: .person)

// NOT PREFERRED
imageView.setImageWithURL(url, type: AsyncImageView.Type.person)
```

* **3.1.10** Don’t use shorthand for class methods since it is generally more difficult to infer the context from class methods as opposed to `enum`s.

```swift
// PREFERRED
imageView.backgroundColor = UIColor.white

// NOT PREFERRED
imageView.backgroundColor = .white
```

* **3.1.11** Prefer not writing `self.` unless it is required.

* **3.1.12** When writing methods, keep in mind whether the method is intended to be overridden or not. If not, mark it as `final`, though keep in mind that this will prevent the method from being overwritten for testing purposes. In general, `final` methods result in improved compilation times, so it is good to use this when applicable. Be particularly careful, however, when applying the `final` keyword in a library since it is non-trivial to change something to be non-`final` in a library as opposed to have changing something to be non-`final` in your local project.

* **3.1.13** When using a statement such as `else`, `catch`, etc. that follows a block, put this keyword on the same line as the block. Again, we are following the [1TBS style](https://en.m.wikipedia.org/wiki/Indentation_style#1TBS) here. Example `if`/`else` and `do`/`catch` code is below.

```swift
if someBoolean {
    // do something
} else {
    // do something else
}

do {
    let fileContents = try readFile("filename.txt")
} catch {
    print(error)
}
```

* **3.1.14** Prefer `static` to `class` when declaring a function or property that is associated with a class as opposed to an instance of that class. Only use `class` if you specifically need the functionality of overriding that function or property in a subclass, though consider using a `protocol` to achieve this instead.

* **3.1.15** If you have a function that takes no arguments, has no side effects, and returns some object or value, prefer using a computed property instead.

### 3.2 Access Modifiers

* **3.2.1** Write the access modifier keyword first if it is needed.

```swift
// PREFERRED
private static let myPrivateNumber: Int

// NOT PREFERRED
static private let myPrivateNumber: Int
```

* **3.2.2** The access modifier keyword should not be on a line by itself - keep it inline with what it is describing.

```swift
// PREFERRED
open class Pirate {
    /* ... */
}

// NOT PREFERRED
open
class Pirate {
    /* ... */
}
```

* **3.2.3** In general, do not write the `internal` access modifier keyword since it is the default.

* **3.2.4** If a property needs to be accessed by unit tests, you will have to make it `internal` to use `@testable import ModuleName`. If a property *should* be private, but you declare it to be `internal` for the purposes of unit testing, make sure you add an appropriate bit of documentation commenting that explains this. You can make use of the `- warning:` markup syntax for clarity as shown below.

```swift
/**
 This property defines the pirate's name.
 - warning: Not `private` for `@testable`.
 */
let pirateName = "LeChuck"
```

* **3.2.5** Prefer `private` to `fileprivate` where possible.

* **3.2.6** When choosing between `public` and `open`, prefer `open` if you intend for something to be subclassable outside of a given module and `public` otherwise. Note that anything `internal` and above can be subclassed in tests by using `@testable import`, so this shouldn't be a reason to use `open`. In general, lean towards being a bit more liberal with using `open` when it comes to libraries, but a bit more conservative when it comes to modules in a codebase such as an app where it is easy to change things in multiple modules simultaneously.

### 3.3 Custom Operators

Prefer creating named functions to custom operators.

If you want to introduce a custom operator, make sure that you have a *very* good reason why you want to introduce a new operator into global scope as opposed to using some other construct.

You can override existing operators to support new types (especially `==`). However, your new definitions must preserve the semantics of the operator. For example, `==` must always test equality and return a boolean.

### 3.4 Switch Statements and `enum`s

* **3.4.1** When using a switch statement that has a finite set of possibilities (`enum`), do *NOT* include a `default` case. Instead, place unused cases at the bottom and use the `break` keyword to prevent execution.

* **3.4.2** Since `switch` cases in Swift break by default, do not include the `break` keyword if it is not needed.

* **3.4.3** The `case` statements should line up with the `switch` statement itself as per default Swift standards.

* **3.4.4** When defining a case that has an associated value, make sure that this value is appropriately labeled as opposed to just types (e.g. `case hunger(hungerLevel: Int)` instead of `case hunger(Int)`).

```swift
enum Problem {
    case attitude
    case hair
    case hunger(hungerLevel: Int)
}

func handleProblem(problem: Problem) {
    switch problem {
    case .attitude:
        print("At least I don't have a hair problem.")
    case .hair:
        print("Your barber didn't know when to stop.")
    case .hunger(let hungerLevel):
        print("The hunger level is \(hungerLevel).")
    }
}
```

* **3.4.5** Prefer lists of possibilities (e.g. `case 1, 2, 3:`) to using the `fallthrough` keyword where possible).

* **3.4.6** If you have a default case that shouldn't be reached, preferably throw an error (or handle it some other similar way such as asserting).

```swift
func handleDigit(_ digit: Int) throws {
    switch digit {
    case 0, 1, 2, 3, 4, 5, 6, 7, 8, 9:
        print("Yes, \(digit) is a digit!")
    default:
        throw Error(message: "The given number was not a digit.")
    }
}
```

### 3.5 Optionals

* **3.5.1** The only time you should be using implicitly unwrapped optionals is with `@IBOutlet`s. In every other case, it is better to use a non-optional or regular optional property. Yes, there are cases in which you can probably "guarantee" that the property will never be `nil` when used, but it is better to be safe and consistent. Similarly, don't use force unwraps.

* **3.5.2** Don't use `as!` or `try!`.

* **3.5.3** If you don't plan on actually using the value stored in an optional, but need to determine whether or not this value is `nil`, explicitly check this value against `nil` as opposed to using `if let` syntax.

```swift
// PREFERERED
if someOptional != nil {
    // do something
}

// NOT PREFERRED
if let _ = someOptional {
    // do something
}
```

* **3.5.4** Don't use `unowned`. You can think of `unowned` as somewhat of an equivalent of a `weak` property that is implicitly unwrapped (though `unowned` has slight performance improvements on account of completely ignoring reference counting). Since we don't ever want to have implicit unwraps, we similarly don't want `unowned` properties.

```swift
// PREFERRED
weak var parentViewController: UIViewController?

// NOT PREFERRED
weak var parentViewController: UIViewController!
unowned var parentViewController: UIViewController
```

* **3.5.5** When unwrapping optionals, use the same name for the unwrapped constant or variable where appropriate.

```swift
guard let myValue = myValue else {
    return
}
```

### 3.6 Protocols

When implementing protocols, there are two ways of organizing your code:

1. Using `// MARK:` comments to separate your protocol implementation from the rest of your code
2. Using an extension outside your `class`/`struct` implementation code, but in the same source file

Keep in mind that when using an extension, however, the methods in the extension can't be overridden by a subclass, which can make testing difficult. If this is a common use case, it might be better to stick with method #1 for consistency. Otherwise, method #2 allows for cleaner separation of concerns.

Even when using method #2, add `// MARK:` statements anyway for easier readability in Xcode's method/property/class/etc. list UI.

### 3.7 Properties

* **3.7.1** If making a read-only, computed property, provide the getter without the `get {}` around it.

```swift
var computedProperty: String {
    if someBool {
        return "I'm a mighty pirate!"
    }
    return "I'm selling these fine leather jackets."
}
```

* **3.7.2** When using `get {}`, `set {}`, `willSet`, and `didSet`, indent these blocks.
* **3.7.3** Though you can create a custom name for the new or old value for `willSet`/`didSet` and `set`, use the standard `newValue`/`oldValue` identifiers that are provided by default.

```swift
var storedProperty: String = "I'm selling these fine leather jackets." {
    willSet {
        print("will set to \(newValue)")
    }
    didSet {
        print("did set from \(oldValue) to \(storedProperty)")
    }
}

var computedProperty: String  {
    get {
        if someBool {
            return "I'm a mighty pirate!"
        }
        return storedProperty
    }
    set {
        storedProperty = newValue
    }
}
```

* **3.7.4** You can declare a singleton property as follows:

```swift
class PirateManager {
    static let shared = PirateManager()

    /* ... */
}
```

### 3.8 Closures

* **3.8.1** If the types of the parameters are obvious, it is OK to omit the type name, but being explicit is also OK. Sometimes readability is enhanced by adding clarifying detail and sometimes by taking repetitive parts away - use your best judgment and be consistent.

```swift
// omitting the type
doSomethingWithClosure() { response in
    print(response)
}

// explicit type
doSomethingWithClosure() { response: NSURLResponse in
    print(response)
}

// using shorthand in a map statement
[1, 2, 3].flatMap { String($0) }
```

* **3.8.2** If specifying a closure as a type, you don’t need to wrap it in parentheses unless it is required (e.g. if the type is optional or the closure is within another closure). Always wrap the arguments in the closure in a set of parentheses - use `()` to indicate no arguments and use `Void` to indicate that nothing is returned.

```swift
let completionBlock: (Bool) -> Void = { (success) in
    print("Success? \(success)")
}

let completionBlock: () -> Void = {
    print("Completed!")
}

let completionBlock: (() -> Void)? = nil
```

* **3.8.3** Keep parameter names on same line as the opening brace for closures when possible without too much horizontal overflow (i.e. ensure lines are less than 160 characters).

* **3.8.4** Use trailing closure syntax unless the meaning of the closure is not obvious without the parameter name (an example of this could be if a method has parameters for success and failure closures).

```swift
// trailing closure
doSomething(1.0) { (parameter1) in
    print("Parameter 1 is \(parameter1)")
}

// no trailing closure
doSomething(1.0, success: { (parameter1) in
    print("Success with \(parameter1)")
}, failure: { (parameter1) in
    print("Failure with \(parameter1)")
})
```

### 3.9 Arrays

* **3.9.1** In general, avoid accessing an array directly with subscripts. When possible, use accessors such as `.first` or `.last`, which are optional and won’t crash. Prefer using a `for item in items` syntax when possible as opposed to something like `for i in 0 ..< items.count`. If you need to access an array subscript directly, make sure to do proper bounds checking. You can use `for (index, value) in items.enumerated()` to get both the index and the value.

* **3.9.2** Never use the `+=` or `+` operator to append/concatenate to arrays. Instead, use `.append()` or `.append(contentsOf:)` as these are far more performant (at least with respect to compilation) in Swift's current state. If you are declaring an array that is based on other arrays and want to keep it immutable, instead of `let myNewArray = arr1 + arr2`, use `let myNewArray = [arr1, arr2].joined()`.

### 3.10 Error Handling

Suppose a function `myFunction` is supposed to return a `String`, however, at some point it can run into an error. A common approach is to have this function return an optional `String?` where we return `nil` if something went wrong.

Example:

```swift
func readFile(named filename: String) -> String? {
    guard let file = openFile(named: filename) else {
        return nil
    }

    let fileContents = file.read()
    file.close()
    return fileContents
}

func printSomeFile() {
    let filename = "somefile.txt"
    guard let fileContents = readFile(named: filename) else {
        print("Unable to open file \(filename).")
        return
    }
    print(fileContents)
}
```

Instead, we should be using Swift's `try`/`catch` behavior when it is appropriate to know the reason for the failure.

You can use a `struct` such as the following:

```swift
struct Error: Swift.Error {
    public let file: StaticString
    public let function: StaticString
    public let line: UInt
    public let message: String

    public init(message: String, file: StaticString = #file, function: StaticString = #function, line: UInt = #line) {
        self.file = file
        self.function = function
        self.line = line
        self.message = message
    }
}
```

Example usage:

```swift
func readFile(named filename: String) throws -> String {
    guard let file = openFile(named: filename) else {
        throw Error(message: "Unable to open file named \(filename).")
    }

    let fileContents = file.read()
    file.close()
    return fileContents
}

func printSomeFile() {
    do {
        let fileContents = try readFile(named: filename)
        print(fileContents)
    } catch {
        print(error)
    }
}
```

There are some exceptions in which it does make sense to use an optional as opposed to error handling. When the result should *semantically* potentially be `nil` as opposed to something going wrong while retrieving the result, it makes sense to return an optional instead of using error handling.

In general, if a method can "fail", and the reason for the failure is not immediately obvious if using an optional return type, it probably makes sense for the method to throw an error.

### 3.11 Using `guard` Statements

* **3.11.1** In general, we prefer to use an "early return" strategy where applicable as opposed to nesting code in `if` statements. Using `guard` statements for this use-case is often helpful and can improve the readability of the code.

```swift
// PREFERRED
func eatDoughnut(at index: Int) {
    guard index >= 0 && index < doughnuts.count else {
        // return early because the index is out of bounds
        return
    }

    let doughnut = doughnuts[index]
    eat(doughnut)
}

// NOT PREFERRED
func eatDoughnut(at index: Int) {
    if index >= 0 && index < doughnuts.count {
        let doughnut = doughnuts[index]
        eat(doughnut)
    }
}
```

* **3.11.2** When unwrapping optionals, prefer `guard` statements as opposed to `if` statements to decrease the amount of nested indentation in your code.

```swift
// PREFERRED
guard let monkeyIsland = monkeyIsland else {
    return
}
bookVacation(on: monkeyIsland)
bragAboutVacation(at: monkeyIsland)

// NOT PREFERRED
if let monkeyIsland = monkeyIsland {
    bookVacation(on: monkeyIsland)
    bragAboutVacation(at: monkeyIsland)
}

// EVEN LESS PREFERRED
if monkeyIsland == nil {
    return
}
bookVacation(on: monkeyIsland!)
bragAboutVacation(at: monkeyIsland!)
```

* **3.11.3** When deciding between using an `if` statement or a `guard` statement when unwrapping optionals is *not* involved, the most important thing to keep in mind is the readability of the code. There are many possible cases here, such as depending on two different booleans, a complicated logical statement involving multiple comparisons, etc., so in general, use your best judgement to write code that is readable and consistent. If you are unsure whether `guard` or `if` is more readable or they seem equally readable, prefer using `guard`.

```swift
// an `if` statement is readable here
if operationFailed {
    return
}

// a `guard` statement is readable here
guard isSuccessful else {
    return
}

// double negative logic like this can get hard to read - i.e. don't do this
guard !operationFailed else {
    return
}
```

* **3.11.4** If choosing between two different states, it makes more sense to use an `if` statement as opposed to a `guard` statement.

```swift
// PREFERRED
if isFriendly {
    print("Hello, nice to meet you!")
} else {
    print("You have the manners of a beggar.")
}

// NOT PREFERRED
guard isFriendly else {
    print("You have the manners of a beggar.")
    return
}

print("Hello, nice to meet you!")
```

* **3.11.5** You should also use `guard` only if a failure should result in exiting the current context. Below is an example in which it makes more sense to use two `if` statements instead of using two `guard`s - we have two unrelated conditions that should not block one another.

```swift
if let monkeyIsland = monkeyIsland {
    bookVacation(onIsland: monkeyIsland)
}

if let woodchuck = woodchuck, canChuckWood(woodchuck) {
    woodchuck.chuckWood()
}
```

* **3.11.6** Often, we can run into a situation in which we need to unwrap multiple optionals using `guard` statements. In general, combine unwraps into a single `guard` statement if handling the failure of each unwrap is identical (e.g. just a `return`, `break`, `continue`, `throw`, or some other `@noescape`).

```swift
// combined because we just return
guard let thingOne = thingOne,
    let thingTwo = thingTwo,
    let thingThree = thingThree else {
    return
}

// separate statements because we handle a specific error in each case
guard let thingOne = thingOne else {
    throw Error(message: "Unwrapping thingOne failed.")
}

guard let thingTwo = thingTwo else {
    throw Error(message: "Unwrapping thingTwo failed.")
}

guard let thingThree = thingThree else {
    throw Error(message: "Unwrapping thingThree failed.")
}
```

* **3.11.7** Don’t use one-liners for `guard` statements.


```swift
// PREFERRED
guard let thingOne = thingOne else {
    return
}

// NOT PREFERRED
guard let thingOne = thingOne else { return }
```

## 4. Documentation/Comments

### 4.1 Documentation

If a function is more complicated than a simple O(1) operation, you should generally consider adding a doc comment for the function since there could be some information that the method signature does not make immediately obvious. If there are any quirks to the way that something was implemented, whether technically interesting, tricky, not obvious, etc., this should be documented. Documentation should be added for complex classes/structs/enums/protocols and properties. All `public` functions/classes/properties/constants/structs/enums/protocols/etc. should be documented as well (provided, again, that their signature/name does not make their meaning/functionality immediately obvious).

After writing a doc comment, you should option click the function/property/class/etc. to make sure that everything is formatted correctly.

Be sure to check out the full set of features available in Swift's comment markup [described in Apple's Documentation](https://developer.apple.com/library/tvos/documentation/Xcode/Reference/xcode_markup_formatting_ref/Attention.html#//apple_ref/doc/uid/TP40016497-CH29-SW1).

Guidelines:

* **4.1.1** 160 character column limit (like the rest of the code).

* **4.1.2** Even if the doc comment takes up one line, use block (`/** */`).

* **4.1.3** Do not prefix each additional line with a `*`.

* **4.1.4** Use the new `- parameter` syntax as opposed to the old `:param:` syntax (make sure to use lower case `parameter` and not `Parameter`). Option-click on a method you wrote to make sure the quick help looks correct.

```swift
class Human {
    /**
     This method feeds a certain food to a person.

     - parameter food: The food you want to be eaten.
     - parameter person: The person who should eat the food.
     - returns: True if the food was eaten by the person; false otherwise.
    */
    func feed(_ food: Food, to person: Human) -> Bool {
        // ...
    }
}
```

* **4.1.5** If you’re going to be documenting the parameters/returns/throws of a method, document all of them, even if some of the documentation ends up being somewhat repetitive (this is preferable to having the documentation look incomplete). Sometimes, if only a single parameter warrants documentation, it might be better to just mention it in the description instead.

* **4.1.6** For complicated classes, describe the usage of the class with some potential examples as seems appropriate. Remember that markdown syntax is valid in Swift's comment docs. Newlines, lists, etc. are therefore appropriate.

```swift
/**
 ## Feature Support

 This class does some awesome things. It supports:

 - Feature 1
 - Feature 2
 - Feature 3

 ## Examples

 Here is an example use case indented by four spaces because that indicates a
 code block:

     let myAwesomeThing = MyAwesomeClass()
     myAwesomeThing.makeMoney()

 ## Warnings

 There are some things you should be careful of:

 1. Thing one
 2. Thing two
 3. Thing three
 */
class MyAwesomeClass {
    /* ... */
}
```

* **4.1.7** When mentioning code, use code ticks - \`

```swift
/**
 This does something with a `UIViewController`, perchance.
 - warning: Make sure that `someValue` is `true` before running this function.
 */
func myFunction() {
    /* ... */
}
```

* **4.1.8** When writing doc comments, prefer brevity where possible.

### 4.2 Other Commenting Guidelines

* **4.2.1** Always leave a space after `//`.
* **4.2.2** Always leave comments on their own line.
* **4.2.3** When using `// MARK: - whatever`, leave a newline after the comment.

```swift
class Pirate {

    // MARK: - instance properties

    private let pirateName: String

    // MARK: - initialization

    init() {
        /* ... */
    }

}
```

