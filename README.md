 stackline full stack assessment

 Introduction

This project is a small ecommerce application built using next.js. the application contains three main parts: a product listing page, filtering and search functionality, and a product detail page. the project also includes several api routes that return product data, categories, and subcategories.

my goal while working on this assignment was to carefully review the codebase like I would during a real production code review. instead of only looking for obvious errors, I focused on identifying areas where the code could break in real-world usage. this includes things like unsafe data handling, missing validation, lack of error handling, and situations where the user interface might crash if the data is slightly different than expected.

while going through the project, I examined both the frontend pages (react components) and the backend api routes. whenever I found a problem, I tried to fix it in a way that improves reliability without changing the overall architecture of the application.

this document explains the issues I found, how I fixed them, and the reasoning behind each fix.



 running the project

to run the project locally:

1. install dependencies

npm install

2. start the development server

npm run dev

3. open the application in a browser

http://localhost:3000/

the homepage displays the product list. from there you can search products, filter by category and subcategory, and open a product detail page.

---

issues found and fixes

 issue 1 – unsafe parsing of query parameters in the products api

in the api route located in app/api/products/route.ts, the code reads query parameters such as limit and offset from the request url. these parameters were parsed using parseInt directly:

limit: searchParams.get('limit') ? parseInt(searchParams.get('limit')!) : 20

the problem here is that parseInt can return NaN if the value cannot be converted to a number. for example:

/api/products?limit=abc

in this situation parseInt("abc") returns NaN, which could break pagination logic inside productService.

to fix this, I added safe validation around the parsing logic and provided fallback values when the input is invalid.

I also added a maximum limit to prevent extremely large requests.

example fix:

const parsedLimit = limitParam ? parseInt(limitParam, 10) : 20
const limit = isNaN(parsedLimit) ? 20 : Math.min(parsedLimit, 50)

this ensures the api always receives a valid number and prevents excessive requests.

---

 issue 2 – missing error handling in the products api

the original api implementation assumed that everything inside productService would always succeed. however in real systems any data operation can fail.

for example:

* unexpected data format
* runtime errors
* invalid parameters

if such an error occurred, the api would crash and return an unhandled error.

to prevent this, I wrapped the entire handler inside a try/catch block.

try {
...api logic
} catch (error) {
return NextResponse.json({ error: "Failed to fetch products" }, { status: 500 })
}

this ensures the api always returns a valid response even if something goes wrong internally.

---

 issue 3 – search input not sanitized

the search query parameter from the url was used directly in the filtering logic.

search: searchParams.get('search')

however users sometimes add accidental spaces when typing in a search box. for example:

"   laptop   "

these spaces can cause search filtering to behave inconsistently.

to solve this, I used the trim() method before using the value.

search: searchParams.get('search')?.trim()

this removes leading and trailing spaces and keeps the search behavior consistent.

---

issue 4 – subcategories not filtered by category

on the product listing page (app/page.tsx), when a category is selected the application fetches subcategories from the api.

however the original request looked like this:

fetch("/api/subcategories")

this means the api returned every subcategory, regardless of which category was selected.

this leads to an incorrect user experience because unrelated subcategories appear in the dropdown.

I fixed this by passing the selected category as a query parameter.

fetch(`/api/subcategories?category=${selectedCategory}`)

this allows the api to return only the relevant subcategories for the selected category.

---

 issue 5 – missing error handling in frontend fetch requests

several frontend fetch calls assumed that the request would always succeed.

examples include:

fetch("/api/categories")
fetch("/api/subcategories")
fetch("/api/products")

if any of these requests fail due to network issues or server errors, the application would fail silently and debugging becomes difficult.

I added catch blocks to log errors.

.catch(err => console.error("Failed to fetch categories:", err))

this improves visibility when debugging and prevents silent failures.

---

 issue 6 – unsafe image rendering in the product list

on the product list page, the code attempted to render the first product image like this:

product.imageUrls[0]

if a product has no images, the array may be empty and the UI could render a broken image container.

I added a fallback placeholder image.

src={product.imageUrls?.[0] || "/placeholder.png"}

this ensures the UI remains stable even if image data is missing.

---

issue 7 – unsafe access to featureBullets

on the product detail page, the code checked the length of featureBullets directly:

product.featureBullets.length

if featureBullets is undefined, this would cause a runtime error.

to fix this I used optional chaining.

product.featureBullets?.length

this ensures the code only runs when the array exists.

---

 issue 8 – JSON parsing safety on the product page

the product page reads the product object from the url query and parses it using JSON.parse.

JSON.parse(productParam)

if the query parameter is invalid or manually modified, JSON.parse will throw an exception.

I wrapped the parsing logic inside a try/catch block.

try {
const parsedProduct = JSON.parse(productParam)
} catch (error) {
console.error("Failed to parse product data")
}

this prevents the entire page from crashing.

---

issue 9 – subcategory reset when category changes

when switching between categories, the previously selected subcategory could remain active.

this leads to an inconsistent filter state.

I fixed this by resetting the subcategory whenever the category changes.

setSelectedCategory(value || undefined)
setSelectedSubCategory(undefined)

this ensures the filters remain logically consistent.

---

 issue 10 – loading state reliability

during product fetching, loading state was set to true before the request, but if the request failed the loading state might not update correctly.

I adjusted the fetch logic to ensure loading is updated after the request finishes.

this guarantees that the UI always exits the loading state even if an error occurs.

---

summary

while reviewing this project, my main focus was improving robustness and preventing runtime failures. many of the fixes involve defensive programming techniques such as input validation, optional chaining, and proper error handling.

the improvements introduced include:

* safer api parameter parsing
* limiting api response sizes
* proper error handling in api routes
* better error handling for frontend requests
* protection against undefined values
* improved filtering behavior
* more stable UI rendering when data is missing

these changes make the application safer, more predictable, and easier to maintain.

---

 time spent

approximately two hours were spent reviewing the project, identifying issues, implementing fixes, and validating the behavior of the application. the main goal was to improve reliability while keeping the original design and structure of the project intact.
