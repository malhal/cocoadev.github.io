---
layout: page
title: SmartFolders
---

How does one go about creating a smart folder on the filesystem programmatically from a Cocoa app?  It doesn't appear that NSFileManager got any new methods in Tiger that will support this functionality.

----

A smart folder is actually just an XML file, which you can see by opening it with a text editor. The format looks to be fairly obvious; I would assume if you generate a properly-structured file with the right extension then it will work as expected.

*Just out of curiosity (I'm still using Panther), can you post the XML from an example file/folder? --JediKnil*

----

Perhaps you should look in the Spotlight API, instead of cocoa?

----

Here's a saved search that searches for items with the name "foo" in the home folder:
    
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>CompatibleVersion</key>
	<integer>0</integer>
	<key>RawQuery</key>
	<string>(* = "foo*"wcd || kMDItemTextContent = "foo*"cd) &amp;&amp; (kMDItemContentType != com.apple.mail.emlx) &amp;&amp; (kMDItemContentType != public.vcard)</string>
	<key>SearchCriteria</key>
	<dict>
		<key>AnyAttributeContains</key>
		<string>foo</string>
		<key>CurrentFolderPath</key>
		<array>
			<string>/Users/mikeash</string>
		</array>
		<key>FXCriteriaSlices</key>
		<array>
			<dict>
				<key>FXSliceKind</key>
				<string>Skin</string>
				<key>Value</key>
				<string>KI**</string>
			</dict>
			<dict>
				<key>FXSliceKind</key>
				<string>Slsv</string>
				<key>Value</key>
				<string>DA**</string>
			</dict>
		</array>
		<key>FXScope</key>
		<integer>1396926573</integer>
		<key>FXScopeArrayOfPaths</key>
		<array>
			<string>kMDQueryScopeHome</string>
		</array>
	</dict>
	<key>Version</key>
	<string>10.4</string>
	<key>ViewOptions</key>
	<dict>
		<key>SidebarWidth</key>
		<integer>183</integer>
		<key>ToolbarVisible</key>
		<true/>
		<key>ViewStyle</key>
		<string>qrsb</string>
		<key>WindowBounds</key>
		<dict>
			<key>bottom</key>
			<integer>0</integer>
			<key>left</key>
			<integer>0</integer>
			<key>right</key>
			<integer>0</integer>
			<key>top</key>
			<integer>0</integer>
		</dict>
	</dict>
</dict>
</plist>


----

From some tinkering, it seems that the Finder executes the RawQuery in the folders inside SearchCriteria/FXScopeArrayOfPaths. All other values are either aestetic or used by the query generation UI (in particular FXCriteriaSlices/* seem to be ignored when executing the query, but are used to fill in the UI and regenerate the query when the user manipulates the search criteria).

----

The default constraint within the predicate query string generated by Finder is curious... It excludes the Mail and vcard UTI types.

i.e.,:
     
&amp;&amp; (kMDItemContentType != com.apple.mail.emlx) &amp;&amp; (kMDItemContentType != public.vcard)


To create more interesting Smart Folders (think BeFS style...), try this for a folder of persons in your Address Book:

 
* Do a simple search in Finder and then save it, preferably the Desktop to make it easier to work with while changing it.
* Open Property List Editor.
* The *Smart Folder* file format is a plist but Open File dialogs recognize them as folders. Simple drag the folder to the Dock icon for the Property List Editor. Viola... structured editing.
* Replace the RawQuery value with the following: *(kMDItemContentType == com.apple.addressbook.person)*
* Save the file, making sure the extension is .savedSearch. The extension can be hidden again in the Finders Get Info panel.
* Open the folder and you've got a smart folder of People. Add constraints such as: *(kMDItemCity = **Seattle**cd)* to only see items that match certain attribute constraints.
* Crack open the  programming guides for metadata attributes, predicates and UTIs to imagine the possibilities.
 

The differences between Finder Smart Folders and Spotlight query result windows seem arbitrary and are likely artifical. For instance, Finder is not the owning process of the Spotlight query result window lives, SystemUIServer is. The question  Apple UE must be considering for future releases is whether the technology basis behind Smart Folders, the metadata store, indexing system and attributes and APIs, will be grokked by enough of the user base to move *Smartness* outside of applications and into something that is not the Finder or a Spotlight query result window. I see Smartness as enabling the owner and user of any computational resource to be able to choose the manner and method best suited to the nature of the data, be it based on the the goal at hand or on the users own conceptual view of their environment. Once one begins to think about checking their mail as not the act of opening Mail.app and looking in the Inbox but instead clicking a single Smart Folder in Finder to present new, interesting and unread mail... one can begin to see the future taking shape.

**.dpm**

----

It's easy enough to use the results from SmartFolders in your code. The trick is to realise that they are created by the Finder 
which is written in Carbon rather than Cocoa, and thus the query string format is for a MDQueryCreate rather than a NSPredicate. I assume that eventually someone might write some code to allow the transforming of the query strings...

It also seems the Cocoa implementation for metadata queries is incomplete. I mean that the NSMetadataQuery NSMetadataQueryDidUpdateNotification seem to be missing the array of changed items - could be a bug.

If you're not afraid of a little Carbon (is this allowed on this site?) then you can use the following to build a smart folder query:
     
NSString * filePath = @"/Users/me/xxyz.savedSearch"; //i.e. a SmartFolder
NSDictionary *doc = [NSDictionary dictionaryWithContentsOfFile:filePath];
NSString * raw = [doc objectForKey:@"RawQuery"];
// Ugh, Carbon from now on...
MDQueryRef query = MDQueryCreate(kCFAllocatorDefault, (CFStringRef)rawQuery, NULL, NULL);
CFNotificationCenterRef nc = CFNotificationCenterGetLocalCenter();
CFNotificationCenterAddObserver(nc, (void*)self, &MyCallBack, NULL/*name*/, query, CFNotificationSuspensionBehaviorDeliverImmediately);
MDQueryExecute(query, kMDQueryWantsUpdates);

// elsewhere add the callback
static void MyCallBack(CFNotificationCenterRef center, void *observer, CFStringRef name, const void *object, CFDictionaryRef userInfo) {
	if(name==kMDQueryDidFinishNotification) {
              // Finished gathering the query results
	} else if(name==kMDQueryDidUpdateNotification) {
              // In userInfo look for arrays of MDItem keyed on:
              //  kMDQueryUpdateAddedItems
              //  kMDQueryUpdateRemovedItems
              //  kMDQueryUpdateRemovedItems
...

- RbrtPntn
