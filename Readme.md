#Offline support in the Azure Mobile Services iOS SDK

We’ve just released a preview version of the Azure Mobile Services SDK for iOS with offline support. Like the offline support in the .NET SDK, we're releasing a preview version of the offline so that we can get feedback to (from you!) to make sure that we release this feature in a way that will suit your applications.

To present the new feature in the SDK, we'll walk through the steps to make our sample Todo app offline-enabled. We'll be talking about the features as they are needed for the app.

## Initial setup

To start off the walkthough, let's use the quickstart application that you can download in the portal. Create a new mobile service (for this example, let's use the node.js backend, which can be set up quicker than the .NET one.

![Create new mobile service](images/001-CreateNewMobileService.png)

Once the service is created, download the iOS quickstart to your Mac.

![Download quickstart](images/002-DownloadQuickstart.png)

Open it in Xcode and we're ready to add offline support to it. We're now ready to start playing with it.

## Updating the quickstart app

The offline support we're previewing here has some nice features. One of them is the ability to resolve conflicts which can happen when pushing the local changes to the table in Azure. For example, if you're running the same app in two phones, and changed *the same* item in both phones locally, when you're ready to push the changes back to the server one of them will fail with a conflict. The SDK allows you to deal with those conflicts via code, and decide what to do with the item which has a conflict.

The current quickstart, however, doesn't really have many occasions where conflicts can arise, since the only action we can take for an item is to mark it as complete. True, we can mark the item as complete in one client and then do the same in another client, but although technically this is a conflict (and the framework will flag it as such) it's not too interesting. Let's then change the quickstart to make it more interesting by allowing full editing of the todo items.

*Note: if you don't want to walk through the modification of the todo item app, feel free to jump directly to the next section ("Updating the framework")*

### Preparing the navigation view

I'll show here the changes for the iPhone storyboard, but the same would apply for the iPad as well. Open the quick start in Xcode and select the MainStoryboard_iPhone.storyboard file. Once in there, select the main view controller, and on the Editor menu, select "Embed In", "Navigation Controller".

![Embedding main view controller in a navigation controller](images/003-MainControllerIntoNavigationController.png)

Next, select the table view cell in the Todo List View Controller and in the properties window set the Accessory mode to be "Disclosure Indicator"

![Add disclosure indicator to table cell](images/004-TableCellAccessoryDisclosureIndicator.png)

### Adding the details view controller

There are some cosmetic changes which I'll make but I'll skip them here for brevity of this post. Next, add a new view controller class which will back the details view controller which we'll add to the storyboard. Add a new Objective-C class called QSTodoItemViewController, derived from UIViewController to your project, and add a property of type NSMutableDictionary which will hold the item to be modified.

    @interface QSTodoItemViewController : UIViewController <UITextFieldDelegate>

    @property (nonatomic, weak) NSMutableDictionary *item;

    @end

In the implementation (.m) file, add two private properties corresponding to the fields of the todo item which we'll edit:

    @interface QSTodoItemViewController ()

    @property (nonatomic, strong) IBOutlet UITextField *itemText;
    @property (nonatomic, strong) IBOutlet UISegmentedControl *itemComplete;

    @end

    @implementation QSTodoItemViewController

    - (id)initWithNibName:(NSString *)nibNameOrNil bundle:(NSBundle *)nibBundleOrNil
    {
        self = [super initWithNibName:nibNameOrNil bundle:nibBundleOrNil];
        if (self) {
            // Custom initialization
        }
        return self;
    }

    - (void)viewDidLoad
    {
        [super viewDidLoad];

        UINavigationItem *nav = [self navigationItem];
        [nav setTitle:@"Todo Item"];

        NSDictionary *theItem = [self item];
        [self.itemText setText:[theItem objectForKey:@"text"]];

        BOOL isComplete = [[theItem objectForKey:@"complete"] boolValue];
        [self.itemComplete setSelectedSegmentIndex:(isComplete ? 0 : 1)];

        [self.itemComplete addTarget:self
                              action:@selector(completedValueChanged:)
                    forControlEvents:UIControlEventValueChanged];
    }

    - (void)completedValueChanged:(id)sender {
        [[self view] endEditing:YES];
    }

    - (void)viewWillDisappear:(BOOL)animated {
        [self.item setValue:[self.itemText text] forKey:@"text"];
        [self.item setValue:[NSNumber numberWithBool:self.itemComplete.selectedSegmentIndex == 0] forKey:@"complete"];
    }

    - (void)didReceiveMemoryWarning
    {
        [super didReceiveMemoryWarning];
        // Dispose of any resources that can be recreated.
    }

    - (BOOL)textFieldShouldEndEditing:(UITextField *)textField {
        [textField resignFirstResponder];
        return YES;
    }

    - (BOOL)textFieldShouldReturn:(UITextField *)textField {
        [textField resignFirstResponder];
        return YES;
    }

    @end

Now return to the storyboard and add a new view controller to the right of the list view. In the identity inspector, select the class added below as the custom class for the view controller. Then add a new push segue from the master view controller to the detail view controller, naming the segue "detailSegue". Add the appropriate fields in the view controller (a text field for the item text; a segmented control for the complete status) and link them to the outlets in the class.

![Item details view controller](images/005-ItemDetailViewController.png)

At this point you should be able to run the app and when selecting an item in the table, it will show the (currently empty) details view controller.

### Filling the detail view

Now that we have the storyboard ready, we need to pass the item from the master to the detail view, and update it once we're back. Let's first remove some methods which we won't need anymore: `tableView:titleForDeleteConfirmationButtonForRowAtIndexPath:`, `tableView:editingStyleForRowAtIndexPath:` and `tableView:commitEditingStyle:forRowAtIndexPath:`. Now add two new properties which we'll use to store the item which is being edited: `editedItem` and `editedItemIndex`.

    @interface QSTodoListViewController ()

    // Private properties
    @property (strong, nonatomic)   QSTodoService   *todoService;
    @property (nonatomic)           BOOL            useRefreshControl;
    @property (nonatomic)           NSInteger       editedItemIndex;
    @property (strong, nonatomic)   NSMutableDictionary *editedItem;

    @end

Now implement the `tableView:didSelectRowAtIndexPath:` method to save the item being edited, and call the segue to display the detail view.

    - (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
        self.editedItemIndex = [indexPath row];
        self.editedItem = [[self.todoService.items objectAtIndex:[indexPath row]] mutableCopy];

        [self performSegueWithIdentifier:@"detailSegue" sender:self];
    }

Finally, implement the `prepareForSegue:sender:` selector to pass the item to the next controller.

    - (void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender {
        if ([[segue identifier] isEqualToString:@"detailSegue"]) {
            QSTodoItemViewController *ivc = (QSTodoItemViewController *)[segue destinationViewController];
            ivc.item = self.editedItem;
        }
    }

You should now be able to run the app and see the item displayed in the detail view.

### Saving the edits

When you click the "Back" button in the navigation view the edits are lost. We've sent data to the detail view, but the data isn't being sent back to the master. Ideally we'd implement a delegate in the master view controller so that it can be notified when edits are done, but since we already passed a pointer to a copy of the item that is being edited, we can use that pointer to retrieve the list of updates made to the item and update it in the server.

But before changing the todo list view controller, we need to update the server wrapper class (`QSTodoService`), since it doesn't have a method to update items (it only has a method to mark an item as complete). Remove the method `completeItem:completion:`, and add the method below (declaration in the header file omitted)

    - (void)updateItem:(NSDictionary *)item atIndex:(NSInteger)index completion:(QSCompletionWithIndexBlock)completion {
        // Cast the public items property to the mutable type (it was created as mutable)
        NSMutableArray *mutableItems = (NSMutableArray *) items;

        // Replace the original in the items array
        [mutableItems replaceObjectAtIndex:index withObject:item];

        // Update the item in the TodoItem table and remove from the items array on completion
        [self.table update:item completion:^(NSDictionary *updatedItem, NSError *error) {
            [self logErrorIfNotNil:error];

            NSInteger index = -1;
            if (!error) {
                BOOL isComplete = [[updatedItem objectForKey:@"complete"] boolValue];
                NSString *remoteId = [updatedItem objectForKey:@"id"];
                index = [items indexOfObjectPassingTest:^BOOL(id obj, NSUInteger idx, BOOL *stop) {
                    return [remoteId isEqualToString:[obj objectForKey:@"id"]];
                }];

                if (index != NSNotFound && isComplete)
                {
                    [mutableItems removeObjectAtIndex:index];
                }
            }

            // Let the caller know that we have finished
            completion(index);
        }];
    }

Now that we can update items on the server, we can implement the `viewWillAppear:` method to call the update method when the master view is being displayed while returning from the details view controller.

    - (void)viewWillAppear:(BOOL)animated {
        if (self.editedItem && self.editedItemIndex >= 0) {
            // Returning from the details view controller
            NSDictionary *item = [self.todoService.items objectAtIndex:self.editedItemIndex];

            BOOL changed = ![item isEqualToDictionary:self.editedItem];
            if (changed) {
                [self.tableView setUserInteractionEnabled:NO];

                // Change the appearance to look greyed out until we remove the item
                NSIndexPath *indexPath = [NSIndexPath indexPathForRow:self.editedItemIndex inSection:0];

                UITableViewCell *cell = [self.tableView cellForRowAtIndexPath:indexPath];
                cell.textLabel.textColor = [UIColor grayColor];

                // Ask the todoService to update the item, and remove the row if it's been completed
                [self.todoService updateItem:self.editedItem atIndex:self.editedItemIndex completion:^(NSUInteger index) {
                    if ([[self.editedItem objectForKey:@"complete"] boolValue]) {
                        // Remove the row from the UITableView
                        [self.tableView deleteRowsAtIndexPaths:@[ indexPath ]
                                              withRowAnimation:UITableViewRowAnimationTop];
                    } else {
                        [self.tableView reloadRowsAtIndexPaths:[NSArray arrayWithObject:indexPath]
                                              withRowAnimation:UITableViewRowAnimationAutomatic];
                    }

                    [self.tableView setUserInteractionEnabled:YES];

                    self.editedItem = nil;
                    self.editedItemIndex = -1;
                }];
            } else {
                self.editedItem = nil;
                self.editedItemIndex = -1;
            }
        }
    }

And we're done with the preparation. The quick start project has been updated to allow edits, so let's finally get to the point of this post.

## Updating the framework

The first thing we'll need to do to add offline support in our application is to get a version of the Mobile Services iOS SDK which supports it. Since we're launching it as a preview feature, it won't be in the official download location. So go to **ADD THE LINK TO THE FRAMEWORK HERE** and download it locally.

Then, remove the existing framework from the project in Xcode, selecting "Move to Trash" to really delete the files.

![Remove existing Mobile Services framework](images/006-RemovePreviousVersionOfFramework.png)

And add in its place the new one which you get after unzipping the file downloaded above (make sure the "Copy items into destination group's folder (if needed)" is selected).

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