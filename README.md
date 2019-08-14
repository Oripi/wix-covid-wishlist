# wix-covid-wishlist
Getting started with Wix Corvid

# What does this walkthrough offer?
In this walkthrough we're going to create our very own Wishlist for a wix website!

we'll learn how to:
* Interact with page elements through Corvid.
* Use some of the various libraries that Corvid exposes to us for manipulating our website.
* Create our own database table and use CRUD (Create, Read, Update, Delete) operations on it.

# So let's get started!
The first thing we need is... a website! so let's start by creating a new website where we can try our code. Since we're creating a Wishlist for an online store, select the "online-store" option when creating a new website. In case you created a different kind of website you can always enable it through the same menu that you add different elements to your website, just under the `store` section.

Now, let's enable Corvid so we'll be able to modify what we need:
![alt enable corvid](images/enableCorvid.png)

Next, let's go to "Product Page" and create an `Add to wishlist` button.
![alt wishlist button](images/addToWishlistButton.png)

### Bonus:
If you wanna be really fancy you can also add a popup that will show once an item has been added to wishlist:
![alt wishlist popup](images/addToWishlistPopup.png)

We'll manipulate it later to show and hide with animations.
Make sure to hide it on page load (this can be done by right clicking on the element, selecting the `View Properties` window from the menu and checking the `Hidden on Load` option).


We're going to need a database to store our Wishlist data, so let's create one.
In the sidebar under `Database` click the `Add a new collection` link and select the `start from scratch` option.
Name the collection `Wishlist` and in permissions select `Member-generated content` so only members will be able to add and remove items from Wishlists.
> After creating the database save the site and refresh the page in order to see native collections, such as `Products` table.

Now we're going to need to add columns to the table to store the `UserId, Product, AddedDate` and any other column that you want your users to fill that can be used later on.
> Do not modify the `Field key` of the column, the `Field key` is generated automatically based on the `Field name` and we're going to use it later to query and display the data. for example the column `UserId` should have a `Field name` named `userId`.

When creating the columns, select following types:
* UserId - Text
* Product - Reference - with a reference to the products table (single reference)
* AddedDate - Date and Time
In case you made a mistake in the column creation you can edit it via the settings of the column. You may get a warning message, but that's ok because we haven't used the table yet.

Finally it's time to write some code!

Select the Product page and open up the dev console by clicking on it in the lower part of the page:
![alt wishlist popup](images/devConsole.png)
You should see the default code:
```javascript
// For full API documentation, including code examples, visit http://wix.to/94BuAAs

$w.onReady(function () {
	//TODO: write your page related code here...

});
```
This function is called when the page is ready and elements are loaded into the page. If we try to access elements outside this function before the page load they won't work, so it's important that we do it here.

Let's modify it by adding an `onClick` event for the button.
```javascript
$w.onReady(function () {
	$w('#AddToWishlistButton').onClick(onWishlistClicked);
});
```
You should see an error in the console, that's because we also need to change the Id of the button to be able to find it and we also need to create a handler for the click.
In the editor right click on the button and select `View properties`. In the window that pops up rename the Id to `AddToWishlistButton` by clicking on it and typing it in the input.
![alt wishlist popup](images/changeId.png)

And let's also create a function that will handle the click event below the `$w.onReady(...)` call.
```javascript
function onWishlistClicked() {
  console.log('hello world');
}
```
If you try to preview the website now you should be able to see the `"hello world"` in the console.

The `$w` and the `ID` field are very important parts in Corvid, In fact this is the way We can interact with elements, as we've seen in the example above. To those of you who are familiar with `jquery` you will recognize the syntax straight away and for those who aren't, the `$w` is the object that allows us to get a reference to an element in the page based on the `ID` that we gaive him in the `ID` field. The syntax for `$w` is: `$w('#<ID>')`, E.g. for `ID: AddToWishlistButton` the call: `$w('#AddToWishlistButton')` will return a reference to the "add to wishlist" button.

In order to insert a new item into the database we need to expose a method from the `backend` that the `client` will invoke and as a result a new item will be added to the Wishlist collection.
In the sidebar, click the `+` while hovering over `backend` or expand it and click on `Add a new web module`. Name the module `wishlist.jsw` and open it.

Open the file and inside it add the following code:
```javascript
// import the wix-users module for working with users.
import wixUsers from 'wix-users-backend';
// import the wix-data module for working with queries.
import wixData from 'wix-data';

const collectionName = 'Wishlist';

export function insertWishlistItem(product) {
	// get the current user
	const user = wixUsers.currentUser;

	// check if user is a logged in member
	if (!user.loggedIn) {
		// if not return an indication that a user is not a member
		return false;
	}
	
	// otherwise insert a new entry to the collection
	return wixData.insert(collectionName, {
		// note the keys are the same as the ID of the column in the collection
		product: product._id,
		addedDate: new Date(),
		userId: user.id
	});
}
```
We've done a couple of things here. First we imported 2 modules: `wix-users-backend, wix-data`, the first is used to handle user status and in our case to check if the current user is a member or not (since only members should be able to update the table). The second is to manipulate collection data and in the above code, to add a new entry to the collection.
> Note that although product is a reference column and you may see the name of the product in the collection view page it is actually referenced by an id, in this case the `_id` field of the product. This is the same case when querying the data, as you'll see soon.

Now we need to call this code from the client, so let's do just that.
Add the following import at the top:
```javascript
import { insertWishlistItem } from 'backend/wishlist.jsw';
```
And back in the product page let's replace the `onClick` handler with the following code:
```javascript
async function onWishlistClicked() {
	// get the current product in the product page
	const product = await $w('#productPage1').getProduct();
	// call the backend function to add the item
	await insertWishlistItem(product);
}
```
> Note: Make sure that the product component Id is `productPage1` or change it to yours.

Now when we click on the button a new item should be added to the collection. You can verify this by previewing the page, clicking the button and then go back to the collection and see if it's added.
> Note: We can also insert to a collection directly from the client side without using a backend code, however this means that we will need to pass the `userId` to the insert method as a parameter and as a result a user maybe be able to insert Wishlist items for other memebers and we don't want to enable that...

But what about the case when our user isn't a member and he clicks the Wishlist button?
Well we can prompt him to sign up instead! let's add the code that does that:
Import the required module at the top:
```javascript
import wixUsers from 'wix-users';
```
And write the code that prompts the user to sign up:
```javascript
async function onWishlistClicked() {
	// if user is not logged in, prompt him to login
	if (!wixUsers.currentUser.loggedIn) {
		await wixUsers.promptLogin();
		return;
	}
	
	// get the current product in the product page
	const product = await $w('#productPage1').getProduct();
	// call the backend function to add the item
	await insertWishlistItem(product);
}
```

We don't want the user to insert the same product multiple times to the Wishlist and we haven't added an option to remove it. We can solve both issues at the same time by coverting the `Add to Wishlist` button to a toggle button that will add or remove an item as necessary.

In order to do that we need 2 new functions in the `backend` to check if an item is in the Wishlist and to remove an item.
Let's add them:
```javascript
export function getItemInWishlist(productId) {
	const user = wixUsers.currentUser;
	// search the collection for item with the same userId and the given productId
	return wixData.query(collectionName)
	  .eq('userId', user.id)
	  // note that the query is done by Id for the reference column
	  .eq('product', productId)
	  .find();
}

export async function removeWishlistItem(productId) {
	const user = wixUsers.currentUser;

	// if user is not a member, don't remove anything
	if (!user.loggedIn) {
		return false;
	}
	const data = await getItemInWishlist(productId);
	
	if (data.items.length === 0) {
		return false;
	}
	return wixData.remove(collectionName, data.items[0]._id);
}
```
And now in the client we need to add the logic for the toggle button.
Again, let's extend the handler function:
```javascript
async function onWishlistClicked() {
	// if user is not logged in, prompt him to login
	if (!wixUsers.currentUser.loggedIn) {
		await wixUsers.promptLogin();
		return;
	}
	
	// get the current product in the product page
	const product = await $w('#productPage1').getProduct();
	// if item is already in list, the click should remove it and show the add button
	if (await isProductInWishlist()) {
		await removeWishlistItem(product._id);
		showAddToWishlistButton();
	} else {
		// otherwise add the product to the wishlist and show the remove button
		await insertWishlistItem(product);
		showRemoveFromWishlistButton();
	}
}
```
And add these functions at the bottom:
```javascript
async function isProductInWishlist() {
	const product = await $w('#productPage1').getProduct();
	const data = await getItemInWishlist(product._id);
	// if there are no item in array result then it's not in the wish list
	return data.items.length > 0;
}

function showAddToWishlistButton() {
	$w('#AddToWishlistButton').label = 'Add to wishlist!';
}

function showRemoveFromWishlistButton() {
	$w('#AddToWishlistButton').label = 'Remove from wishlist';
}
```
And these imports at the top:
```javascript
import { insertWishlistItem, getItemInWishlist, removeWishlistItem } from 'backend/wishlist.jsw';
```

We also need to update the `$w.onReady(...)` to reflect the initial state:
```javascript
$w.onReady(async function () {
	$w('#AddToWishlistButton').onClick(onWishlistClicked);
	if (await isProductInWishlist()) {
		showRemoveFromWishlistButton();
	} else {
		showAddToWishlistButton();
	}
});
```

### Bonus
If you added the popup before then here's the code to show it when the user adds an item to the Wishlist:
```javascript
async function onWishlistClicked() {
	// if user is not logged in, prompt him to login
	if (!wixUsers.currentUser.loggedIn) {
		await wixUsers.promptLogin();
		return;
	}
	
	// get the current product in the product page
	const product = await $w('#productPage1').getProduct();
	// if item is already in list, the click should remove it and show the add button
	if (await isProductInWishlist()) {
		await removeWishlistItem(product._id);
		showAddToWishlistButton();
	} else {
		// otherwise add the product to the wishlist and show the remove button
		await insertWishlistItem(product);
		
		$w('#insertWishlistNotif').show('fold');
		setTimeout(function () {
			$w('#insertWishlistNotif').hide('fade');
		}, 3000);
		
		showRemoveFromWishlistButton();
	}
}
```
The `show` and `hide` functions do... exactly what you expect, they show and hide the element and the extra parameter is the animation used to show and hide the element (the full list can be found in the documentation). We also added some code to hide it after 3 seconds so it won't just hang there...
> Note: Make sure you are using the correct Id for the popup.


### Time to build the Wishlist!
So far we only created the functionality to add and remove items from the list but we still can't see it.
For that we need to create a new page that will show the Wishlist itself.
So, add a new page named "Wishlist" and inside it add a new item from the `Lists & Grids` section. That will be our `repeater` and we'll use that to show the Wishlist.

Update the Ids in the repeater so we'll be able to use them later on.
Here's how my repeater looks like with the Ids I chose (writtend in red and `wishlistRepeater` as the Id of the repeater):
![alt repeater ids](images/repeaterIds.png)

First we need a way to get the items, we can do that by adding another function to our `backend` file.
```javascript
export async function getWishlistItems() {
	const user = wixUsers.currentUser;
	
	// if this is not a member don't return any items
	if (!user.loggedIn) {
		return [];
	}
	return await wixData.query(collectionName)
		.eq('userId', user.id)
		// this will add the referenced product to the result
		.include('product')
		.find();
}
```

Next, let's add a `loadWishlist` function on page load.
```javascript
// import backend functions
import { getWishlistItems, removeWishlistItem } from 'backend/wishlist';
// import the wix-location module for navigating to pages. we'll use that in a moment
import wixLocation from 'wix-location';

$w.onReady(function () {
	// hide repeater in case it has any previous unwanted items
	$w('#wishlistRepeater').hide();
	loadWishlist();
});

async function loadWishlist() {
	const data = await getWishlistItems();

	const repeater = $w('#wishlistRepeater');
	
	// assign items to repeater
	repeater.data = data.items;
	
	// set the handler that binds data for each item
	repeater.onItemReady(onWishlistItemReady);
	repeater.show();
}
```
As you can see we get the items from the `backend` and set them as the data for the grid, but we're still missing the function that binds the data to each row item. We'll do that right now.
```javascript
function onWishlistItemReady($item, wishlistItem) {
	// the referenced product is in the wishlist item since we included it in query
	const product = wishlistItem.product;
	
	// set the image src to the product's image
	$item('#wishlistImage').src = product.mainMedia;
	
	// let's add a click event to navigate to the product page when clicking on image
	$item('#wishlistImage').onClick(() => {
		// this is using wixLocation module to Navigate to the item's product page
		wixLocation.to(product.productPageUrl);
	});
	$item('#wishlistProductName').text = product.name;
	$item('#wishlistDescription').text = product.description;
	
	// here we're converting the 'added date' to a readable format
	$item('#wishlistAddedDate').text = wishlistItem.addedDate.toLocaleString();
	
	// if item is not in stock we shouldn't enable the button (assuming it's disabled by default)
	if (product.inStock) {
		$item('#wishlistAddToCart').enable();
		
		// for products that have variants we need to send the user to product page to select one
		// otherwise we can immediately add item to cart		
		const hasVariants = !isObjectEmpty(product.productOptions)
		
		if (hasVariants) {
			// set the button text to explain action
			$item('#wishlistAddToCart').label = 'Select variants';
			
			// set the navigation to the product page
			$item('#wishlistAddToCart').onClick(() => {
				// Navigate to the wishlist item's product page.
				wixLocation.to(product.productPageUrl);
			});
		} else {
			// otherwise we can add it to cart
			$item('#wishlistAddToCart').label = 'Add to cart';
			$item('#wishlistAddToCart').onClick(async () => {
				// in order to add to cart we have to use the cart icon
				// in case you don't want to show one in this page you can add an icon and hide it
				await $item('#shoppingCartIcon1').addToCart(product._id);
			});
		}
	}
}

// checks if object has any keys
function isObjectEmpty(obj) {
	return Object.entries(obj).length === 0 && obj.constructor === Object;
}
```
The `$item` is the element in the list that corresponds to the given `wishlistItem`, so we can use that to select any  previously defined sub-elements by the Ids we gave them and bind the data as we see fit.

Now we're just missing the option to remove an item from the wishlist. let's implement that:
```javascript
function onWishlistItemReady($item, wishlistItem) {
	// same implementation as before
	//...
	
	// here we're converting the 'added date' to a readable format
	$item('#wishlistAddedDate').text = wishlistItem.addedDate.toLocaleString();
	
	// ------------ ADDED THESE LINES -------------------
	$item('#wishlistRemoveButton').onClick(async () => {
		await removeItemFromWishlist(product._id);
	});
	// ------------ END -------------------
	
	// if item is not in stock we shouldn't enable the button (assuming it's disabled by default)
	if (product.inStock) {
		// nothing changed here
		// ...
		} else {
			// otherwise we can add it to cart
			$item('#wishlistAddToCart').label = 'Add to cart';
			$item('#wishlistAddToCart').onClick(async () => {
				// in order to add to cart we have to use the cart icon
				// in case you don't want to show one in this page you can add an icon and hide it
				await $item('#shoppingCartIcon1').addToCart(product._id);
				// ------------ ADDED THESE LINES -------------------
				await removeItemFromWishlist(product._id);
				// ------------ END -------------------
			});
		}
	}
}

async function removeItemFromWishlist(productId) {
	// use the same backend call from before to remove the item
	await removeWishlistItem(productId);
	
	// load the list again after the item was removed
	await loadWishlist();
}
```

# Congragulations!
![alt repeater ids](https://media.giphy.com/media/bKBM7H63PIykM/giphy.gif)

If you got to here then you have a fully functional wishlist in your website!
Users should now be able to add and remove items from their Wishlist and as the site owner you can see the products in the collection and manage it as you see fit.

If you want to see how you can take this further check out the next step.

# Something Extra
So we have a fully functioning Wishlist, but I bet that with a little more effort we can improve it by a lot!

First things first, let's add some indications to a non-member user when he navigates to the Wishlist.
![alt repeater ids](images/wishlistUserNotMember.png)
> Note that we're hiding it by default. we'll show it only if the user is not a member.
To do that we can add a condition for user membership in the `loadWishlist` function:
```javascript
async function loadWishlist() {
	// ------------ ADDED THESE LINES -------------------
	// check if user is a member
	const isLoggedIn = await isUserLoggedIn();
	if (!isLoggedIn) {
		$w('#wishlistNotLoggedIn').show();
		return;
	}
	// ------------ END -------------------
	// otherwise display the list just like before
	const data = await getWishlistItems();

	const repeater = $w('#wishlistRepeater');
	
	// assign items to repeater
	repeater.data = data.items;
	
	// set the handler that binds data for each item
	repeater.onItemReady(onWishlistItemReady);
	repeater.show();
}
```
That's already a nice edition to our website instead of showing some empty page.

How about the case when the user doesn't have anything in the Wishlist? let's add a text for that too!
![alt repeater ids](images/wishlistNoItems.png)
As you can see in the image I've added another text in the same location as the member's text and i'm using the layers of the editor to select it and modify it (you can also use the layers functionality to show and hide other items so you can focus only on the parts that you need right now).

And just like before let's add the code that shows/hides the text when we want:
```javascript
async function loadWishlist() {
	// check if user is a member
	const isLoggedIn = await isUserLoggedIn();
	if (!isLoggedIn) {
		$w('#wishlistNotLoggedIn').show();
		return;
	}
	
	// otherwise display the list just like before
	const data = await getWishlistItems();

	const repeater = $w('#wishlistRepeater');
	// ------------ ADDED THESE LINES -------------------
	if (data.items.length === 0) {
		repeater.hide();
		$w('#wishlistNoItemsText').show();
		return;
	}
	// ------------ END -------------------
	
	// assign items to repeater
	repeater.data = data.items;
	
	// set the handler that binds data for each item
	repeater.onItemReady(onWishlistItemReady);
	repeater.show();
}
```
And now the user won't see a blank page when he has nothing in his Wishlist.

So we tweaked some thing to make our site more attractive to users but we haven't seen something new yet and this is the **EXTRA** part after all...

And that's why I want to show you how to create a custom page in your dashboard to show you which products your customers want the most! this could give you a real advantage is managing your online store.

Let's get right to it, in the editor go to the "add a new page" section and select `Dashboard Page`:
![alt repeater ids](images/addDashboardPage.png)

I named mine `Most Wished for products` so I could easily find it in the dashboard section.
You can navigate to the page through the pages menu:
![alt repeater ids](images/navigateToDashboard.png)

What we're going to do is add another repeater like before, only this time we're going to do a couple of new things:
1. Create an aggregation query to count how many users added a product them to their wishlists.
2. Bind the repeater to a dataset and use filter to show only the products that we want.

We can drop in a repeater from the `lists & grids` section and set up Ids just like we did in the Wishlist page and now instead of manually binding the columns in the data to the UI we're going to bind that data from a `dataset` object.
First we need to create a `dataset` object:
![alt repeater ids](images/createDataset.png)

Select the `Products` collection and click `create`. As a result a new `dataset` object should appear on your screen:
![alt repeater ids](images/datasetObject.png)

This object will not be displayed in your website, it's only shown in the editor so you could manage it, so you can move it anywhere you want on the screen without affecting your website.
Now that we have a `dataset` we can bind it's columns to elements in the UI. Let's bind the name of the product to a text element. Select the text element you want to connect and click on the "connect to data" button, then select the "Stores/Products dataset" in the as the dataset to connect and in the "Text connects to" select the `Name` field.
![alt repeater ids](images/connectText.png)

Now if you try to preview the your website you should see that the repeater shows the products in your store with their corresponding names. You can connect the rest of the items in the repeater just like the text so they'll be displayed in the list.

Meanwhile, i'll skip ahead to show you how to also display the amount of items (that's what we're here for after all!).
We're going to need a query to get the data that we want. We already know by now that the right place to put queries is in the `backend` section and we can add a function in our already existing file:
```javascript
// previously defined imports
// ...

// previously defined functions
// ...

export async function getMostWishedForItems() {
	return wixData.aggregate(collectionName)
		// we want to group the entries by the product column
		.group('product')
		// and count them
		.count()
		.run();
}
```

`wixData` exposes an aggregate API to group items by the fields that we want in a collection. In this example we want to count the amount of products in all Wishlists so we group by the `product` column and simply count the occurences.

Moving back to our dashboard page, let's add the code that handles the binding to the "count" of the products:
```javascript
import { getMostWishedForItems } from 'backend/wishlist';


let data;

$w.onReady(function () {
	loadWishlist();
});

function onWishlistItemReady($w, product) {
	const matchingItem = data._items.find(item => item._id === product._id);
	const itemCount = matchingItem ? matchingItem.count.toString() : '0';
	$w('#itemRequestedAmount').text = itemCount;
}

async function loadWishlist() {
	data = await getMostWishedForItems();
	const repeater = $w('#mostWishedForRepeater');
	repeater.onItemReady(onWishlistItemReady);
}
```

Let's analyze this code. first we're importing the `getMostWishedForItems` from the backend so we'll be able to retrieve the "count" of the products. Next we're defining a `data` variable where we're going to store the result of that query so we'll be able to use it in `onWishlistItemReady` (we can't pass it directly, since the `onWishlistItemReady` is called for us when the item is ready and we can't pass the result of `getMostWishedForItems` to it).
Just like before we're adding a handler function for `onItemReady`, since this method is called whenever the repeater needs to render an item it will also be called when the data from the `dataset` needs to be rendered.

In the `onWishlistItemReady` we're searching the aggregated data for the product with the same id as the currently rendered item to get it's count amount. If we find an item then we show it's count value and in case we don't we'll just show 0.
> Note: The aggregated result doesn't return products with `0` count, since it only aggregates on products that exist in the Wishlist table and `0` count means that no product was found in the Wishlist table.

At this point we already have a working dashboard page that shows the count of products, but i'd like to do one more thing and that is to filter out the items that don't appear in any Wishlist (0 count items).

To do that we need to add a `filter` to our `dataset`:
```javascript
// add wixData import to create filters
import wixData from 'wix-data';

async function loadWishlist() {
	data = await getMostWishedForItems();
	
	// ------------ ADDED THESE LINES -------------------
	// get the product ids from the aggregated result
	const productIds = data._items.map(item => item._id);
	
	// add a filter to the dataset to filter items by the given product ids
	$w('#dataset1').setFilter(wixData.filter().hasSome('_id', productIds));
	// ------------ END -------------------
		
	const repeater = $w('#mostWishedForRepeater');
	repeater.onItemReady(onWishlistItemReady);
}
```
What we did here is add a filter to the `dataset` object that shows only products with `_id` that is present the array of ids (`productIds`) that we supplied it. The result is a Wishlist that displays only products that have a `count > 0`, just like we wanted!
> Note: At this point we can replace this line of code: `const itemCount = matchingItem ? matchingItem.count.toString() : '0';` with this: `const itemCount = matchingItem.count.toString();` since there will always be a match (we made sure of that in the filter), but we can also just keep this as it is.
> Note: The `count` property is a `number` and we're converting it to a string, since `Text` elements must recieve a string in their `text` property.

That's it! hope you liked it and learned something new along the way :)

![alt repeater ids](https://media.giphy.com/media/V2ZrZfHghzSNi/giphy.gif)


## Additional info
* You can find a working example site [here](https://oripi3.wixsite.com/wishlisttest/wishlist).
* A link to corvid api can be found [here](https://www.wix.com/corvid/reference).
* Lodash documentation can be found [here](https://lodash.com/docs)

