
# Notification Code

## JavaScript
```js
// Create a container for notifications if it doesn't exist
if (!document.querySelector('.notification-container')) {
    const notificationContainer = document.createElement('div');
    notificationContainer.className = 'notification-container';
    document.body.appendChild(notificationContainer);
}

function showNotification(message, type) {
    const notification = document.createElement('div');
    notification.className = `notification-popup notification-${type}`;
    notification.textContent = message;

    // Add notification to the container
    const container = document.querySelector('.notification-container');
    container.appendChild(notification);
    notification.style.display = 'block';

    // Set timeout to start the slideout animation
    setTimeout(() => {
        notification.classList.add('slideout');
    }, 3000); // Display time for the notification

    // Remove notification after slideout animation ends
    notification.addEventListener('animationend', () => {
        if (notification.classList.contains('slideout')) {
            container.removeChild(notification);
        }
    });
}
```

## CSS
```css
.notification-container {
    position: fixed;
    bottom: 20px;
    right: 20px;
    z-index: 1000;
    display: flex;
    flex-direction: column;
    align-items: flex-end;
    gap: 10px;
}

.notification-popup {
    background-color: rgba(0, 0, 0, 0.7);
    color: white;
    padding: 15px;
    border-radius: 5px;
    min-width: 200px;
    text-align: center;
    animation: slidein 0.5s;
}

.notification-success {
    background-color: #4CAF50;
}

.notification-error {
    background-color: #f44336;
}

.notification-copied {
    background-color: #2196F3;
}

@keyframes slidein {
    from {transform: translateX(100%); opacity: 0;}
    to {transform: translateX(0); opacity: 1;}
}

@keyframes slideout {
    from {transform: translateX(0); opacity: 1;}
    to {transform: translateX(100%); opacity: 0;}
}

.slideout {
    animation: slideout 0.5s forwards;
}
```
