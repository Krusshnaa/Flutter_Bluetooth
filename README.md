# Bluetooth Classic Flutter App

This is a simple Flutter app that uses the `bluetooth_classic` plugin to connect, scan, and communicate with Bluetooth devices. The app demonstrates how to handle Bluetooth operations like fetching paired devices, scanning for nearby devices, and sending/receiving data via Bluetooth.

## Features

- Connect to paired Bluetooth devices.
- Scan for nearby Bluetooth devices.
- Send and receive data via Bluetooth.
- Monitor device status and received data.

## Requirements

- Flutter SDK
- Android SDK 21 and above
- Bluetooth Classic-capable devices
- Permissions for Bluetooth usage (handled by the app)

## Installation

1. Add the `bluetooth_classic` plugin to your `pubspec.yaml` file:

```yaml
dependencies:
  flutter:
    sdk: flutter
  bluetooth_classic: ^1.0.0  # Add the appropriate version
```

2. Run `flutter pub get` to fetch the dependencies.

## Permissions

You need to request Bluetooth permissions in your `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
<uses-permission android:name="android.permission.BLUETOOTH_ADVERTISE" />
```

Also, for Android 12 and above, add the following permissions for background access:

```xml
<uses-permission android:name="android.permission.BLUETOOTH_PRIVILEGED" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
```

## How to Use

1. **Initialize Bluetooth Permissions**: The app initializes Bluetooth permissions on a button press.

2. **Get Paired Devices**: Displays the list of paired Bluetooth devices.

3. **Scan for Nearby Devices**: Starts scanning for nearby Bluetooth devices.

4. **Connect and Communicate**: You can connect to a device and send a simple "ping" message.

## Code Example

```dart
import 'package:flutter/material.dart';
import 'dart:async';
import 'package:flutter/services.dart';
import 'package:bluetooth_classic/bluetooth_classic.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatefulWidget {
  const MyApp({super.key});

  @override
  State<MyApp> createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  String _platformVersion = 'Unknown';
  final _bluetoothClassicPlugin = BluetoothClassic();
  List<Device> _devices = [];
  List<Device> _discoveredDevices = [];
  bool _scanning = false;
  int _deviceStatus = Device.disconnected;
  Uint8List _data = Uint8List(0);

  @override
  void initState() {
    super.initState();
    initPlatformState();
    _bluetoothClassicPlugin.onDeviceStatusChanged().listen((event) {
      setState(() {
        _deviceStatus = event;
      });
    });
    _bluetoothClassicPlugin.onDeviceDataReceived().listen((event) {
      setState(() {
        _data = Uint8List.fromList([..._data, ...event]);
      });
    });
  }

  Future<void> initPlatformState() async {
    String platformVersion;
    try {
      platformVersion = await _bluetoothClassicPlugin.getPlatformVersion() ?? 'Unknown platform version';
    } on PlatformException {
      platformVersion = 'Failed to get platform version.';
    }
    if (!mounted) return;
    setState(() {
      _platformVersion = platformVersion;
    });
  }

  Future<void> _getDevices() async {
    var res = await _bluetoothClassicPlugin.getPairedDevices();
    setState(() {
      _devices = res;
    });
  }

  Future<void> _scan() async {
    if (_scanning) {
      await _bluetoothClassicPlugin.stopScan();
      setState(() {
        _scanning = false;
      });
    } else {
      await _bluetoothClassicPlugin.startScan();
      _bluetoothClassicPlugin.onDeviceDiscovered().listen(
        (event) {
          setState(() {
            _discoveredDevices = [..._discoveredDevices, event];
          });
        },
      );
      setState(() {
        _scanning = true;
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        appBar: AppBar(
          title: const Text('Bluetooth Classic App'),
        ),
        body: SingleChildScrollView(
          child: Column(
            children: [
              Text("Device status is $_deviceStatus"),
              TextButton(
                onPressed: () async {
                  await _bluetoothClassicPlugin.initPermissions();
                },
                child: const Text("Check Permissions"),
              ),
              TextButton(
                onPressed: _getDevices,
                child: const Text("Get Paired Devices"),
              ),
              TextButton(
                onPressed: _deviceStatus == Device.connected
                    ? () async {
                        await _bluetoothClassicPlugin.disconnect();
                      }
                    : null,
                child: const Text("Disconnect"),
              ),
              TextButton(
                onPressed: _deviceStatus == Device.connected
                    ? () async {
                        await _bluetoothClassicPlugin.write("ping");
                      }
                    : null,
                child: const Text("Send Ping"),
              ),
              Center(
                child: Text('Running on: $_platformVersion\n'),
              ),
              ...[
                for (var device in _devices)
                  TextButton(
                    onPressed: () async {
                      await _bluetoothClassicPlugin.connect(
                        device.address,
                        "00001101-0000-1000-8000-00805f9b34fb",
                      );
                      setState(() {
                        _discoveredDevices = [];
                        _devices = [];
                      });
                    },
                    child: Text(device.name ?? device.address),
                  )
              ],
              TextButton(
                onPressed: _scan,
                child: Text(_scanning ? "Stop Scan" : "Start Scan"),
              ),
              ...[
                for (var device in _discoveredDevices)
                  Text(device.name ?? device.address)
              ],
              Text("Received data: ${String.fromCharCodes(_data)}"),
            ],
          ),
        ),
      ),
    );
  }
}
```


