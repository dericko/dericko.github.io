---
layout: post
title: "Multiple Fetched Results Controllers in a Table View"
description: "After a lot of searching and a bit of experimentation, I finally implemented a table view with two fetched results controllers."
categories: blog
tags: ios tech
---

After a lot of searching and a bit of experimentation, I finally implemented a table view with two fetched results controllers.

{% highlight objective-c %}

@interface SplitTableView : UITableViewController <NSFetchedResultsControllerDelegate>

// Managed object context for fetch requests
@property (nonatomic, strong) NSManagedObjectContext *managedObjectContext;

// Two fetched results controller to populate separate sections
@property (nonatomic, strong) NSFetchedResultsController *fetchedResults1;
@property (nonatomic, strong) NSFetchedResultsController *fetchedResults2;

@end

{% endhighlight %}

I'm using two networking client managers, one for articles, one for projects (e.g. from separate base urls, with different underlying entities)

{% highlight objective-c %}

@interface SplitTableView ()

@property (nonatomic, strong) ArticleManager *articleManager;
@property (nonatomic, strong) Article *article;

@property (nonatomic, strong) ProjectManager *projectManager;
@property (nonatomic, strong) Project *project;

@property int numberToLoad;

@end

@implementation SplitTableView

- (void)viewDidLoad
{
    [super viewDidLoad];
    
    // Setup networking managers
    _articleManager = [ArticleManager sharedManager];
    _projectManager = [ProjectManager sharedManager];
    
    // Setup core data stack
    _articleManager.managedObjectStore = [RKManagedObjectStore defaultStore];
    _projectManager.managedObjectStore = [RKManagedObjectStore defaultStore];
    self.managedObjectContext = [RKManagedObjectStore defaultStore].mainQueueManagedObjectContext;
    
    // Set up request
    self.numberToLoad = 3;
    
    // GET content
    [self getData];
}

- (void)getData
{
    NSString *numberToLoad = [NSString stringWithFormat:@"%d", self.numberToLoad];
    NSDictionary *parameters = @{@"limit": numberToLoad};
    
    if (_articleManager) {
        [self controllerWillChangeContent:_fetchedResults1];
        [_articleManager
         loadContentWithParameters:parameters
         success:^(void) {
             // Success!
         } failure:^(NSError *error) {
             // Show Error or do something about it!
         }];
    }
    
    if (_projectManager) {
        [self controllerWillChangeContent:_fetchedResults2];
        [_projectManager
         loadContentWithParameters:parameters
         success:^(void) {
             // Success!
         } failure:^(NSError *error) {
             // Show Error or do something about it!
         }];
    }
}

{% endhighlight %}

Some major changes to table views, all about maintaining a total section count of 3, with each fetchedResultsController working as if its in section[0]

{% highlight objective-c %}

#pragma mark - Table view data source

- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView
{
    // Should equal three (2 fetched results sections + 1 top section for menu)
    NSLog(@"--sections in FR1: %d", [[self.fetchedResults1 sections] count]);
    NSLog(@"--sections in FR2: %d", [[self.fetchedResults2 sections] count]);
    return [[self.fetchedResults1 sections] count] + [[self.fetchedResults2 sections] count] + 1;
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section
{
    // Return the number of rows in the section.
    if (section == 0) {
        return 1;
    } else if (section == 1) {
        // Shift fetched results controller by one (should be 0)
        id <NSFetchedResultsSectionInfo> sectionInfo = [[self.fetchedResults1 sections] objectAtIndex:section - 1];
        NSLog(@"--number of objects=%d for section %d", [sectionInfo numberOfObjects], section);
        // Cap the number of rows at the predined limit of "numberToLoad"
        return ([sectionInfo numberOfObjects] > self.numberToLoad) ? self.numberToLoad : [sectionInfo numberOfObjects];
        
    } else if (section == 2) {
        // Shift fetched results controller by two (should be 0)
        id <NSFetchedResultsSectionInfo> sectionInfo = [[self.fetchedResults2 sections] objectAtIndex:section - 2];
        NSLog(@"--number of objects=%d for section %d", [sectionInfo numberOfObjects], section);
        // Cap the number of rows at the predined limit of "numberToLoad"
        return ([sectionInfo numberOfObjects] > self.numberToLoad) ? self.numberToLoad : [sectionInfo numberOfObjects];
    }
    
    NSLog(@"Failed to set up number of rows for section %d", section);
    return 0;
}


- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    UITableViewCell *cell = [UITableViewCell new];
    
    NSLog(@"--number of sections: %d", self.tableView.numberOfSections);
    
    switch (indexPath.section) {
        case 0:
            cell = [tableView dequeueReusableCellWithIdentifier:@"menuCell" forIndexPath:indexPath];
            break;
        case 1:
            cell = [tableView dequeueReusableCellWithIdentifier:@"articleCell" forIndexPath:indexPath];
            break;
        case 2:
            cell = [tableView dequeueReusableCellWithIdentifier:@"projectCell" forIndexPath:indexPath];
            break;
        default:
            break;
    }
    
    [self configureCell:cell atIndexPath:indexPath];
    
    return cell;
}

- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath
{
    switch (indexPath.section) {
        case 0:
            return 80;
        default:
            return 70;
    }
}

- (void)configureCell:(UITableViewCell *)cell atIndexPath:(NSIndexPath *)indexPath
{
    if (indexPath.section == 1) {
        NSIndexPath *indexPath1 = [NSIndexPath indexPathForRow:indexPath.row inSection:indexPath.section - 1];

        _article = [self.fetchedResults1 objectAtIndexPath:indexPath1];
        
        // Style up the Article Cell

    } else if (indexPath.section == 2) {
        NSIndexPath *indexPath2 = [NSIndexPath indexPathForRow:indexPath.row inSection:indexPath.section - 2];
        _project = [self.fetchedResults2 objectAtIndexPath:indexPath2];
        
        // Style up the Project Cell
    }
}

{% endhighlight %}

The fetchedResultsControllers are basically the same, there are just two of them...

{% highlight objective-c %}

#pragma mark - Fetched results controller

- (NSFetchedResultsController *)fetchedResults1
{
    if (_fetchedResults1 != nil) {
        return _fetchedResults1;
    }

    NSFetchRequest *fetchRequest = [[NSFetchRequest alloc] init];
    
    // Assign Core Data entity for fetchedResultsController
    NSEntityDescription *entity = [NSEntityDescription entityForName:@"Article" inManagedObjectContext:self.managedObjectContext];
    [fetchRequest setEntity:entity];
    
    // Assign sort descriptor (one of entity's properties)

    NSSortDescriptor *sortDescriptor = [[NSSortDescriptor alloc] initWithKey:@"postID" ascending:NO];
    NSArray *sortDescriptors = @[sortDescriptor];
    [fetchRequest setSortDescriptors:sortDescriptors];
    
    // Assign section key path and cache
    // Edit the section name key path and cache name if appropriate.
    // nil for section name key path means "no sections".
    NSFetchedResultsController *aFetchedResultsController =
    [[NSFetchedResultsController alloc] initWithFetchRequest:fetchRequest
                                        managedObjectContext:self.managedObjectContext
                                          sectionNameKeyPath:nil
                                                   cacheName:@"Master"];
    aFetchedResultsController.delegate = self;
    self.fetchedResults1 = aFetchedResultsController;
    
	NSError *error = nil;
	if (![self.fetchedResults1 performFetch:&error]) {
        // Replace this implementation with code to handle the error appropriately.
        // abort() causes the application to generate a crash log and terminate. You should not use this function in a shipping application, although it may be useful during development.
	    NSLog(@"Unresolved error %@, %@", error, [error userInfo]);
	    abort();
	}
    
    return _fetchedResults1;
}

- (NSFetchedResultsController *)fetchedResults2
{
    if (_fetchedResults2 != nil) {
        return _fetchedResults2;
    }
    
    NSFetchRequest *fetchRequest = [[NSFetchRequest alloc] init];
    
    // Assign Core Data entity for fetchedResultsController
    NSEntityDescription *entity = [NSEntityDescription entityForName:@"Project" inManagedObjectContext:self.managedObjectContext];
    [fetchRequest setEntity:entity];
    
    // Assign sort descriptor (one of entity's properties)
    
    NSSortDescriptor *sortDescriptor = [[NSSortDescriptor alloc] initWithKey:@"opportunityID" ascending:NO];
    NSArray *sortDescriptors = @[sortDescriptor];
    [fetchRequest setSortDescriptors:sortDescriptors];
    
    // Assign section key path and cache
    // Edit the section name key path and cache name if appropriate.
    // nil for section name key path means "no sections".
    NSFetchedResultsController *aFetchedResultsController =
    [[NSFetchedResultsController alloc] initWithFetchRequest:fetchRequest
                                        managedObjectContext:self.managedObjectContext
                                          sectionNameKeyPath:nil
                                                   cacheName:@"Master"];
    aFetchedResultsController.delegate = self;
    self.fetchedResults2 = aFetchedResultsController;
    
	NSError *error = nil;
	if (![self.fetchedResults2 performFetch:&error]) {
        // Replace this implementation with code to handle the error appropriately.
        // abort() causes the application to generate a crash log and terminate. You should not use this function in a shipping application, although it may be useful during development.
	    NSLog(@"Unresolved error %@, %@", error, [error userInfo]);
	    abort();
	}
    
    return _fetchedResults2;
}

{% endhighlight %}

It's the section-handling methods that change. We need to re-add the section indices to make sure everything knows where its supposed to be

{% highlight objective-c %}

- (void)controller:(NSFetchedResultsController *)controller didChangeSection:(id <NSFetchedResultsSectionInfo>)sectionInfo
           atIndex:(NSUInteger)sectionIndex forChangeType:(NSFetchedResultsChangeType)type
{
    if (controller == _fetchedResults1) {
        sectionIndex += 1;
    } else if (controller == _fetchedResults2) {
        sectionIndex += 2;
    }
    
    switch(type) {
        case NSFetchedResultsChangeInsert:
            [self.tableView insertSections:[NSIndexSet indexSetWithIndex:sectionIndex] withRowAnimation:UITableViewRowAnimationFade];
            break;
            
        case NSFetchedResultsChangeDelete:
            [self.tableView deleteSections:[NSIndexSet indexSetWithIndex:sectionIndex] withRowAnimation:UITableViewRowAnimationFade];
            break;
    }
}

- (void)controller:(NSFetchedResultsController *)controller didChangeObject:(id)anObject
       atIndexPath:(NSIndexPath *)indexPath forChangeType:(NSFetchedResultsChangeType)type
      newIndexPath:(NSIndexPath *)newIndexPath
{
    UITableView *tableView = self.tableView;
    NSIndexPath *adjustedIndexPath = nil;
    NSIndexPath *adjustedNewIndexPath = nil;
    
    if (controller == _fetchedResults1) {
        adjustedIndexPath = [NSIndexPath indexPathForRow:indexPath.row inSection:indexPath.section + 1];
        adjustedNewIndexPath = [NSIndexPath indexPathForRow:newIndexPath.row inSection:newIndexPath.section + 1];
    } else if (controller == _fetchedResults2) {
        adjustedIndexPath = [NSIndexPath indexPathForRow:indexPath.row inSection:indexPath.section + 2];
        adjustedNewIndexPath = [NSIndexPath indexPathForRow:newIndexPath.row inSection:newIndexPath.section + 2];
    }
    
    if (adjustedIndexPath && adjustedNewIndexPath) {
        switch(type) {
            case NSFetchedResultsChangeInsert:
                [tableView insertRowsAtIndexPaths:@[adjustedNewIndexPath] withRowAnimation:UITableViewRowAnimationFade];
                break;
                
            case NSFetchedResultsChangeDelete:
                [tableView deleteRowsAtIndexPaths:@[adjustedIndexPath] withRowAnimation:UITableViewRowAnimationFade];
                break;
                
            case NSFetchedResultsChangeUpdate:
                [self configureCell:[tableView cellForRowAtIndexPath:adjustedIndexPath] atIndexPath:adjustedIndexPath];
                break;
                
            case NSFetchedResultsChangeMove:
                [tableView deleteRowsAtIndexPaths:@[adjustedIndexPath] withRowAnimation:UITableViewRowAnimationFade];
                [tableView insertRowsAtIndexPaths:@[adjustedNewIndexPath] withRowAnimation:UITableViewRowAnimationFade];
                break;
        }
    } else {
        NSLog(@"Did not update NSFetchedResults change. Check NSFetchedResultsControllerDelegate methods");
    }
}

- (void)controllerWillChangeContent:(NSFetchedResultsController *)controller
{
    [self.tableView beginUpdates];
}

- (void)controllerDidChangeContent:(NSFetchedResultsController *)controller
{
    [self.tableView endUpdates];
}

{% endhighlight %}

And a few changes in navigation control

{% highlight objective-c %}

#pragma mark - Navigation

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath
{
    if (indexPath.section == 1) {
        NSString *segueIdentifier = @"showArticleDetail";
        [self performSegueWithIdentifier:segueIdentifier sender:[self.tableView cellForRowAtIndexPath:indexPath]];
    } else if (indexPath.section == 2) {
        NSString *segueIdentifier = @"showProjectDetail";
        [self performSegueWithIdentifier:segueIdentifier sender:[self.tableView cellForRowAtIndexPath:indexPath]];
    }
}

- (void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender
{
    
    if ([segue.identifier isEqualToString:@"showArticleDetail"]) {
         NSIndexPath *indexPath = [NSIndexPath indexPathForRow:[self.tableView indexPathForSelectedRow].row inSection:[self.tableView indexPathForSelectedRow].section - 1];
        
        // Pass the Article object and anything else
    }
    
    else if ([segue.identifier isEqualToString:@"showProjectDetail"]) {
         NSIndexPath *indexPath = [NSIndexPath indexPathForRow:[self.tableView indexPathForSelectedRow].row inSection:[self.tableView indexPathForSelectedRow].section - 2];
        
        // Pass the Project object and anything else
        
    }
    
    [segue.destinationViewController setHidesBottomBarWhenPushed:YES];
}


@end

{% endhighlight %}

And that's it! Here's an example:

(insert screenshot)