# CoreDataToDoist (Lab)
## Intro
- If you look at the `main` branch you'll see a lot of code already. In fact, this is a functioning to-do app already! 
- But we can make it better!
- For starters, this app doesn't save the user's to-do items accross launches. How useful is that?
- For this lab, you will be helping the user persist their data accross launches using (you guessed it) CORE DATA
- Before we dive into Core Data though, familiarize yourself with the project. 

## Get to know the project
  - 🔨🏃‍♂️ Build and Run
  - Type some to-do items
  - check any number of them off as completed
  - Check out the code!
    - Focus on the data flow
    - Where do the Items live?
    - How do you create a new Item?
    - Especially look at the `ItemManager` That's where we'll do a lot of our work
      - Right now its simple because all the data is stored in a local array of `[Item]`
      - But pretty soon it will utilize Core Data
  
## Add Core Data Model
- Lets do some modeling. . . Lets start with `Blue Steel` and then move on to `Magnum` jk 😜
- There's already a model file in the project but no entities to be found (Its called `ToDoist` with the little database icon next to it)
- Add a new entity called `Item`
  - Make it match the `Item` found in `Item.swift`
    - id: String
    - title: String
    - createdAt: Date
    - completedAt: Date
  - Set the code generation to `Class Definition`
- 🔨🏃‍♂️ Build and run
- You'll notice some compile errors. What's going on?
  - Remember what `Class Definition` does? It creates a secret class definition of `Item` and now there are two objects called `Item` and the compiler is not happy!
- Go over to `Item.swift`  and lets make some changes:
  - Delete the struct definition, and the 4 properties and replace it with this line:
  ```
  extension Item { 
  ```
  - Delete all the other code that says  `// Remove for Core Data`
  - You'll notice you still can't build and run. A Core Data subclass is a lot different than a swift struct. We'll need to jump through the `Core Data` hoops to get this app working again. 
  - Onward!
  
## Add the Stack
- To access the guts of Core Data you have to have an `NSPersistentContainer` and an `NSPersistentStoreCoordinator` and a handful of other long names that start with `NS`
  - And remember we DON'T expect a 10 page essay on the inner workings of the Core Data Stack
  - We just expect you to know that there's a core data stack that gives you access to the `ManagedObjectContext` which is where all your work will happen. 
  - If you want to learn more about whats happening under the hood, you can look at the lesson one slides, and read the apple documentation [here](https://developer.apple.com/documentation/coredata) which is actually quite good. 
- Go check out the projects `AppDelegate.swift`
- You'll find a bunch of code at the bottom of the file under the `// MARK: - Core Data stack`
- That is a full Core Data Stack Apple provides as part of your project. That's one of the things you get when you check the `Core Data` box when the project is created
- Create the Stack
  - Ceate a new swift file called `PersistenceController` 
  - Create a class with the same name
  - `import CoreData` at the top of the file
  - Add a shared instance (`static let shared = PersistenceController()`)
  - Copy all the code in the `AppDelegate` under the `MARK: - Core Data Stack`
  - paste it into the `Persistence` class
  - (make sure to not take too many `{` and `}` with you)
- Get access to the context:
  - Now lets write a computed property to be able to access the context directly
  - Add this above the `persistenceContainer`
  ```
  var viewContext: NSManagedObjectContext {
    return persistentContainer.viewContext
  }
  ```
  - When we need the use the viewContext, we'll use `PersistenceController.shared.viewContext`
  - Let's put that context to use. We'll use the context in creating, retreiving, updating, and deleting to do items
  
## Crud (Create)
- Go to the `ItemManager` and check out the `createNewItem()` function
- Right now it initializes an Item, and append it to the array of items. 
- But you can't just initialize a subclass of `NSManagedObject` like you would a swift struct. You have to use a designated initializer.
- Remove everything after the equals on this line:  `let newItem = Item(title: title)`, and then retype `Item(` to see the autocomplete suggestions
- Use the one that takes a `context`
- So the line will look like this: 
```
let newItem = Item(context: PersistenceController.shared.viewContext)
```
This creates a new item, but doesn't fill in all of its properties or save it to the persistance controller. It only adds it to your scratch pad (Managed Object context) Let's do that next. 
```
newItem.id = UUID().uuidString
newItem.title = title
newItem.createdAt = Date()
newItem.completedAt = nil
```
- So we've created the item in the viewContext, and added the properties but they're just sitting there on the context. Remember the context is like your working scratch pad. We need to take the things on the scratch pad and save them. 
`PersistenceController.shared.saveContext()`
- That's it! We're creating new items and storing them in core data  
- If you build and run, you will be adding items into Core Data, but your list of items is still based on the local array of item, so they won't show up
- Lets do that next
  
## cRud (Retrieve)
- Q: How do you get data out of core data?
- A: Fetch Requests
- A Fetch Request is a request for a specific subset of Core Data entities. 
- Lets write a Fetch Request to get `Item`s stored in Core Data based on a predicate.

- First, let's import CoreData in the `ItemManager`
- Now lets add a function that will fetch `Item`s given a predicate
  - First you create the fetch request
  - Then use the predicate (Like a filter) of which specific entities you want (we'll talk briefly about predicates in a second)
  - Then you execute the fetch request on a `NSManagedObjectContext` (we'll use the viewContext)
- Here's what it should look like in the end: 
```
    private func fetchItems(matching predicate: NSPredicate) -> [Item] {
        // Create the fetch request
        let fetchRequest = Item.fetchRequest()
        // Add the predicate (kind of like a filter)
        fetchRequest.predicate = predicate        
        do {
            let context = PersistenceController.shared.viewContext
            // Perform the request on the MOC
            return try context.fetch(fetchRequest)
        } catch {
            print("Error fetching items: \(error)")
            return []
        }
    }

```
- Now making a function to fetch incomplete items is basically just figuring out the correct predicate. 
### Predicates Detour
 - NSPredicate could be a whole day in itself! 
- Predicates are basically filters. We're saying, fetch the data, but only the ones that match this predicate (or specific filter requirements)
  - We won't cover it right now but you could check out [this article](https://nshipster.com/nspredicate/) which is an oldie but a goodie
 - For incomplete items, for example, we want all `Item`s but really we only want the items whose `completedAt` property is nil (not yet completed)
 - So we will call the `fetchItems` function and use this predicate: `completedAt == nil`
  - It should look like this:
  ```
      func fetchIncompleteItems() -> [Item] {
        return fetchItems(matching: NSPredicate(format: "completedAt == nil"))
    }

  ```
- Do the same for completed items (`fetchCompletedItems`) see if you can figure out the predicate on your own (hint, you only have to change one character)

  ### (Back to fetching)
- Now lets head over to the `ItemsViewController` and fix the errors
- Instead of populating the `Item`s from the array of items (that we deleted) we're going to use the new `fetchIncompleteItems` function we just wrote to get our items from Core Data 🎉
- Go to the top of `ItemsViewController` and create a local variable called `incompleteItems` that we will use as our new source of truth for incompleteItems
```
var incompleteItems = [Item]()
```
- Find all the instances of `itemManager.incompleteItems()` and replace it with `incompleteItems` in the following functions:
  - item(at indexPath)
  - title(for header)
  - numberOfRows(in section)
- Then make a new function in the `ItemsViewController` called `refreshData` that will refetch the items from the `ItemManager`, set the result to this new `incompleteItems` property, and reload the table
  - Only call `tableView.reloadData` inside this new `refreshData` function. Replace any other call to reloadData with `refreshData`
- 🔨🏃‍♂️ Build and run! (You will need to comment out the code in `ItemManager.toggleItemCompleted(item:)` and `ItemManager.remove(item:)`)
- Assuming you did everything correctly, and if you added any items earlier before they were showing up, they should show up now! They were saved in Core Data and persist accross launches now! 
- Do the same with completed items by adding a local variable `completedItems` populating the data with this variable, and refetching completed items in `refreshData`
- What if we want the newest item to show up at the top?
- Queue: Sort Descriptors

### Sorting
- Fetch Requests can have a sort descriptor to indicate which order the data should come back in
- Sort descriptors look like this: 
- `NSSortDescriptor(key: "createdAt", ascending: false)`
- Basically you give it a key of the property you want the items to be sorted on, and if they should be ordered ascending (1->10) or descrending (10->1)
- You add them to the fetch request like this:
- `fetchRequest.sortDescriptors = [sortDescriptor]`
- Make sure to add it the `fetchCompletedItems` function as well (but with `completedAt`)
- And voila, sorted by the right date

## crUd (Update)
- You'll notice the button to mark an item as completed does not work. That's because we commented out the code.
- Let's refactor it for Core Data
- Updating is easy, just make a change to the entity, and save the context.
- Should look like this:
  ```
    func toggleItemCompletion(_ item: Item) {
      item.completedAt = item.isCompleted ? nil : Date()
      PersistenceController.shared.saveContext()
    }

  ```

## cruD (Delete)
- Lets buy the last letter to solve the puzzle
- Deletion is simple as well. 
- Just get the context, and call `.delete()` on it
- Should look like this: 
  ```
      func remove(_ item: Item) {
        let context = PersistenceController.shared.viewContext
        context.delete(item)
        PersistenceController.shared.saveContext()
    }
  ```
  - You can delete the other functions since we will only be using this one. (`delete(at indexPath)`, and item(at `indexPath)`)

## That's it!
- Make sure you're using Core data to Create, Retrieve, Update, and Delete `Items`
- Make sure you can add items, see the items you added, remove items, and toggle them complete + incomplete
- Feel free to look at the other branch `complete` for a completed version of the app
# ToDoList
