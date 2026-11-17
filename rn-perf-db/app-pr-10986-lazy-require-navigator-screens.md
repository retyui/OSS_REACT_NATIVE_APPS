# Optimise navigators so that they import screens only when needed.
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/10986>
- **Merged Date:** 2022-09-28
- **Category:** Startup

## Problem
`AuthScreens` and `ModalStackNavigators` imported about 40 page components (settings, IOU, workspace, payments and others) at the top of the file. A top-level `import` makes Metro evaluate the module and its dependencies at launch, so everything was initialized up front even though the first screen needs only a few. The PR notes that some libraries do heavy work when first required (for example Reanimated memoizing animations and sending them over the bridge), and that even without such work every file must still be evaluated. The PR reports no before/after numbers.

## Solution
- Replace the static imports with `getComponent={() => require('...').default}` on each `Screen`.
- Metro's `require` evaluates a module the first time it is accessed and then returns the cached exports, so a screen's file and its dependencies are loaded only when React Navigation first needs that screen.
- `createModalStackNavigator` now passes `getComponent` through to `Screen` instead of `component`.
- Applied to the main drawer navigator, the login/validate pages and all modal stack screens.

## Code Pattern (Before vs. After)
```js
// Before
import MainDrawerNavigator from './MainDrawerNavigator';
import ValidateLoginPage from '../../../pages/ValidateLoginPage';

<RootStack.Screen
    name="ValidateLogin"
    options={{headerShown: false, title: 'New Expensify'}}
    component={ValidateLoginPage}
/>

// After
<RootStack.Screen
    name="ValidateLogin"
    options={{headerShown: false, title: 'New Expensify'}}
    getComponent={() => {
        const ValidateLoginPage = require('../../../pages/ValidateLoginPage').default;
        return ValidateLoginPage;
    }}
/>
```
```js
// ModalStackNavigators.js, Before
import IOUBillPage from '../../../pages/iou/IOUBillPage';
const IOUBillStackNavigator = createModalStackNavigator([{
    Component: IOUBillPage,
    name: 'IOU_Bill_Root',
}]);

// After: "We use getComponent/require syntax so that file used by screens are not loaded until we need them."
const IOUBillStackNavigator = createModalStackNavigator([{
    getComponent: () => {
        const IOUBillPage = require('../../../pages/iou/IOUBillPage').default;
        return IOUBillPage;
    },
    name: 'IOU_Bill_Root',
}]);
```

## Takeaway
Use `getComponent` with an inline `require` (or inline requires in Metro config) for navigator screens so rarely visited screens do not add to startup work.
