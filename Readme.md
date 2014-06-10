#Offline support in the Azure Mobile Services iOS SDK

We’ve just released a preview version of the Azure Mobile Services SDK for iOS with offline support. Like the offline support in the .NET SDK, we're releasing a preview version of the offline so that we can get feedback to (from you!) to make sure that we release this feature in a way that will suit your applications.

To present the new feature in the SDK, we'll walk through the steps to make our sample Todo app offline-enabled. We'll be talking about the features as they are needed for the app.

## Initial setup

To start off the walkthough, let's use the quickstart application that you can download in the portal. Create a new mobile service (for this example, let's use the node.js backend, which can be set up quicker than the .NET one.

[[Image: 001-CreateNewMobileService.PNG]]

Once the service is created, download the iOS quickstart to your Mac.

[[Image: 002-DownloadQuickstart.PNG]]

Open it in Xcode and we're ready to add offline support to it. We're now ready to start playing with it.

## Updating the quickstart app

The offline support we're previewing here has some nice features. One of them is the ability to resolve conflicts which can happen when pushing the local changes to the table in Azure. For example, if you're running the same app in two phones, and changed *the same* item in both phones locally, when you're ready to push the changes back to the server one of them will fail with a conflict. The SDK allows you to deal with those conflicts via code, and decide what to do with the item which has a conflict.

The current quickstart, however, doesn't really have many occasions where conflicts can arise, since the only action we can take for an item is to mark it as complete. True, we can mark the item as complete in one client and then do the same in another client, but although technically this is a conflict (and the framework will flag it as such) it's not too interesting. Let's then change the quickstart to make it more interesting by allowing full editing of the todo items.

I'll show here the changes for the iPhone storyboard, but the same would apply for the iPad as well. 
Since we'll show how to handle conflicts, let's make the quickstart more interesting, by letting you edit items instead of just completing them.

## Updating the framework

Download alpha version of our framework from some TBD link

## Setting up CoreData

The boilerplate code

## Defining the model

Instructions on building the model. Copy the existing model (from the framework zip file), and update it to add the TodoItem entity.

## From table to sync table

Just like in the managed case, we make a simple change in the code - instead of using MSTable, we start using MSSyncTable (from `[client tableWithName:]` to `[client syncTableWithName:]`).

## Pulling and pushing

At this point we have a "pure offline" solution, so we need to start exchanging data between the local and the remote tables. That is accomplished via pull and push operations, and this is how we do it...

## Threading considerations

Here's where we talk about dispatching results to the main (UI) thread, given that we used a background managed object queue. 

## Conflict handling

Edits in two different places, how can we handle that in the client?

## Wrapping up

We're now adding offline support for native iOS applications, and like in the managed SDK, we're releasing it in a preview format. We really appreciate your feedback so we can continue improving in the SDKs for Azure Mobile Services. As usual, please leave comments / suggestions / questions in this post, or in or [MSDN Forum](http://social.msdn.microsoft.com/Forums/windowsazure/en-US/home?forum=azuremobile).