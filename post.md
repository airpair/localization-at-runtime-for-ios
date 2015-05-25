## Intro

There is an interesting thing with localisation: it does not work at runtime. Except if you hack it as I will show below.

I will assume you know what localisation is and how to set it up. If you don't, [this](http://www.raywenderlich.com/64401/internationalization-tutorial-for-ios-2014 "Internationalization Tutorial") is a good place to start. I will create a basic view with a UILabel that needs localisation at runtime and a switch between 3 languages. We have already created our Localizable.strings files as  per the tutorial where we are storing a single string:

```swift,linenums=true
"Once upon a time" = "Once upon a time";
```

We want to call the standard method:

```swift,linenums=true
label.text = NSLocalizedString("Once upon a time", comment: "")
```

but internally we want to change its implement so that it will read from the correct strings file. What this method does is to call NSBundle's localizedStringForKey:value:table: so this is what we must replace. To do that we will use Method Swizzling which is described in an excellent manner by [NSHipster](http://nshipster.com/method-swizzling/ "Method Swizzling").  This is essentially telling the runtime to replace one method of a class with a custom one so that when the original method is called, the custom gets triggered underneath. It is a feature of Objective-C that has found its way into Swift (I guess due to popular demand). We will do this in the AppDelegate for lack of a better place*.

```swift,linenums=true
func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject: AnyObject]?) -> Bool {
    
    method_exchangeImplementations(
        class_getInstanceMethod(NSBundle.self, "localizedStringForKey:value:table:"),
        class_getInstanceMethod(NSBundle.self, "kd_localizedStringForKey:value:table:"))
    
    
    return true
}
```

We will encapsulate the implementation of that new method inside an extension of NSBundle.

*Note: The difference between extensions and Objective-C categories is that the former do not include overriding of methods. If this was the case we could create an extension that would override the class method load, where we could do the swizzling. But sadly, this is not the case.

Our extension looks like this:

```swift,linenums=true
extension NSBundle {
    
    func kd_localizedStringForKey(key: String, value: String?, table tableName: String?) -> String {
        
        // default
        var valueToReturn : String = (value != nil && value != "") ? value! : key
        
        if let languages = NSUserDefaults.standardUserDefaults().objectForKey(AppleLanguagesKey) as? [String] {
            
            let currentLanguage = languages[0]
            
            if let path = NSBundle.mainBundle().pathForResource(currentLanguage, ofType: "lproj") {
                
                if let currentLanguageBundle = NSBundle(path: path) {
                    
                    if let localizedStringsFilePath = currentLanguageBundle.pathForResource("Localizable", ofType: "strings") {
                        
                        let localizedStringDictionary = NSDictionary(contentsOfFile: localizedStringsFilePath)
                        
                        if let value = localizedStringDictionary?[key] as? String {
                            valueToReturn = value
                        }
                    }   
                } 
            }
        }        
        return valueToReturn
    }
}
```

We use the same convention as the original system method in looking for a file called Localizable.strings which we have hardcoded here. The interesting thing is the where?!

When adding localization, XCode is creating folders which have the lproj extension and the ISO 639-1 language code, such as en.lproj. So we take the current language and append lproj to it and look for the Localizable.strings in there.

What we also need to do is being able to set the default language and for that I extended the NSUserDefaults class.

```swift,linenums=true
extension NSUserDefaults {
    
    class func setLanguage(languageName : String) -> Void {
        
        if let value = standardUserDefaults().objectForKey(AppleLanguagesKey) as? [String] {
            
            var languages = value
            
            if contains(languages, languageName) == true {
                
                let index = find(languages, languageName)!
                let languageSelected = languages[index]
                languages.removeAtIndex(index)
                languages.insert(languageSelected, atIndex: 0)
            }
            else {
                languages.insert(languageName, atIndex: 0)
            }
            
            standardUserDefaults().setObject(languages, forKey: AppleLanguagesKey)
            
            standardUserDefaults().synchronize()
        }
    }
}
```

All we need to do now is to listen to the callback of the UISegmentedControl and respond accordingly

```swift,linenums=true
@IBAction func languageSwitched(sender: UISegmentedControl) {
    
    let languageCodeToSet = languageCodesArray[sender.selectedSegmentIndex]
    NSUserDefaults.setLanguage(languageCodeToSet);
    
    self.refreshView()
}
```