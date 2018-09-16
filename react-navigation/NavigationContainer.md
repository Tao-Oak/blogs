<font face="Times New Roman">

```javascript
// Statefull navigation container 
<NavigationContainer
	// Pass down extra options to child screens
	// this props will get passed to the screen component as this.props.screenProps
	screenProps={{ a, 'a', b: 'b' }}
	
	// Screen tracking
	onNavigationStateChange={ (prevState, currentState) => {} }
	
	// State persistence
	persistenceKey
	renderLoadingExperimental
	
	// Deep linking
	enableURLHandling
	uriPrefix
	
	// set true to disable explicitly-rendering-more-than-one-navigator wranning
	detached
/>

// Sub-navigation container
<NavigationContainer
	screenProps={{ a, 'a', b: 'b' }}
	navigation={this.props.navigation}
/>
```


```javascript
function isStateful(props) {
  return !props.navigation;
}

function validateProps(props) {
  if (isStateful(props)) {
    return;
  }

  const { navigation, screenProps, ...containerProps } = props;

  const keys = Object.keys(containerProps);

  if (keys.length !== 0) {
    throw new Error(
      'This navigator has both navigation and container props, so it is ' +
        `unclear if it should own its own state. Remove props: "${keys.join(
          ', '
        )}" ` +
        'if the navigator should get its state from the navigation prop. If the ' +
        'navigator should maintain its own state, do not pass a navigation prop.'
    );
  }
}
```

Hello world
</font>