## Scrape the MRT from Wikipedia

https://en.wikipedia.org/wiki/List_of_Singapore_MRT_stations

```js
const elements = [...$('#test tbody tr')]
elements.map((element) => {
	return element.innerText.split('\t')
}).filter((element) => {
	return element.length > 1 && element[0].trim().length > 1
}).map(element => [element[0].trim(), element[1].trim()])
```
