BIND
====

 [![Pod](http://img.shields.io/badge/pod-1.1.0-lightgrey.svg)](http://cocoapods.org/)
 [![License](http://img.shields.io/:license-mit-blue.svg)](http://doge.mit-license.org)
 [![Build Status](https://travis-ci.org/markohlebar/BIND.svg?branch=master)](https://travis-ci.org/markohlebar/BIND)

Data binding and MVVM for iOS

## History ##

This library emerged from my exploration of [different architectures](https://github.com/markohlebar/iOSArchitectures) for iOS apps. 
It draws some ideas from other similar projects like 
 - [KJSimpleBinding](https://github.com/kristopherjohnson/KJSimpleBinding)
 - [KeyPathBindings](https://github.com/dewind/KeyPathBindings)
 - [objc-simple-bindings](https://github.com/mruegenberg/objc-simple-bindings)

with a couple of major feature advantages.

## Features ##

- **data binding** from **XIBs** or **code** by using a custom **BIND DSL**
- **data transforms** by using subclasses of `NSValueTransformer`
- **protocols** and **abstract classes** to make your **MVVM**-ing easier
- **automagic unbinding** no more **KVO exceptions** on dealloc

## Why Use This? ##

Because your `View` implementation code will literally start looking like this:
```
@implementation MyTableViewCell
@end
```
That's one of the reasons anyways... 

## Integration ##
#### CocoaPods ####
- pod 'BIND'
- `#import "BIND.h"`

## BIND DSL ##

Bind offers a special language to build your bindings. You can use it either from code or from XIB-s. The language describes which keypaths are assigned for each binding. 

### Examples ###

Let's say you are building a table view based app and you want to show names of different persons in the cells. 
Assume that the `PersonViewModel` **view model** has a property `name` which you want to display on you **cell's** `textLabel`. 

#### Binding From XIB ####

**BIND** lets you create your bindings from XIBs. The easiest way to do this is to use 
the `BNDTableViewCell` class or create it's subclass. `BNDTableViewCell` exposes an interface
for assigning the `viewModel`, which should happen on each `tableView:cellForRowAtIndexPath:` call, 
and `bindings` which is a xib collection outlet of bindings which will be updated with each subsequent `viewModel` assignment.

![](https://raw.githubusercontent.com/markohlebar/BIND/master/misc/bind_from_xib.gif)

In the gif above you can observe a simple procedure of adding a binding to a cell which is a subclass of `BNDTableViewCell`. The steps are as follows: 
- create an empty XIB and name it the same as your `BNDTableViewCell` subclass
- from Objects Library drag in a `Table View Cell` and change it's class to subclass you created.
- next, from Objects Library drag in an `Object` and change it's class to `BNDBinding`
- add a keypath, change it's "Type" to `String`, "Key Path" to `BIND`, and type a BIND expression as the "Value" (In the example above I'm connecting my viewmodel's `name` keypath to `textLabel.text` of the cell)
- right click on your table view cell and find the Outlet Collection called `bindings`
- connect the previously created Binding with the outlet collection. 
- in your table view delegate's 'tableView:cellForRowAtIndexPath:' you should set the `viewModel` property of the cell with your view model

#### Binding From Code ####

Similar to the binding from XIB example, what you need to do is bind the cell's `textLabel.text` key path with the `name` key path of your **view model**. 

```
@interface PersonTableViewCell : BNDTableViewCell
@end

@implementation PersonTableViewCell 

- (instancetype)init {
    ...
    //this will bind viewModel.name to cell's textLabel.text property
    BNDBinding *binding = [BNDBinding bindingWithBIND:@"name -> textLabel.text"];
    
    //assign the binding to the array of bindings property (BNDTableViewCell).
    //any calls to setViewModel: will automatically refresh the binding by calling 
    //[binding bindLeft:self.viewModel withRight:self];
    //making sure that your objects are bound on cell reuse.
    self.bindings = @[binding]; 
    ...
}

- (void)viewDidUpdateViewModel:(id <BNDViewModel> )viewModel {
    //this method is called after each call to setViewModel: on the cell
    //use this instead of overriding setViewModel: 
} 
    
@end
``` 

#### Binding Direction and Initial Value Assignment####
Observe the symbol `->` in the expression `name -> textLabel.text`. 
**BIND** syntax lets you configure the way that the binding is reflected on the bound objects values. 
It offers six different direction and intial value assignment configurations:
```
  name -> textLabel.text /// left object passes values to right object
  name <- textLabel.text /// right object passes values to left object.
  name <> textLabel.text /// binding is bidirectional. Initial value is passed from left to right object.
  name !-> textLabel.text /// left object passes values to right object with no initial value assignment.
  name <-! textLabel.text /// right object passes values to left object with no initial value assignment.
  name <!> textLabel.text /// binding is bidirectional with no initial value assignment. 
```

#### Transformers ####
**BIND** also lets you assign your own subclasses of `NSValueTransformer` to transform values coming from object
to other object and reverse. Let's take the previous example, and assume that there is a requirement that the names should be displayed capitalized in the cells. You could then build your subclass of `NSValueTransformer` and easily assign it to the binding. When assigning a value transformer, you can either use it's class name or transformer name. If you pass in a class name that hasn't yet been registered, **BIND** will register that `NSValueTransformer`, and use the class name as the registered transformer name on any subsequent calls. 

```
@interface CapitalizeStringTransformer : NSValueTransformer
@end

@implementation CapitalizeStringTransformer 

///transformValue: is called when assigning from object to otherObject
- (NSString *)transformValue:(NSString *)string {
    return string.capitalizedString; 
}

///reverseTransformValue: is called when assigning from otherObject to object
- (NSString *)reverseTransformValue:(NSString *)string {
    return string;
}

@end 

...
- (instancetype)init {
    ...
    _binding = [BNDBinding bindingWithBIND:@"name -> textLabel.text | CapitalizeStringTransformer"];
    ...
}
...

```
Observe `| CapitalizeStringTransformer` syntax which tells the binding to use the `CapitalizeStringTransformer` subclass of `NSValueTransformer` to transform the values. 
You can reverse the transformation direction if you need to by adding a `!` modifier before transformer name like so `name -> textLabel.text | !CapitalizeStringTransformer`.

## KVO with BIND ##

KVO is ugly and it will crash your app if you forget to remove your observer. 
**BIND** is not just limited to viewModels and cells, you can use it for same purposes as you would use KVO. 
Let's say you have a car and an engine within that car. 
When the engine increases it's rpm's then you want your car's speed to change as well. 
Also let the transformation between rpm's -> speed be as trivial as 100:1. 

```
    car.speed = 100;
    engine.rpm = 10000;
    BNDBinding *binding = [BNDBinding bindingWithBIND:@"rpm -> speed | RPMToSpeedTransformer"];
    [binding bindLeft:engine withRight:car];
    engine.rpm = 20000;
    //car.speed is 200 at this point. 
```

## Automagic Unbinding ##

As of version 1.1.0, **BIND** is automatically handling unbinding of bound objects. This means no more KVO exceptions like the following:
`"NSInternalInconsistencyException", "An instance 0xF00B400 of class XYZ was deallocated while key value observers were still registered with it.`

## Sample Project ##

Check [iOSArchitectures project](https://github.com/markohlebar/iOSArchitectures).

## TBC ##
- more abstract classes like viewcontrollers, views etc. 
