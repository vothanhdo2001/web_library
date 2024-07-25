
# Copy when click table

## Javascript
```js
function copyToClipboard(value) {
    var tempInput = document.createElement('input');
    tempInput.style.position = 'absolute';
    tempInput.style.left = '-9999px';
    tempInput.value = value;
    document.body.appendChild(tempInput);
    tempInput.select();
    document.execCommand('copy');
    document.body.removeChild(tempInput);
}
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
