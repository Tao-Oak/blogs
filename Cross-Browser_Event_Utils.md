```javascript
// How to add event handler
element.addEventListener	// DOM 2, element EC
element.attachEvent			// IE, global EC
element['on' + type]		// DOM 0, element EC
HTML inline					// global EC

// DOM Event Object
1. bubble, eventPhase, stopPropagation, stopImmediatePropagation
2. target, currentTarget
3. cancelable, defaultPrevented, preventDefault
4. type, isTrusted/trusted, view, detail

// IE Event Object
srcElement				// target
cancelBubble			// stopPropagation
returnValue				// preventDefault

// IE get event object
function handler (event) {		// DOM 0, element['on' + type]
}
function handler () {			// DOM 2, attachEvent
	var event = window.event
}

// Cross-browser Event Utils
const EventUtil = {
  addHandler: function (element, type, handler) {
    if (element.addEventListener) {
      element.addEventListener(type, handler, false)
    } else if (element.attachEvent) {
      element.attachEvent('on' + type, handler)
    } else {
      element['on' + type] = handler
    }
  },
  removeHandler: function (element, type, handler) {
    if (element.removeEventListener) {
      element.removeEventListener(type, handler, false)
    } else if (element.detachEvent) {
      element.detachEvent('on' + type, handler)
    } else {
      element['on' + type] = null
    }
  },
  getEvent: function (event) {
    return event ? event : window.event
  },
  getTarget: function (event) {
    return event.target || event.srcElement
  },
  preventDefault: function (event) {
    if (event.preventDefault) {
      event.preventDefault()
    } else if (event.returnValue) {
      event.returnValue = false
    }
  },
  stopPropagation: function (event) {
    if (event.stopPropagation) {
      event.stopPropagation()
    } else if (event.canceBubble) {
      event.canceBubble = true
    }
  }
}
```