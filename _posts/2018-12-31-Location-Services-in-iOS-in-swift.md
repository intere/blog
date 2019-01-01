---
layout: post
title: iOS Location Services in Swift
---

Have you ever used Location Services in iOS?  Do you use them often enough to remember how to do it?  If you answered no to either of these questions and you're trying to use location services, then read on.

### Scope
This is not meant to be a comprehensive guide to Location Services, but a "getting started" guide as a developer that is new to Location Services.  We'll also be looking at a single use case of location services: obtaining the highest quality location data (aka: highest battery consuming).  We'll be moving relatively quickly, but there is a [reference project]({{ site.baseurl }}/images/2018-12-31/LocationServices.zip) that you can download and try for yourself.

`Note:` I maintain a library ([GeoTrackKit](https://github.com/intere/GeoTrackKit)) that does much of this already and adds a bunch of utilities on top of the location services, but there are still steps you need to take in your own application when integrating this library.

## Strategy
In principle, it's pretty easy, you do the following:
1. Update your `Info.plist` file to enable location services
    - `Note:` there are 3 possible keys that you can use 1, two or all of them
2. When appropriate, request permission to use location services
    - `Note:` there are 2 modes: `When in use` and `Always`, we'll talk about the difference later
3. After the authorization callback notifies you that you have been granted access, configure the `LocationManager` and begin tracking.

## Code Walkthrough
In this code walkthrough we're going to build a sample application that uses location services.  This application will have the following capabilities:
- Request Access to Location Services
- Open settings if access to Location Services were denied
- If Access to Location Services is granted:
    - Start tracking the user's location
    - Display the location information on the screen (when tracking)
    - Allows the user to stop tracking if they are currently tracking

### 1. Create a new project:
![_config.yml]({{ site.baseurl }}/images/2018-12-31/01_new_project.gif)

### 2. Update Info.plist
![_config.yml]({{ site.baseurl }}/images/2018-12-31/02_info_plist.gif)
Add the following to Info.plist:

```xml
<key>NSLocationWhenInUseUsageDescription</key>
<string>Your location will be used to demonstrate location services when the application is in use.</string>
```
`NOTE:` if you do not add the above line, when you request location services authorization, your app will be terminated.

### 3. Location Services Authorization State
![_config.yml]({{ site.baseurl }}/images/2018-12-31/03_authorization.gif)
There are numerous authorization states in Location Services, so let's distill it down to a few simplified states using our own enumeration, `LocationAuthorizationState`:

```swift
/// The current Location Services Authorization state
enum LocationAuthorizationState: String {
    case unknownAuthorization
    case deniedAuthorization
    case approvedAuthorization

    /// Distills the current location authorization status down to three
    /// simple states:
    static var current: LocationAuthorizationState {
        switch CLLocationManager.authorizationStatus() {
        case .notDetermined:
            return .unknownAuthorization

        case .denied, .restricted:
            return .deniedAuthorization

        case .authorizedAlways, .authorizedWhenInUse:
            return .approvedAuthorization
        }
    }
}
```

Then, we set the text of our button, based on the current authorization state from the `ViewController`:
```swift
/// Updates the title of the button, based on the authorization state
/// and tracking state.
func updateButtonTitle() {
    switch LocationAuthorizationState.current {
    case .approvedAuthorization:
        // TODO: set it based on whether or not we're tracking (start / stop)
        actionButton.setTitle("Track", for: .normal)

    case .unknownAuthorization:
        actionButton.setTitle("Request Access", for: .normal)

    case .deniedAuthorization:
        actionButton.setTitle("Open Settings", for: .normal)
    }
}
```

### 4. Request Location Authorization
First, Add a `CLLocationManager` to our ViewController:
```swift
let manager = CLLocationManager()
```

Next, set the manager's delegate to be our ViewController inside `viewDidLoad()`:
```swift
manager.delegate = self
```

Next, conform our `ViewController` to the `CLLocationManagerDelegate` and implement the `locationManager(_:didChangeAuthorization)` function:

```swift
// MARK: - Location Delegate

extension ViewController: CLLocationManagerDelegate {

    func locationManager(_ manager: CLLocationManager, didChangeAuthorization status: CLAuthorizationStatus) {
        updateButtonTitle()
    }

}
```

![_config.yml]({{ site.baseurl }}/images/2018-12-31/04_request_access.gif)

### 5. Build in tracking / finish off proof of concept
First, add a boolean to keep track of whether we're tracking or not:
```swift
private var tracking = false
```

Create a couple of helper functions to start / stop tracking for us (and configure location services)
```swift
/// Begin tracking
func startTracking() {
    tracking = true
    configureUpdates()
    manager.startUpdatingLocation()
}

/// Finish tracking
func stopTracking() {
    tracking = false
    manager.stopUpdatingLocation()
}

/// Configure the Location Manager for our trackign requirements
func configureUpdates() {
    manager.activityType = .fitness
    manager.desiredAccuracy = kCLLocationAccuracyBest
}
```

Create a helper function to open the settings for our app (in the case that the user has prevented access to location services):
```swift
/// Opens the system settings for our app
func openSettings() {
    guard let systemSettingsUrl = URL(string: UIApplication.openSettingsURLString) else {
        return
    }
    UIApplication.shared.openURL(systemSettingsUrl)
}
```

Refactor the `updateButtonTitle` function to be called `updateUIState` and make it update all of the UI components in our `ViewController` class:
```swift
/// Updates the title of the button, based on the authorization state
/// and tracking state.
func updateUIState() {
    switch LocationAuthorizationState.current {
    case .approvedAuthorization:
        authLabel.text = "Access approved by user"
        if tracking {
            actionButton.setTitle("Stop Tracking", for: .normal)
        } else {
            trackingLabel.text = "Not Tracking"
            actionButton.setTitle("Start Tracking", for: .normal)
        }

    case .unknownAuthorization:
        authLabel.text = "Access not yet requested"
        actionButton.setTitle("Request Access", for: .normal)

    case .deniedAuthorization:
        authLabel.text = "Access denied by user"
        actionButton.setTitle("Open Settings", for: .normal)
    }
}
```

Implement the `locationManager(_:didUpdateLocations)` function as part of the `CLLocationManagerDelegate` conformance:

```swift
func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
    guard let lastPoint = locations.last else {
        return
    }
    trackingLabel.text = "\(lastPoint)"
}
```

And finally, update the `tappedButton(_:)` function to do the right thing:
```swift
func tappedButton(_ source: Any) {
    switch LocationAuthorizationState.current {
    case .unknownAuthorization:
        manager.requestWhenInUseAuthorization()

    case .approvedAuthorization:
        if tracking {
            stopTracking()
        } else {
            startTracking()
        }
        updateUIState()

    case .deniedAuthorization:
        openSettings()
    }
}
```
![_config.yml]({{ site.baseurl }}/images/2018-12-31/05_tracking.gif)

## References

### Sample Code
- [Sample Code]({{ site.baseurl }}/images/2018-12-31/LocationServices.zip)
    - Written with swift 4.2 in Xcode 10.1
- [GeoTrackKit](https://github.com/intere/GeoTrackKit)

### Further reading
- [Requesting Always Authorization](https://developer.apple.com/documentation/corelocation/choosing_the_authorization_level_for_location_services/requesting_always_authorization?language=swift)
- [Requesting When In Use Authorization](https://developer.apple.com/documentation/corelocation/cllocationmanager/1620562-requestwheninuseauthorization)
- [Core Location](https://developer.apple.com/documentation/corelocation)

### WWDC Videos
- [2017 - What's New in Location Technologies](https://developer.apple.com/videos/play/wwdc2017/713/)
- [2016 - Core Location Best Practices](https://developer.apple.com/videos/play/wwdc2016/716/)
