# Bluejay

Bluejay is a simple Swift framework for building reliable Bluetooth apps.

Bluejay's primary goals are:
- Simplify talking to a Bluetooth device
- Make good use of Swift features and conventions
- Make it easier to build Bluetooth apps that are actually reliable

## Requirements

- iOS 10 or above
- Xcode 8.2.1 or above

## Getting started

Install using CocoaPods:

`pod 'Bluejay', :git => 'https://github.com/steamclock/bluejay.git', :branch => 'master'`

Import using:

```swift
import Bluejay
```

## Demo

Simulator does not work without a BLE dongle.

1. Prepare two iOS devices, one will act as a virtual BLE peripheral, and the other will run the demo app using the Bluejay API.
2. On the iOS device serving as a virtual BLE peripheral, go to the App Store and download the free [LightBlue Explorer](https://itunes.apple.com/ca/app/lightblue-explorer-bluetooth/id557428110?mt=8) app.
3. Open the LightBlue Explorer app, and tap on the "Create Virtual Peripheral" button located at the very bottom of the peripheral list.
4. For simplicity, choose "Heart Rate" from the base profile list, and finish by tapping the "Save" button located at the top right of the screen.
5. Finally, build and run the BluejayDemo app on the other iOS device, and you will be able to interact with the virtual heart rate peripheral using Bluejay.

Notes:

- You can turn the virtual peripheral on or off in LightBlue Explorer by tapping and toggling the blue checkmark to the left of the virtual peripheral's name in the peripheral list.
	- If the virtual peripheral is not working as expected, you can try to reset it this way.
- By default, the demo app will read and write to the "Body Sensor Location" characteristic, and listen to the "Heart Rate Measurement" characteristic.
	- The heart rate measurement returns only zeroes by default, because it is a virtual peripheral without an actual heart rate detector.
- You can use LightBlue Explorer to do CRUD on various characteristics in the virtual peripheral.

## Usage

The Bluejay interface can be accessed through using the Bluejay singleton:

```swift
fileprivate let bluejay = Bluejay.shared
```

### Initialization

Start Bluejay at the appropriate time during initialization. When it is appropriate depends on the context and requirement of your app. For example, in the demo app Bluejay is started inside `viewDidLoad` of the only existing view controller.

```swift
bluejay.start(connectionObserver: self, listenRestorer: self, enableBackgroundMode: true)
```

Having to explicitly start Bluejay is important because this gives your app an opportunity to make sure two critical delegates are instantiated and available before the CoreBluetooth stack is initialized. This will ensure CoreBluetooth's startup and restoration events are being handled.

Note:

Background mode is disabled by default. In order to support background mode, you must set the parameter `enableBackgroundMode` to true in the above `start` function, as well as turn on the "Background Modes" capability in your Xcode project with "Uses Bluetooth LE accessories" enabled.

### Bluetooth Events

The `observer` conforms to the `ConnectionObserver` protocol, and allows the delegate to react to major connection-related events:

```swift
public protocol ConnectionObserver: class {
    func bluetoothAvailable(_ available: Bool)
    func connected(_ peripheral: Peripheral)
    func disconected()
}
```

Beyond the `start` function, you can also add other observers using:

```swift
bluejay.register(observer: batteryLabel)
```

Unregistering an observer is not necessary, because Bluejay only holds weak references to registered observers. But if you require unregistering an observer explicitly, use:

```swift
bluejay.unregister(observer: batteryLabel)
```

### Listen Restoration

Listen restoration is only relevant if your app supports background modes.

From Apple's doc:

> [CoreBluetooth can] preserve the state of your app’s central and peripheral managers and to continue performing certain Bluetooth-related tasks on their behalf, even when your app is no longer running. When one of these tasks completes, the system relaunches your app into the background and gives your app the opportunity to restore its state and to handle the event appropriately.

Therefore, when your app has ceased running either due to memory pressure or by staying in the background past the allowed duration (max 3min since iOS 7), then the next time your app is launched, the `ListenRestorer` protocol provides an opportunity to restore the lost callbacks to the still ongoing listens.

```swift
public protocol ListenRestorer: class {
    func didFindRestorableListen(on characteristic: CharacteristicIdentifier) -> Bool
}
```

By default, if there is no `ListenRestorer` delegate provided in the `start` function, then **all** active listens will be ended during state restoration.

The listen restorer can only be provided to Bluejay via its `start` function, because the restorer must be available to respond before CoreBluetooth initiates state restoration.

To restore the callback on the given characteristic, use the `restoreListen` function and return true, otherwise, return false and Bluejay will end listening on that characteristic for you:

```swift
extension ViewController: ListenRestorer {

    func didFindRestorableListen(on characteristic: CharacteristicIdentifier) -> Bool {
        if characteristic == heartRate {
            bluejay.restoreListen(to: heartRate, completion: { (result: ReadResult<UInt8>) in
                switch result {
                case .success(let value):
                    log.debug("Listen succeeded with value: \(value)")
                case .failure(let error):
                    log.debug("Listen failed with error: \(error.localizedDescription)")
                }
            })

            return true
        }

        return false
    }

}
```

### Specifying Services & Characteristics

On a high level, a Service is like a category, and a Characteristic is an attribute belonging to a category. For example, BLE peripherals that can detect heart rates usually have a Service named "Heart Rate" with a UUID of "180D". And inside that Service are Characteristics such as, "Body Sensor Location" with a UUID of "2A38", as well as "Heart Rate Measurement" with a UUID of "2A37".

Many of these Services and Characteristics are standards specified by the Bluetooth SIG organization, and most hardware adopt their specifications. For example, a common Service that can be found in BLE peripherals is "Device Information" with a UUID of "180A", where Characteristics such as firmware version, serial number, and other hardware details can be read and written.

Of course, there are many usage of BLE peripherals not covered by the Bluetooth Core Spec, and custom hardwares often have their own unique Services and Characteristics.

Here is how you can specify Services and Characteristics for use in Bluejay:

```swift
let heartRateService = ServiceIdentifier(uuid: "180D")
let bodySensorLocation = CharacteristicIdentifier(uuid: "2A38", service: heartRateService)
let heartRate = CharacteristicIdentifier(uuid: "2A37", service: heartRateService)
```

### Scan & Connect

```swift
bluejay.scan(service: heartRateService) { (result) in
    switch result {
    case .success(let peripheral):
        log.debug("Scan succeeded with peripheral: \(peripheral.name)")

        self.bluejay.connect(PeripheralIdentifier(uuid: peripheral.identifier), completion: { (result) in
            switch result {
            case .success(let peripheral):
                log.debug("Connect succeeded with peripheral: \(peripheral.name)")
            case .failure(let error):
                log.debug("Connect failed with error: \(error.localizedDescription)")
            }
        })
    case .failure(let error):
        log.debug("Scan failed with error: \(error.localizedDescription)")
    }
}
```

### Disconnect

```swift
bluejay.disconnect()
```

### Read

```swift
bluejay.read(from: bodySensorLocation) { (result: ReadResult<IncomingString>) in
    switch result {
    case .success(let value):
        log.debug("Read succeeded with value: \(value.string)")
    case .failure(let error):
        log.debug("Read failed with error: \(error.localizedDescription)")
    }
}
```

### Write

```swift
bluejay.write(to: bodySensorLocation, value: OutgoingString("Wrist")) { (result) in
    switch result {
    case .success:
        log.debug("Write succeeded.")
    case .failure(let error):
        log.debug("Write failed with error: \(error.localizedDescription)")
    }
}
```

### Listen

```swift
bluejay.listen(to: heartRate) { (result: ReadResult<UInt8>) in
    switch result {
    case .success(let value):
        log.debug("Listen succeeded with value: \(value)")
    case .failure(let error):
        log.debug("Listen failed with error: \(error.localizedDescription)")
    }
}
```

### End Listen

```swift
bluejay.endListen(to: heartRate)
```

### Receivable & Sendable

The `Receivable` and `Sendable` protocols provide the blueprints to model the data packets being exchanged between the your app and the BLE peripheral, and are mandatory to type the results and the deliverables when using the `read` and `write` functions.

Examples:

```swift
struct IncomingString: Receivable {

    var string: String

    init(bluetoothData: Data) {
        string = String(data: bluetoothData, encoding: .utf8)!
    }

}

struct OutgoingString: Sendable {

    var string: String

    init(_ string: String) {
        self.string = string
    }

    func toBluetoothData() -> Data {
        return string.data(using: .utf8)!
    }

}
```

## Documentaion

- Link out or straight-up include the API docs
