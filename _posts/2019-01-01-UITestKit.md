---
layout: post
title: UITestKit
---

[UITestKit](https://github.com/intere/UITestKit) is a better UI Test Framework for iOS.

A framework that lets you write UI Tests, directly from an XCTest target.

### Why?
`XCUITest's work just fine, why not just use them?`

👆🏼 That's the problem, they don't work `just fine`.  

1. The code that is generated when you record is overly verbose
2. You have to jump through hoops (and wire in mocking logic) into your codebase to mock a service
3. You don't have any direct access to the code under test
4. System dialogs are difficult if not impossible to interact with

### So you built your own framework?
Yes, but it's very lightweight.  Did I mention that it runs from an XCTest container?

### What are the benefits of UITestKit?
- Automatically generate screenshots when a test assertion fails
- Disable animations to help prevent non-deterministic UI Test failures
- Leverage the full power of an XCTest harness for
    - Driving your UI, programmatically
    - Mocking your data sources
    - Simulating failures
    - Keeping test code out of your production codebase


## Example
Here's a screenshot of a test suite running:

![Example Tests](https://user-images.githubusercontent.com/2284832/50541549-86a3e080-0b65-11e9-95d2-176b9ce3164b.gif)

Let's talk through some of the code (excerpts from [Reference Code - ShapeTableTests.swift](https://github.com/intere/UITestKit/blob/develop/Example/UITestKit_ExampleTests/ShapeTableTests.swift)):

```swift
31. openShapeTab()
32. XCTAssertTrue(waitForCondition({ self.shapeTableVC != nil }, timeout: 1), topVCScreenshot)
```

Line 31 `openShapeTab()` is a call to a function defined in the base test class, it merely opens up the tab that we're about to start testing.
Line 32 `XCTAssertTrue(...)` is a lot of code on one line.  Let's break it into pieces:
- `{ self.shapeTableVC != nil }` This is a block that returns a Boolean value telling us if the `ShapeTableViewController` is currently at the "top" of the view hierarchy.
- `waitForCondition(_:timeout)` is a call that does the following:
    1. Free up the UI Thread for a fraction of a second
    2. Check to see if the condition (the block) is true or false, if it's true, exit and return true
    3. If the condition is false, then check to see if the timeout has elapsed.  If it has, then return false
    4. If the timeout hasn't elapsed, then take another pass through the loop
- `topVCScreenshot` grabs a screenshot of the rootViewController, then grabs the name of the "Top" View Controller and then writes a file to `/tmp` that uses the name of the top view controller and the current time (in elapsed milliseconds since the unix epoch)
    - `NOTE:` if there is a visible `SFSafariViewController` in the view hierarchy, it will show as a blank view (this is a security feature provided by iOS)

Now if we reassemble everything that's happening in that single `XCTAssertTrue` line, it reads something like this:
- Poll for at most, 1 second and Assert that the `ShapeTableViewController` is the top view or take a screenshot and fail with an error message that is the filename to that screenshot

<br /><br />
```swift
33. guard let shapeTableVC = shapeTableVC else {
34.     return XCTFail("No ShapeTableVC")
35. }
36. pauseForUIDebug()
```

In the upcoming lines of code, we're going to need a reference to the `ShapeTableViewController`, so we setup a guard at line 33.  If we can't get this ViewController for some reason, we will drop into line 34 which is using a bit of a trick to combine two lines of code into a single line.  This test function does not return anything and `XCTFail` also does not return anything.  Therefore this is a legal call that the compiler does not complain about and it saves some vertical space.  This code block essentially reads like this:
- Get a reference to the ShapeTableViewController or fail and exit the test

Line 36 is a call that will optionally pause for a specific amount of time (this is particularly useful if you're debugging a UI Test and want to watch what's going on in the UI).  This pause is a no-op by default, but becomes a pause (defaults to 0.5 seconds) if you enable the `shouldPauseUI` flag.

<br /><br />
```swift
36. XCTAssertEqual(0, shapeTableVC.tableView.numberOfRows(inSection: 0), "Wrong number of rows")
37.
38. guard let rightButton = shapeTableVC.navigationItem.rightBarButtonItem else {
39.     return XCTFail("No right button or action")
40. }
```

- Line 36: Make sure there are 0 rows in the `ShapeTableViewController`.
- Line 38: Get a reference to the (optional) `rightBarButtonItem` or fail and exit.

<br /><br />
```swift
44. for index in 0..<Shape.allCases.count {
45.     shapeTableVC.addShape(rightButton)
46.     XCTAssertEqual(index+1, shapeTableVC.tableView.numberOfRows(inSection: 0), topVCScreenshot)
47.     pauseForUIDebug()
48. }
```

- Line 44: Iterate over all of the shape cases and do the following:
- Line 45: Simulate the user tapping on the right button (the + button)
- Line 46: Assert that we have the expected number of rows in the table or fail and take a screenshot
- Line 47: Optionally pause the UI (depending on the `shouldPauseUI` setting)

<br /><br />
```swift
50. for _ in 0..<5 {
51.     shapeTableVC.addShape(rightButton)
52.     XCTAssertEqual(Shape.allCases.count, shapeTableVC.tableView.numberOfRows(inSection: 0), topVCScreenshot)
53. }
54. pauseForUIDebug()
```

- `Context:` At this point in the test, we've exhausted the happy path, which is tapping the + button as many times as we have shapes for.  In practice, the + button should be disable (we could test for that too, but have opted to skip over that).
- Line 50: Loop for an arbitrary number of times, I've chosen 5
- Line 51: Simulate the user tapping on the right button (the + button).  
    - In practice, the user shouldn't even be able to achieve this, becuase the button should be disabled.
- Line 52: Assert that we still have the same number of rows as total number of shapes or fail and take a screenshot
- Line 54: Optionally pause the UI (depending on the `shouldPauseUI` setting)
