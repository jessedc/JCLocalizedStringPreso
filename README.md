
    /* Melbourne Cocoheads August 2012 presentation title */
    "talk-title" = "Localising your apps.";

    /* presentation author */
    "presentation-by" = "Jesse Collis";

    /* presentation author email address */
    "author-email" = "jesse@jcmultimedia.com.au";

.

## Three stages of localisation

1. Initial development
2. Initial localisation effort
3. Ongoing updates

## If your Localizable.strings file looks like this but you're lazy

    /* No comment provided by engineer. */
    "Next" = "Next";

## The absolute _worst_ localisation offenders

### The _"Oh I heard literal strings in code are bad"_ kind of localisation

    NSLocalizedString(@"Next", @"");
	NSLocalizedString(@"Next", nil);

### The _wtf is the comment for anyway_ kind of localisation

    NSLocalizedString(@"Next", @"Next");

### The _out of sight out of mind_ kind of localisation

	#define MYBadCompanyLocalizedString(key) NSLocalizedString((key), @"")

### The _entire book as a key_ kind of localisation

	NSLocalizedString(@"In order to determine your location, location services must be turned on in settings", @"In order to determine your location, location services must be turned on in settings");

### The _key can be anything but I know what it is_  kind of localisation

    NSString *aStringFromAPI = [response objectForKey:@"next-title"];
    NSLocalizedString(aStringFromAPI, @"api resposne for `next`");

> In this case Apple's tools can't help you; `genstrings` will throw warnings and ignore your `NSLocalizedString()` all together.


## The _not as bad_ offenders

### Duplicating keys

This is a _grey_ area; good comments can make duplicating keys manageable

    NSLocalizedString(@"Next", @"RootViewController 'next' button title");
    NSLocalizedString(@"Next", @"DetailViewController 'next page' button title");

Good comments show you the `key`'s appearances in your app in the .strings file

    /* DetailViewController 'next page' button title
       RootViewController 'next' button title */
    "Next" = "Next";

* `genstrings` will warn you about duplicate keys
* Bad/stupid/empty comments make this a *bad offender*
* keys with `nil` or `@""` comments aren't picked up as duplicate


### Using localised strings in format strings

    button.titleLabel.text = [NSString stringWithFormat:@"4 %@, 3 %@", 
                              NSLocalizedString(@"pineapples",@"plural pineapples"),
                              NSLocalizedString(@"pears",@"plural pears")];

This is _grey_ area because it's inflexible and implies a structure, but can be okay for really small strings, but often you should consider `NSNumberFormatter` for numbers. (That's another talk)

### Localising the format string too

    button.titleLabel.text = [NSLocalizedString stringWithFormat:
                              NSLocalizedString(@"4 %@, 3 %@", @"fruit quantities format string"),
                              NSLocalizedString(@"pineapples",@"plural pineapples"),
                              NSLocalizedString(@"pears",@"plural pears")];

But at this point we've got localised pieces everywhere. But it's good that you're trying.

## How NSLocalizedString() works 


### NSLocalizedString() macros all call to `NSBundle`

    //NSBundle.h
    #define NSLocalizedString(key, comment) \
    	    [[NSBundle mainBundle] localizedStringForKey:(key) value:@"" table:nil]
    #define NSLocalizedStringFromTable(key, tbl, comment) \
    	    [[NSBundle mainBundle] localizedStringForKey:(key) value:@"" table:(tbl)]
    #define NSLocalizedStringFromTableInBundle(key, tbl, bundle, comment) \
    	    [bundle localizedStringForKey:(key) value:@"" table:(tbl)]
    #define NSLocalizedStringWithDefaultValue(key, tbl, bundle, val, comment) \
    	    [bundle localizedStringForKey:(key) value:(val) table:(tbl)]

    - (NSString *)localizedStringForKey:(NSString *) value:(NSString *) table:(NSString *)

`localizedStringForKey:value:table:` has defaults

* default `table` is *Localizable*
* default `value` is `nil` or `@""`
* unlocalized keys return `vale` or the key if `value` is `nil`

### NSBundle's localisation selection

You can localise resources by duplicating them in `*.lproj` directories. You can see the supported localisations as an array of string in your NSBundle instance has with:

    NSArray *localisations = [bundle localizations];

    po [[NSBundle mainBundle] localizations]
    (id) $4 = 0x08356290 <__NSCFArray 0x8356290>(en,ko)
    

These folders must be of a particular format. `en` / english, `ko` / Korean. You can't just give it random .lproj folder names. .lproj directory names` should be canonicalized IETF BCP 47 language identifier strings. [Here's a list](http://www.iana.org/assignments/language-subtag-registry)


NSBundle then references the `@"AppleLanguages"` key from [NSUserDefaults standardUserDefaults] in combination with it's `- localizations` to work out what .lproj directory it will look for `key:` and `table:`.

My educated `guess` is the @"AppleLanguages" key in NSUserDefaults is derived from `[NSLocale preferredLocalizations]` which represents your system's preferred localisations in order of preference.

    po [[NSUserDefaults standardUserDefaults] objectForKey:@"AppleLanguages"];
    (id) $1 = 0x0754e790 <__NSCFArray 0x754e790>(
    en,fr,de,ja,nl,it,es,pt,pt-PT,da,fi,nb,sv,ko,zh-Hans,zh-Hant,ru,pl,tr,uk,ar,hr,cs,el,
    he,ro,sk,th,id,ms,en-GB,ca,hu,vi)

>[There is a number of Stack Overflow posts](http://stackoverflow.com/questions/1669645/how-to-force-nslocalizedstring-to-use-a-specific-language) on how to force NSLocalizedString() to use a language by changing this value in NSUserDefaults early in the app lifecycle, even changing it, syncing it and force quitting the app.

There's also some smarts here about what to look for if nothing is localised, but it's not that important here.

### Within localizedStringForKey:value:table

At this point we're looking for a `stringsTable` based on the set `localisation` which we know already is just an IETF BCP 47 `NSString`.

To find our strings table path we use another NSBundle method:

    NSString *tablePath = [bundle pathForResource:tableName 
                                           ofType:@"strings" 
                                      inDirectory:nil 
                                  forLocalization:localization];

To translate this path from a `.strings` file into something we can use, we use:

    NSString *stringsFile = [NSString stringWithContentsOfFile:stringsFilePath 
                                                      encoding:NSUTF8StringEncoding 
                                                         error:nil];

    NSDictionary *stringsDict = [stringsFile propertyListFromStringsFileFormat];


And finally, where `key` is the original key we passed to NSLocalizedString():

    return [stringsDict objectForKey:key];

### NB:

From NSString.h regarding `propertyListFromStringsFileFormat`

> These methods are no longer recommended since they do not work with property lists and strings files in binary plist format. Please use the APIs in NSPropertyList.h instead.


## My way to use NSLocalizedString()


    self.title = NSLocalizedStringFromTable(@"nav-title",@"StationDetailController",@"NavigationItem title (Station Detail)");

### Use the longer `NSLocalizedStringFromTable()`

Provide a `tableName` to each string.

This splits your `.strings` files by logical sections making localisation easier and implies a context to your keys and comments.

### Use unique keys

Even with the extra context provided by the different table names, I'm an advocate for unique keys across everything.

### Comments

Provide useful comments with the suggested English (en) localisation included

    @"NavigationItem title (Station Detail)"

    @"TableView section 2 ('Other City Metro apps')"

## Localising format strings

Here's how not to do it

    overviewCell.detail = [NSString stringWithFormat:
    NSLocalizedString(@"stops and transfers format",@"Format for saying (1- 15) (3- stops) and (2- 4) (4-Transfers)"),
    [NSNumber numberWithInt:[self.route.routePath count] - 1],
    [NSNumber numberWithInt:[self.route.transfers count] - 1],
    NSLocalizedString(@"stops",@"stops"), 
    [self.route.transfers count] > 1 ? 
      NSLocalizedString(@"Transfers",@"Plural transfers") : 
      NSLocalizedString(@"Transfer",@"Singular Transfers")];
      
Don't do this... localise each case

    if (stops > 1){
      if (transfers > 1){
        detailString = [NSString stringWithFormat:NSLocalizedStringFromTable(@"plural-stops-plural-transfers",@"RouteDetail",@"Overview Header - 2 (plural) stops and 2 (plural) transfers ('%@ stops and %@ transfers')"), stopsNumber, transfersNumber];
      }else if (transfers == 1){
        detailString = [NSString stringWithFormat:NSLocalizedStringFromTable(@"plural-stops-single-transfer",@"RouteDetail",@"Overview Header - 2 (plural) stops and 1 (singular) transfer ('%@ stops and %@ transfer')"), stopsNumber, transfersNumber];
      }else{
        detailString = [NSString stringWithFormat:NSLocalizedStringFromTable(@"plural-stops",@"RouteDetail",@"Overview Header - 2 (plural) stops ('%@ stops')"), stopsNumber];
      }
    }else{
      detailString = JCLocalizedStringFromTable(@"one-stop",@"RouteDetail",@"Overview Header - 1 (singular) stop ('only one stop!')");
    }

## Localising non-literal strings

If you've got strings from API or external source that aren't statically defined, `genstrings` complains, as it can't read your keys. 

**My solution** is to use a specific table and use `genstrings -skipTable` to skip it so it won't error, and will let me create my own .strings file.

    NSLocalizedStringFromTable(apiValue, @"ApiTranslations", @"apiValue for such and such");

## Using `genstrings` properly

  * Use `genstring -s` to define your own macros. Make sure their name doesn't clash with other class names.
  * If you use *stringFromTable: it will create the appropriate .strings file for you
  * It will read `NSLocalizedString*` elements in comments
  * Output your files to any directory, like en.lproj
  * It gives good good warnings
  * suppress multiple key warnings with `-q`
  * It will add numbers to format string parameters `@1$%` etc
  * You can skip certain `tableName`s with `-skipTable` (see localising non-literals)

  .

      genstrings -s JCLocalizedString ./*.m -o ./en.lproj

## Localising xibs and story boards with `ibtool`

generate your strings

    ibtool --generate-strings-file MainStoryboard.strings MainStoryboard.storyboard

write your strings

    ibtool --strings-file ~/MainStoryboard.strings 
           --write ../ko.lproj/MainStoryboard.storyboard MainStoryboard.storyboard

> Xcode 4.4 , Mountain Lion, iOS 6 lets you localise a xib file with a .strings file named the same.

## Show non localised strings with -NSShowNonLocalizedStrings

Pass `-NSShowNonLocalizedStrings YES` as a launch argument in your scheme and see the unlocalised strings come to life in **ALLCAPS**

## Supporting your app going forward

The previous info is great when you build an app from scratch, but updating an app months or years after initial release means your strings change, contexts change and this is hard to manage, especially if you use some of the bad methods above.

### Linguan 

* Manages your strings files, and changes.
* A smarter wrapper around `genstrings` and `libtool`

### AppCode

AppCode has blazed ahead in this space

Their latest [blog post](http://blog.jetbrains.com/objc/2012/07/appcode-1-6-eap120-293-new-features-again-youll-see/) details a lot of new features

* Rename localisation keys
* Provide missing localisations
* Highlight unused keys
* Find usage of a key

## Localisation services

* International friends
* [crowdin.net](crowdin.net)
* [applingua.com](http://http://applingua.com)

## Other Third Party Tools

* Greenwich 

## Extensions to this talk

* Auto Layout OSX Lion, iOS 6
* Internationalisation with NSLocale, NSDateFormatter, NSNumberFormatter

## Other References

WWDC Session 244 'Internationalization tips and tricks' presented by Dave De Long

http://www.albertmata.net/articles/introduction-to-internationalization-using-storyboards-on-ios-5.html
https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/LoadingResources/Strings/Strings.html#//apple_ref/doc/uid/10000051i-CH6-SW1