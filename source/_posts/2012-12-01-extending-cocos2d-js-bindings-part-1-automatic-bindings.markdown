---
layout: post
title: "Extending cocos2d JS bindings. Part 1: automatic bindings"
date: 2012-12-01 11:07
comments: true
categories: objective-c ios cocos2d javascript
---

Since version 2.1 [cocos2d-iphone](http://www.cocos2d-iphone.org) comes with ability to write code in JavaScript.
This allows you to write your code once and then reuse the code with cocos2d-x or cococs2d-html5.
All cocos2d nodes, CocosDension, CocosBuilder reader and Chipmunk are supported. But what about third parties?
Do we need to rewrite other great libraries in JS to use them? No. And here [jsbindings](https://github.com/zynga/jsbindings) project comes for the rescue. 

While official documentation is pretty good to get started, there are few pitfalls when using this tool
in real project. So, lets explore the process of binding generation step by step and allow JS coders to use 
[CCScrollView](https://github.com/vlidholt/CCBReader/tree/master/CCScrollView). At this tutorial we'll explore
automatic binding process and bind most of the CCScrollView methods to JS, next time we'll focus on manual
bindings and complete the rest.
<!-- more -->

## Preparations

We will be using [cocos2d-iphone 2.1-beta3](http://www.cocos2d-iphone.org/archives/2084).
Download it and install Xcode templates if you haven't yet. Create new "cocos2d iOS with JavaScript" project.

Next download [CCBReader project](https://github.com/vlidholt/CCBReader) and copy CCScrollView directory to `libs` 
folder in your project (same directory that contains cocos2d, CocosDension and other 3rd party libraries).

Finally, download [JSBindings 0.3](https://github.com/zynga/jsbindings/archive/release-0.3.zip) and place it
somewhere on your computer. In following examples I'll assume that its installed in home directory. If you have
installed it somewhere else, don't forget to replace `~/jsbingings` with your path when following further
instructions.

## Step 1: gen_bridge_metadata

As documentation suggests, step 1 is generating bridge support files for your project. Open terminal in 
`YOUR_PROJECT/libs/CCScrollView` directory and execute following command:

```
gen_bridge_metadata -F complete --no-64-bit -c '-DNDEBUG -I.' *.h -o CCScrollView.bridgesupport
```

BridgeSupport is an xml file that describes C functions, ObjC classes, methods and their parameters, etc. 
`gen_bridge_metadata` script uses [libclang](http://clang.llvm.org/doxygen/group__CINDEX.html) to parse Objective-C code and generate BridgeSupport files.

If you look carefully through generated file you'll see the first problems: all cocos2d classes like `CCNode`
replaced with `id`. To fix this you'll need to tell `gen_bridge_metadata` script to include cocos2d headers 
in search path. The `-c` options in above command allows to pass additional compiler arguments to libclang and 
`-I` compiler flag adds directory to compiler path. So, fixed version of the command will be:

```
gen_bridge_metadata -F complete --no-64-bit -c '-DNDEBUG -I. -I../cocos2d' *.h -o CCScrollView.bridgesupport
```

Now BridgeSupport file should contain right types of parameters and properties.

## Step 2: complement file

Second step is generating a complement file. Complement is a file with additional metadata not provided by 
BridgeSupport such as class hierarchy, protocols and properties. The command is:

```
~/jsbindings/generate_complement.py -o CCScrollView-complement.txt *.h
```

Script will tell you that it completed successfully, but don't trust him: if you open the file it generated you'll 
see that it contains no data.

The problem is that CCScrollView files for some reason have Mac OS 9 line endings (CR) and `generate_complement` 
scripts expects Unix (LF). We can fix either the script, or convert file line endings. I choose the second path.
Execute following commands from CCScrollView directory:

```
vim -c ':e ++ff=mac | :setlocal ff=unix | :wq' CCScrollView.h
vim -c ':e ++ff=mac | :setlocal ff=unix | :wq' CCScrollView.m
```

Let's run a `generate_complement` command again:

```
Inf:CCScrollView$ ~/jsbindings/generate_complement.py -o CCScrollView-complement.txt *.h
Traceback (most recent call last):
  File "/Users/Inf/jsbindings/generate_complement.py", line 183, in <module>
    instance.parse()
  File "/Users/Inf/jsbindings/generate_complement.py", line 97, in parse
    raise Exception("Fatal: Unparented attrib: %s (%s)" % (str(property_attribs.groups()), filename))
Exception: Fatal: Unparented attrib: ('nonatomic, assign', 'CGFloat', None, None, None, 'zoomScale') (CCScrollView.h)
```

Now the problem is the following: to parse code `generate_complement` uses set of regular expressions and they are
very specific about code formatting. Again, the solution would be either to patch utility (the ideal variant would
be using libclang instead of regexps) or fix the code formatting. Again, I choose the second variant.

Open CCScrollView.h header and find `CCScrollViewDelegate` protocol declaration. Change it from this:

``` objective-c
@protocol CCScrollViewDelegate
<
          NSObject
>
```

to this:

``` objective-c
@protocol CCScrollViewDelegate<NSObject>
```

Then, change `CCScrollView` class declaration from this:

``` objective-c
@interface CCScrollView 
:          CCLayer {
```

to this:

``` objective-c
@interface CCScrollView: CCLayer {
```

Run the same command third time and you'll finally get a complement file.

## Step 3: Config file

Finally, lets write a config file and generate some bindings. Create `CCScrollView.jsb.ini` file in 
`libs/CCScrollView` directory with a following contents:

```
[CCScrollView]

#1
obj_class_prefix_to_remove = CC 

classes_to_parse = CCScrollView

#2
class_properties = CCLayer = manual,
                   CCNode = manual,
                   NSObject = manual

#3
inherit_class_methods = Auto

#4
import_files = CCScrollView.h, js_bindings_cocos2d_ios_classes.h

method_properties = 
#5
    CCScrollView # viewWithViewSize:container: = name: "create"; merge: "viewWithViewSize:",
    CCScrollView # initWithViewSize:container: = name: "initWithViewSize"; merge: "initWithViewSize:",
    CCScrollView # setContentOffset:animated: = name: "setContentOffset"; merge: "setContentOffset:",
    CCScrollView # setZoomScale:animated: = name: "setZoomScale"; merge: "setZoomScale:",
#6
    CCScrollView # setContentOffset:animatedInDuration: = name: "setContentOffsetInDuration",
    CCScrollView # setZoomScale:animatedInDuration: =  name: "setZoomScaleInDuration",
#7
    CCScrollView # delegate = ignore,
    CCScrollView # setDelegate: = ignore

#8
struct_properties = CGPoint = manual,
                    CGRect = manual,
                    CGSize = manual

#9
bridge_support_file = CCScrollView.bridgesupport
complement_file = CCScrollView-complement.txt
```

Explanations:

1. We tell generator to remove CC prefix from class names. Later, we will register binded class under `cc` 
namespace, so `CCScrollView` will be accessible as `cc.ScrollView` in JS code;
2. We tell that we already have manual bindings for `CCLayer`, `CCNode` and `NSObject`. Cocos2d bindings already 
covers this classes, but without this line generator will try to generate them again. `manual` option prevents
generator from this behaviour while still allowing remaining code to use this classes;
3. Generated bindings will inherit class methods from the base class until virst class constructor encoutered;
4. All binding files will include two headers:
    1. `CCScrollView.h` to access our native class;
    2. `js_bindings_cocos2d_ios_classes.h` to access bindings for `CCLayer`, `CCNode` and `NSObject`.
5. Two important things happens here:
    1. Diffrent name for the methods in JS is set using `name` option. This allows to use JS-friendly names in
    our script code and brings comatability with 
    [cocos2d-html5 implementation](https://github.com/cocos2d/cocos2d-html5/tree/master/extensions/GUI/CCScrollView);
    2. Multiple native methods are merged into single JS method using `merge` option. This makes sense for similar
    methods with different number of arguments. Glue code will choose appropriate native implementation based on 
    the number of arguments passed to JS function. **Note**: method with largest number of argments should be on a 
    left side of expression.
6. Just rename a few methods to be compatable with HTML5 version;
7. Don't generate bindings for delegate getter and setter. Currently, jsbindings can't generate glue code for
protocols. In the following article we change this setting to `manual` and write binding ourself, but for now we 
will just ignore the methods.
8. Tell generator that we'll have manual bindings for `CGPoint`, `CGRect` and `CGSize`. Similar to the manual
classes above, cocos2d will provide this bindings. Without this bindings properties of this types could not be set
or read from JavaScript.
9. Setting the path to the BridgeSupport and complement files generated on previous steps.

Now lets run binding generator:

```
~/jsbindings/generate_js_bindings.py -c CCScrollView.jsb.ini                                     
NOT OK: "CCScrollView#delegate" Error: Explicitly ignoring method
NOT OK: "CCScrollView#initWithViewSize:" Error: Explicitly ignoring method
NOT OK: "CCScrollView#setContentOffset:" Error: Explicitly ignoring method
NOT OK: "CCScrollView#setDelegate:" Error: Explicitly ignoring method
NOT OK: "CCScrollView#setZoomScale:" Error: Explicitly ignoring method
NOT OK: "CCScrollView#viewWithViewSize:" Error: Explicitly ignoring method
```

For some reason it reports ignored methods as errors. Looks like a bug in `generate_js_bindings` script. Just
ignore this errors for now, correct binding files will be generated anyway.

## Step 4: Registration file

Now we need to create a function that will register our bindings with JS engine. This part is done
manually.

Create `js_bindings_CCScrollView_registration.h` with the following content:

``` objective-c

#ifndef __JSB_CCSCROLLVIEW_REGISTRATION
#define __JSB_CCSCROLLVIEW_REGISTRATION

void jsb_register_CCScrollView( JSContext *_cx, JSObject *globalO);


#endif

```

Create Create `js_bindings_CCScrollView_registration.mm` with the following content:

``` objective-c
#import "js_bindings_config.h"
#import "js_bindings_core.h"
#import "js_bindings_CCScrollView_classes.h"
#import "js_bindings_CCScrollView_registration.h"

void jsb_register_CCScrollView( JSContext *_cx, JSObject *globalO) { //1
    jsval ns;
    JS_GetProperty(_cx, globalO, "cc", &ns); //2
    JSObject* CCScrollView = JSVAL_TO_OBJECT(ns); //3
    
#import "js_bindings_CCScrollView_classes_registration.h" //4
}
```

Some knowledge of [Spider Monkey API](https://developer.mozilla.org/en-US/docs/SpiderMonkey) required to 
understand this code. We dive more deeply into JSAPI in the next article when we'll be discussing manual bindings,
now I give only breif explanation:

1. Registration function receives two parameters:
    1. JSContext - central part of JSAPI. It maintains call stack, contains global object and required by
    almost every JSAPI function.
    2. Global object - object, that contains all other objects, avaliable to the scripts. For example, `window`
    inside web browser is a global object for it.
2. Getting `cc` property from global object. This property is a namespace for cocos2d, it contains all other 
functions, classes and constants of the engine. We want to add ScrollView to the same namespace, so we need to get
a reference to it. This code assumes that it will be called after cocos2d has been registered.
3. Now, we have a namespace property, but it can have value of any type in JS: number, string, object, etc.
Autegenerated registration functions require namespace to be an object named `CCScrollView` (generally, 
namespace variable should be the same as section header in config file). So, we convert the value to satisfy
requirements.
4. Importing files with autegenrated functions that register all the classes.  

## Step 5: Adding files to a project

Open Xcode project and add following files to it: 
* `js_bindings_CCScrollView_classes.h`;
* `js_bindings_CCScrollView_classes.mm`;
* `js_bindings_CCScrollView_classes_registration.h`;
* `js_bindings_CCScrollView_registration.mm`;
* `js_bindings_CCScrollView_registration.mm`.

Open `libs/jsbindings/src/manual/js_bindings_config.h` and add following lines to it:

``` objective-c
#ifndef JSB_INCLUDE_CCSCROLLVIEW
#define JSB_INCLUDE_CCSCROLLVIEW 1
#endif
```

This will enable compilation of our bindings.

Open `libs/jsbindings/src/manual/js_bindings_core.mm` and add followig import to it:

``` objective-c
#import "js_bindings_CCScrollView_registration.h"
```

Then find `createRuntime` method and add following lines to it
**after** cocos2d registration:

```
#if JSB_INCLUDE_CCSCROLLVIEW
    jsb_register_CCScrollView(_cx, _object);
#endif
```

Step 6: Constants file and test:

Add jsb_constants_ccscrollview.js file to resources of your application:

``` javascript
cc.SCROLLVIEW_DIRECTION_HORIZONTAL = 0;
cc.SCROLLVIEW_DIRECTION_VERTICAL = 1;
cc.SCROLLVIEW_DIRECTION_BOTH = 2;
```

Its just the same constants that defined in `CCScrollViewDirection` enum. jsbindings doesn't support enums,
so all constants should be redefined in JS.

Now its time to test the results. Replace `Resources/main.js` file content with the following code:

``` javascript 
require("jsb_constants.js");
require("jsb_constants_ccscrollview.js");

var MainLayer = cc.Layer.extend({
    ctor: function() {
        cc.associateWithNative(this, cc.Layer);
        this.init();
        var winSize = cc.Director.getInstance().getWinSize();
        var container = cc.LayerGradient.create(cc.c4b(255, 255, 255, 255, 255),
                                                cc.c4b(0, 0, 0, 255));
        container.setContentSize(cc.size(winSize.width, 1000));
        var scrollView = cc.ScrollView.create(winSize, container);
        scrollView.setDirection(cc.SCROLLVIEW_DIRECTION_VERTICAL);
        this.addChild(scrollView);
    }
});


function run()
{
    var scene = cc.Scene.create();
    var layer = new MainLayer();
    scene.addChild( layer );

    cc.Director.getInstance().runWithScene( scene );
}

run();
```

If everything was done right, you should see nice scrolling and bouncing gradient.

## The end

First part of tutorial is over, code can be found on [GitHub](https://github.com/SevInf/CCScrollView), but it
uses different folder structure. Next time we'll dive into manual binding process and allow CCScrollView to have
JavaScript delegate.
