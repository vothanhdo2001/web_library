
#Copy when click table

##Javascript
```js
document.querySelector('.data-table').addEventListener('click', function(event) {
    var element = event.target;
    var valueToCopy = '';

    if (element.tagName === 'TD' || element.tagName === 'TH') {
        valueToCopy = element.textContent;
    }

    if (valueToCopy) {
        copyToClipboard(valueToCopy);
        showNotification('Copied: ' + valueToCopy, 'copied');
    }
});
```
