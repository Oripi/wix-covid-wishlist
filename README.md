# wix-covid-wishlist
Getting started with Wix Corvid

# What does this walkthrough offer?
In this walkthrough we're going to create our very own Wishlist for a wix website!

we'll learn how to:
* Interact with page elements through Corvid.
* Use some of the various libraries that Corvid exposes us to manipulate our website.
* Create our own database table and use CRUD (Create, Read, Update, Delete) operations on it.

# so let's get started!
the first thing we need is... a website! so let's start by creating a new website where we can try our code. since we're creating a Wishlist for an online store, select the "online-store" option when creating a new website. in case you created a different kind of website you can always enable it through the same menu that you add different elements to your website, just under the `store` section.

now, let's enable Corvid so we'll be able to modify what we need:
![alt enable corvid](images/enableCorvid.png)

next, let's go to "Product Page" and create an `Add to wishlist` button.
![alt wishlist button](images/addToWishlistButton.png)

### Bonus:
if you wanna be really fancy you can also add a popup that will show once an item has been added to wishlist:
![alt wishlist popup](images/addToWishlistPopup.png)
we'll manipulate it later to show and hide with animations.
make sure to hide it on page load (can be done through the properties window).


we're going to need a database to store our Wishlist data, so let's create one.
in the sidebar under `Database` click the `Add a new collection` link and select the `start from scratch` option.
name the collection `Wishlist` and in permissions select `Member-generated content` so only members will be able to add and remove items from Wishlists.
> after creating the database save the site and refresh the page in order to see native collections, such as `Products` table.

now we're going to need to add columns to the table to store the `UserId, Product, AddedDate` and any other column that you want your users to fill that can be used later on.
> Do not modify the id of the column, since we'll use it later to query and display the data. for example for `UserId` the Id of the column should be `userId`.

when creating the columns create them with the following types:
* UserId - Text
* Product - reference column that is referenced to the products table
* AddedDate - Date and Time
in case you made a mistake in the column creation you can edit it via the settings of the column. You may get a warning message, but that's ok because we haven't used the table yet.

Finally it's time to write some code!

open up the dev console by clicking on it in the lower part of the page:
![alt wishlist popup](images/devConsole.png)
you should see the default code:
```javascript
// For full API documentation, including code examples, visit http://wix.to/94BuAAs

$w.onReady(function () {
	//TODO: write your page related code here...

});
```

this function is called when the page is ready and elements are loaded into the page. If we try to access elements outside this function before the page load they won't work, so it's important that we do it here.

let's modify it by adding an `onClick` event for the button.
```javascript
$w.onReady(function () {
	$w('#AddToWishlistButton').onClick(onWishlistClicked);
});
```
you should see an error in the console, that's because we also need to change the Id of the button so we'll be able to find it and we also need to create a handler for the click.
in the editor right click on the button and select `View properties`. in the window that pops up rename the Id to `AddToWishlistButton`.
![alt wishlist popup](images/changeId.png)

and let's also create a function that will handle the click event below the `$w.onReady(...)` call.
```javascript
function onWishlistClicked() {
  console.log('hello world');
}
```
now if you preview the website you should be able to see the `"hello world"` in the console.

In order to insert a new item in the database we need to expose a method from the `backend` that will `client` will invoke and as a result a new item will be added to the Wishlist collection.
In the sidebar, click the `+` when hovering over `backend` or expand it and click on `Add a new web module`. name the module `wishlist.jsw` and open it.

inside it add the following code:
```javascript
// Import the wix-users module for working with users.
import wixUsers from 'wix-users-backend';
// Import the wix-data module for working with queries.
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
we've done a couple of things here. first we imported 2 modules: `wix-users-backend, wix-data`, the first is used to handle user status and in our case to check if the current user is a member or not (since only members should be able to update the table). the second is to manipulate collection data and in the above code, to add a new entry to the collection.
> Note that although product is a reference column and you may see the name of the product in the collection view page it is actually referenced by an id, in this case the `_id` field of the product. this is the same case when we are querying the data, as you'll see soon.

Now we need to call this code from the client, so let's do just that.
back in the product page let's replace the `onClick` handler with the following code:
```javascript
async function onWishlistClicked() {
	// get the current product in the product page
	const product = await $w('#productPage1').getProduct();
	// call the backend function to add the item
	await insertWishlistItem(product);
}
```
and add the following import at the top:
```javascript
import { insertWishlistItem } from 'backend/wishlist.jsw';
```
> Notes: 
> * make sure that the product component Id is `productPage1` or change it to yours.
now when we click on the button a new item should be added to the collection. you can verify this by previewing the page, clicking the button and then go back to the collection and see if it's added.
> * we can also insert to a collection directly from the client side without using a backend code, however this means exposing the userId in the client side and don't want to do that...

But what about the case when our user isn't a member and he clicks the Wishlist button?
well we can prompt him to sign up instead! let's add the code that does that:
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
and add the following import at the top:
```javascript
import wixUsers from 'wix-users';
```

We don't want the user to insert the same product multiple times to the Wishlist and we haven't added an option to remove it. we can solve both issues at the same time by coverting the `Add to Wishlist` button to a toggle button that will add or remove an item as necessary.

In order to do that we need 2 new functions in the `backend` that will handle these calls.
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
and now in the client we need to add the logic for the toggle button.
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
and add these functions at the bottom:
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
and these imports at the top:
```javascript
import { insertWishlistItem, getItemInWishlist, removeWishlistItem } from 'backend/wishlist.jsw';
```
we also need to update the `$w.onReady(...)` to reflect the initial state:
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
the `show` and `hide` functions do... exactly what you expect, they show and hide the element and the extra parameter is the animation used to show and hide the element (the full list can be found in the documentation). we also added some code to hide it after 3 seconds so it won't just hang there...
> Note: make sure you use the correct Id for the popup.


### Time to build the Wishlist!
so far we only created the functionality to add and remove items from the list but we still can't see it.
For that we need to create a new page that will show the Wishlist itself.
So, add a new page named "Wishlist" and inside it add a new item from the `Lists & Grids` section. that will be our `repeater` and we'll use that to show the Wishlist.

update the Ids in the repeater so we'll be able to use them later on.
here's how my repeater looks like with the Ids I chose (writtend in red and `wishlistRepeater` as the Id of the repeater):
![alt repeater ids](images/repeaterIds.png)

first we need a way to get the items. We can do that by adding another function to our `backend` file.
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

next, let's add a `loadWishlist` function on page load.
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
as you can see we get the items from the `backend` and set them as the data for the grid. but we're still missing the function that binds the data to each row item. we'll do that right now.
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
the `$item` is the element in the list that corresponds to the given `wishlistItem`, so we can use that to select any  previously defined sub-elements by the Ids we gave them and bind the data as we see fit.

now we're just missing the option to remove an item from the wishlist. let's implement that:
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
if you got to here then you have a fully functional wishlist in your website!



## additional info
you can find a working example site [here](https://oripi3.wixsite.com/wishlisttest/wishlist).

a link to corvid api can be found [here](https://www.wix.com/corvid/reference).
