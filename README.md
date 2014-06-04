# Laravel & IOS
We are going to build an app that will read and write to a Laravel API. 

# Create App

Create a new Laravel app called noteApp.
`composer create-project laravel/laravel noteApp --prefer-dist`

Add the [Way Generators](https://github.com/vanderlin/LaravelTutorial/blob/master/readme.md#add-ways-generators) 

Update composer.
`composer update`

Turn on MAMP and test the site. 

# Setup Database
open `/app/config/database.php` and set you mysql credentials. We are using MAMP so just use the defaults. [Create](https://github.com/vanderlin/LaravelTutorial/blob/master/readme.md#mamp-database) or use an existing database locally. 
> Note: If you are using a existing database its good practice to set a prefix in the `database.php`.  

You should have something like this.
```
'mysql' => array(
			'driver'    => 'mysql',
			'host'      => 'localhost',
			'database'  => 'app',
			'username'  => 'root',
			'password'  => 'root',
			'charset'   => 'utf8',
			'collation' => 'utf8_unicode_ci',
			'prefix'    => 'notes_',
		),
```

# Build a Resource
We are going to create an app that creates notes, so we want to create a new resource call `note`.

Run.
`php artisan generate:resource note --fields="body:text"`

Open `app/routes.php` and a the new resource. 
`Route::Resource('notes', 'NotesController');`

Test your new routes. In terminal type
`php artisan routes`	
Your should see all the new verbs for creating/updating/deleting `notes`.

Open `app/controllers/notesController.php` and the the functionality for creating updating and deleting notes.

Your file should look like this.
```
<?php

class NotesController extends \BaseController {

	// GET
	public function index() {
		return Response::json(['notes'=>Note::all()]);
	}

	// POST
	public function store() {
		$note = new Note();
		$note->body = Input::get('body', 'empty note');
		$note->save();

		return Response::json($note);
	}

	// GET
	public function show($id) {
		$note = Note::find($id);
		return $note;
	}

	// PUT
	public function update($id) {
		$note = Note::find($id);
		$note->body = Input::get('body', 'empty note');
		$note->save();
		
		return Response::json($note);
	}

	// DELETE 
	public function destroy($id) {
		$note = Note::find($id);
		$note->delete();
		return Response::json($note);
	}

}
```

# Test the API.
- In [Chrome](https://www.google.com/intl/en-US/chrome/browser/) get the app called [PostMan](https://chrome.google.com/webstore/detail/postman-rest-client/fdmmgilgnpjigdojojpjoooidkmcomcm?hl=en) this is a nice little REST app for Chrome that allows you to test all the verbs for you API.

Looking at your routes `php artisan routes` you can see how to all the URIs.

**POST** a new `Note`. Add the key `body` and value `my new note`.		
http://laraveldemo:8888/notes

**GET** all the `Notes`. 	
http://localhost:8888/notes

**DELETE** a `Note`.	
http://localhost:8888/notes/1

# IOS
Open XCode and create a new IOS project and select *Single View Application*. Name is NoteApp, select iPhone and save.

### Setup the Main.storyboard
Delete everything in the `Main.storyboard`. Select the file and hit *(⌘+a) then (delete)*		
Delete the `ViewController.h` and `ViewController.m` files.

### Create NotesController
File new. *(⌘+n)*	
Cocoa Touch -> Objective-C class	
Click Next			
Class: NotesController	
Subclass of: UITableViewController			
Click Next		
Create	

Click on the `Main.storyboard` and drag a new `Table View Controller` to the stage. With the Controller selected third button at the top of the inspector. Set the `Custom Class` to `NotesController`. Now in the top menu click *Editor -> Embed In -> Navigation Controller*

Hit  *(⌘+r)* and test the app. Everything should be connected and you should see a TableViewController in the simulator. 

### Load the data from the server
We are going to write all the loading functions in the `AppDelegate` this is good practice so that we can access these functions throughout the app.

In the `AppDelegate.h` add  `#define BASE_URL @"http://localhost:8888"` this will save us from having to type this a bunch and easy to set for production.	

**Singleton AppDelegate**	
This is a nice helper function to get the `AppDelegate` instance. Create this static method.

**AppDelegate.h**	
`+(AppDelegate*)getInstance;`	

**Load Notes**	
```
+(AppDelegate*)getInstance {
    return (AppDelegate*)[[UIApplication sharedApplication] delegate];
}
```
	
We can now call this simply by typing `[AppDelegate getInstance]`	

Lets make a function that will load any url with a http method. We also want to have a callback for when the data is done loading. We are going to use a `Block`.

in your `AppDelegate.h` create a `typedef` of the `Block`.	
`typedef void(^RequestBlock)(NSDictionary*data);`

add this function.	
`-(void)makeRequest:(NSString*)urlString method:(NSString*)method onComplete:(RequestBlock)callback;`

**AppDelegate.m**	
```
-(void)makeRequest:(NSString*)urlString method:(NSString*)method onComplete:(RequestBlock)callback {

    // create the url
    NSURL * url = [NSURL URLWithString:[NSString stringWithFormat:@"%@/%@", BASE_URL, urlString]];
    NSMutableURLRequest * request = [[NSMutableURLRequest alloc] initWithURL:url];
    
    // set the method (GET/POST/PUT/UPDATE/DELETE)
    [request setHTTPMethod:method];
    
    // submit the request
    [NSURLConnection sendAsynchronousRequest:request
                                       queue:[NSOperationQueue mainQueue]
                           completionHandler:^(NSURLResponse *response, NSData *data, NSError *connectionError) {

                               // do we have data?
                               if(data && data.length > 0) {
                                   
                                   NSDictionary * json = [NSJSONSerialization JSONObjectWithData:data options:kNilOptions error:nil];
                                  
                                   // if we have a block lets pass it
                                   if(callback) {
                                       callback(json);
                                   }
                                   
                               }
                               
                           }];
    
}
```

Now lets test this function. In `AppDelegate.m`	 `didFinishLaunchingWithOptions` add the function to load all the notes.

```
    [self makeRequest:@"notes" method:@"GET" onComplete:^(NSDictionary *data) {
        NSLog(@"Json Loaded: %@", data);
    }];
```
Run & Build. You will see output in the console of the a `NSDictionary` of all the notes. 

# Load Notes - NotesController

First include the `AppDelegate` in the `NotesController`

**NotesController.h**
`#import "AppDelegate.h"`

We want to have a `NSMutableArray` of all the notes. 
**NotesController.h**
`@property (strong, nonatomic) NSMutableArray * notes;`
**NotesController.m**
`@synthesize notes;`

Load the notes when the view loads.
```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    [[AppDelegate getInstance] makeRequest:@"notes" method:@"GET" onComplete:^(NSDictionary *data) {

        if([data objectForKey:@"notes"]) {
            notes = [NSMutableArray arrayWithArray:[data objectForKey:@"notes"]];

            // now reload the tableview
            [self.tableView reloadData];
        }
        
    }];
}
```

**Setup The TableView**
Now that we have the data loaded we can set the sections and number of rows.

```
- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView {
    return notes ? 1 : 0; // safe way to not load rows if notes is nil
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return notes.count;
}
```

Setup the cell.
```
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
   
    static NSString * cellID = @"NOTES_CELL";
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:cellID];
    
    if(cell == nil) {
        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleSubtitle reuseIdentifier:cellID];
    }
    
    // get the note
    NSDictionary * note = [notes objectAtIndex:indexPath.row];
    
   // create a date formatter so we can display the date
    NSDateFormatter * df = [[NSDateFormatter alloc] init];
    [df setDateFormat:@"yyyy-MM-dd H:mm:ss"];
    
    // make a date object from the timestamp
    NSDate * date = [df dateFromString:[note objectForKey:@"created_at"]];

    // change the format weekday/month/day/year
    [df setDateFormat:@"EE MMM d yyyy"];
    NSString * dateStr = [df stringFromDate:date];

    // change the format hour:min
    [df setDateFormat:@"h:mm a"];
    NSString * timeStr = [df stringFromDate:date];

    // update the cell
    cell.textLabel.text = dateStr;
    cell.detailTextLabel.text = timeStr;
    
    
    return cell;
}
```

At this point we are loading the `Notes` and able to display them in our tableview -Yeah! Now we need to make some notes.

# Create Notes
Lets add right bar button item that will launch a note creator view. 

**Add the bar button**
In the `viewDidLoad` add the following code. 
```
 UIBarButtonItem * noteButton = [[UIBarButtonItem alloc] initWithBarButtonSystemItem:UIBarButtonSystemItemAdd
                                                                                 target:self
                                                                                 action:@selector(openNoteCreator:)];
    self.navigationItem.rightBarButtonItem = noteButton;
```

And add this function
```
-(void)openNoteCreator:(id)sender {
    
}
```

Run the app and you will see a (+) icon in the top right. Let create the `NoteCreatorController`. Like before create a new class that subclasses a `UIViewController`.

Connect the `NoteCreatorController` in the `NotesController`.

Import the class
** NotesController.m**
`#import "NoteCreatorController.h"`

Update the `openNoteCreator` method.
```
-(void)openNoteCreator:(id)sender {
    NoteCreatorController * noteVC = [[NoteCreatorController alloc] init];
    [self.navigationController presentViewController:noteVC animated:YES completion:nil];
}
```

### Build the note creator
We need a few things in the class. Buttons to close the modal and save the note as well a textview to write the note. 



















