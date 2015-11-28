---
layout: post
title: "Introduction to Google Eddystone"
date: 2015-11-28 15:49:19 +0100
comments: true
categories: eddystone, beacons, android
---

Bluetooth beacons are transmitters that use Bluetooth Low Energy 4.0 (BLE) to broadcast signals that can be heard by compatible or smart devices.  These transmitters can be powered by batteries or a fixed power source such as a USB adapter. When a smart device is in a beacon’s proximity, the beacon will automatically recognize it and will be able to interact with that device.

A few weeks ago, we were lucky enough to be introduced to the Eddystone project by the team here in London. It's certainly a very interesting project and I thought it'd be a good idea to share our experience.

{% img https://lh3.googleusercontent.com/gcEhJFg9o3geDddamHn-HQthPyaYhyyWxAC7nY2E6LI-bBCkHeLsPqB7ZpebwAFXsDvlx9V66Jqw98YlTy6DxeE4soDA7Q=s1600 %}

<!-- more -->

# What's the Google Eddystone project

Eddystone is a protocol specification that defines a BLE message format for proximity beacon messages. It is robust and extensible: It describes several different frame types that may be used individually or in combinations to create beacons that can be used for a variety of applications. It’s cross-platform, capable of supporting Android, iOS or any platform that supports BLE beacons. And it’s available on [GitHub](https://github.com/google/eddystone) under the open-source Apache v2.0 license, for everyone to use and help improve.

# Register your beacons

Before we can start playing with the APIs we need to provision our devices. Since beacons is not something you can buy from your local grocery store, you will need to purchase one of the developer kits from one of the [several manufacturers](https://developers.google.com/beacons/?#beacon-manufacturers) which Google has already partnered with.

In our case, we are going to use one of Developer Kits from [Estimote](http://estimote.com/)

{% img https://order.estimote.com/images/box_devkit.png %}

Unfortunately, each manufacturer has it's own process to provision the beacons. For that reason, I'd strongly recommend sticking with just one manufacturer for your development purposes. Estimote happens to have a very simple process in order to provision the beacons, just a couple of minutes and we were ready to go!

## No beacons? No problem

It's also possible to give a go at the Eddystone platform without purchasing some beacons. There is a [simple Android application](https://github.com/google/eddystone/tree/master/eddystone-uid/tools/txeddystone-uid) that allows your phone to broadcast Eddystone-UID BLE packets. Only some more recent devices are capable of using this ability classes. The application will check if your device is compatible and show an error dialog if not. The Nexus 6 phone and Nexus 9 tablet are known to work well.

# Getting started

In order to make use of the beacons platform, you'll need to setup a project in the Google Developers Console account. Follow [these instructions](https://developers.google.com/beacons/proximity/get-started) to create a project and register the application with your account. You'll have to enable the [Google Proximity Beacon API](https://developers.google.com/beacons/proximity/guides) and create both an Android API Key and an Android OAuth 2.0 client ID.

As it turns out, following this order is very important. It took me a while to figure that out, but creating an API Key before enabling the permission was causing some problems and I wasn't able to connect to the Google services. After creating the project and adding the proper APIs, this is what you should be seeing.

{% img https://raw.githubusercontent.com/malmstein/eddystone-sample/master/art/gdc_setup.png %}

# Google Proximity Beacon API

One of the biggest advantages that the Proximity API introduces is the ability to manage your BLE beacons using a REST interface. It's now possible to manage an entire beacon fleet from your living room with ease. There is no need to be physically next to the beacon in order to manage it. If you take a look at the [complete reference](https://developers.google.com/beacons/proximity/reference/rest/) you'll see that this API provides services to register, manage, index, and search beacons.

The authentication happens using an [OAuth](https://developers.google.com/identity/protocols/OAuth2) access token from a signed-in user with **Is owner** or **Can edit** permissions in the Google Developers Console project.

# The sample application

For the purpose of this post, I've built an application that **allows you to register and manage** Eddystone beacons. It's very simple to use and showcases several endpoints from the Proximity API that are worth looking at.

{% img https://developers.google.com/beacons/images/beacon_integration_overview.png %}

# Scanning Beacons

Before we can start registering and managing the beacons, seems obvious but we first need to find them, The setup is not very complicated, however it's very important to understand what's happening. First of all, we need to add the permissions so the application has access access to Bluetooth sensor.

{% codeblock lang:xml %}
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
{% endcodeblock %}

Lucky for us, [BluetoothLeScanner](http://developer.android.com/reference/android/bluetooth/le/BluetoothLeScanner.html) provides methods to perform scan related operations for Bluetooth LE devices. We can restrict the scan to only those beacons that are of interest to us [ScanFilter](http://developer.android.com/reference/android/bluetooth/le/ScanFilter.html) and we can also request different types of callbacks for delivering the result.

{% codeblock lang:java %}
// The Eddystone Service UUID, 0xFEAA.
public static final ParcelUuid EDDYSTONE_SERVICE_UUID =
        ParcelUuid.fromString("0000FEAA-0000-1000-8000-00805F9B34FB");

// A filter that scans only for devices with the Eddystone Service UUID.
private static final ScanFilter EDDYSTONE_SCAN_FILTER = new ScanFilter.Builder()
        .setServiceUuid(EDDYSTONE_SERVICE_UUID)
        .build();

// An aggressive scan for nearby devices that reports immediately.
public static final ScanSettings SCAN_SETTINGS =
        new ScanSettings.Builder().
                setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY)
                .setReportDelay(0)
                .build();

public void startScan(BluetoothAdapter btAdapter) {
    scanner = btAdapter.getBluetoothLeScanner();
    scanner.startScan(EDDYSTONE_SCAN_FILTER,
        SCAN_SETTINGS,
        scanCallback);
}
{% endcodeblock %}

Since we are only interested in **Eddystone** beacons we need to create a filter that scans only devices with **Eddystone Service UUID**. The second parameter in this method corresponds to the scan settings used to define the parameters for the scan via [ScanSettings](http://developer.android.com/reference/android/bluetooth/le/ScanSettings.htmlused). There are three different scan modes available: [balanced](http://developer.android.com/reference/android/bluetooth/le/ScanSettings.html#SCAN_MODE_BALANCED), [low latency](http://developer.android.com/reference/android/bluetooth/le/ScanSettings.html#SCAN_MODE_LOW_LATENCY) and [low power](http://developer.android.com/reference/android/bluetooth/le/ScanSettings.html#SCAN_MODE_LOW_POWER). For the purpose of this example we are using an aggressive approach since we want to react to the beacons as soon as possible. This approach is **only recommended when the application is running in the foreground**.

Now that we've decided the scanning approach we want to follow it's time to look at the **Callback** that the BluetoothAdapter calls.

{% codeblock lang:java %}
ScanCallback scanCallback = new ScanCallback() {
    @Override
    public void onScanResult(int callbackType, ScanResult result) {
      ...
    }
};
{% endcodeblock %}

At this point we need to check if we are receiving a valid Eddystone beacon. What the adapter sends us back is a [ScanResult](http://developer.android.com/reference/android/bluetooth/le/ScanResult.html) which may or may not be an Eddystone beacon. We'll use the frame type byte to do the validation.

{% codeblock lang:java %}
// The Eddystone-UID frame type byte https://github.com/google/eddystone
public static final byte EDDYSTONE_UID_FRAME_TYPE = 0x00;

public static boolean isValid(ScanResult result) {
  if (result.getScanRecord() == null) {      
      return false;
  }

  ScanRecord scanRecord = result.getScanRecord();
  byte[] serviceData = scanRecord.getServiceData(EDDYSTONE_SERVICE_UUID);
  if (serviceData == null) {
      return false;
  }

  if (serviceData[0] != Eddystone.EDDYSTONE_UID_FRAME_TYPE) {
      return false;
  }
  return true;
}
{% endcodeblock %}

We are now able to recognize if the scan result is an Eddystone beacon or not, but still don't know anything about the beacon itself. In order to do that we'll need to use the Google Client API, which is why earlier we had to create a Project in the Developers Console.

{% img https://raw.githubusercontent.com/malmstein/eddystone-sample/master/art/unspecified_beacons.png %}

# Authentication

You'll only be able to manage the beacons that are associated to your Google account. There are several steps involved in this process and we'll cover them all in a second.

First of all you'll need to associate your account with the project that we created earlier in the Google Developer Console.

{% codeblock lang:java %}
@Override
 public void onSignInClicked() {
     String[] accountTypes = new String[]{"com.google"};
     Intent intent = AccountPicker.newChooseAccountIntent(
             null, null, accountTypes, false, null, null, null, null);
     startActivityForResult(intent, REQUEST_CODE_PICK_ACCOUNT);
 }
{% endcodeblock %}

 We'll use the `AccountPicker` from Play Services in order to start the authentication flow. Once the account is selected we'll start the **Authentication** call. Using Play Services and the useful `GoogleAuthUtil` class we'll be able to generate the token that validates the user. Besides the account name, we'll also need to send the **AUTH_SCOPE** for the task. In this case, authentication to register the beacons.

 Yes, it's an `AsyncTask` but obviously you are more than welcome to handle this with your preferred Non-UI thread solution.

{% codeblock lang:java %}
AUTH_SCOPE = "oauth2:https://www.googleapis.com/auth/userlocation.beacon.registry";

@Override
protected Void doInBackground(Void... params) {
  Log.i(TAG, "checking authorization for " + accountName);
  try {
      GoogleAuthUtil.getToken(activity, accountName, AUTH_SCOPE);
  } catch (UserRecoverableAuthException e) {
      // GooglePlayServices.apk is either old, disabled, or not present
      // so we need to show the user some UI in the activity to recover.
      handleAuthException(activity, e);
  } catch (GoogleAuthException e) {
      // Some other type of unrecoverable exception has occurred.
      // Report and log the error as appropriate for your app.
      Log.w(TAG, "GoogleAuthException: " + e);
  } catch (IOException e) {
      // The fetchToken() method handles Google-specific exceptions,
      // so this indicates something went wrong at a higher level.
      // TIP: Check for network connectivity before starting the AsyncTask.
  }
  return null;
}
{% endcodeblock %}

The first time the user starts the authentication flow a dialog will appear:

{% img https://raw.githubusercontent.com/malmstein/eddystone-sample/master/art/authentication.png %}

# Accessing to a beacon data

At this point we can both scan beacons and use our account to register them with Eddystone, however we are still not reading any data from Eddystone yet. The magic happens in the [BluetoothScanner](https://github.com/malmstein/eddystone-sample/master/proximitybeacon/BluetoothScanner) class thanks to the interface [ProximityBeacon](https://github.com/malmstein/eddystone-sample/master/proximitybeacon/ProximityBeacon). This interface maps all the rest endpoints supported by Eddystone and that will allow us to access all it's capabilities. We wouldn't be able to access all these endpoints unless our account has been previously authenticated, this is why is so important to follow the proper order of steps to get here.

{% codeblock lang:java %}
ScanCallback scanCallback = new ScanCallback() {
    @Override
    public void onScanResult(int callbackType, ScanResult result) {      
      Beacon scannedBeacon = validateScan(result);
      if (scannedBeacon != null) {
        if ((proximityBeacon == null)) {
            listener.onBeaconScanned(beacon);
        } else if (!proximityBeacon.hasAccount()) {
            listener.onBeaconScanned(beacon);
        } else {
            proximityBeacon.getBeacon(GetBeaconCallback, beacon.getBeaconName());
        }
      }
    }
};
{% endcodeblock %}

You are already familiar with this callback, it's what we receive from the `BluetoothScanner` when locating a beacon. After validating that the scan result is indeed an Eddystone beacon, we'll make use of the `ProimityBeacon` API in order to retrieve it's data.

{% codeblock lang:java %}
proximityBeacon.getBeacon(new Callback() {
    @Override
    public void onFailure(Request request, IOException e) {
        Log.e(TAG, String.format("Failed request: %s, IOException %s", request, e));
    }

    @Override
    public void onResponse(Response response) throws IOException {
        Beacon fetchedBeacon;
        switch (response.code()) {
            case 200:
                try {
                    String body = response.body().string();
                    fetchedBeacon = beaconConverter.from(new JSONObject(body));
                } catch (JSONException e) {
                    Log.e(TAG, "JSONException", e);
                    return;
                }
                break;
            case 403:
                fetchedBeacon = new Beacon(beacon.getType(), beacon.getId(), Beacon.Status.NOT_AUTHORIZED);
                break;
            case 404:
                fetchedBeacon = new Beacon(beacon.getType(), beacon.getId(), Beacon.Status.UNREGISTERED);
                break;
            default:
                Log.e(TAG, "Unhandled beacon service response: " + response);
                return;
        }
    }
}, beacon.getBeaconName());
{% endcodeblock %}

The Proximity Beacon API will return a JSON response when the request is successful. We'll get more into detail about that JSON to Object conversion in a second. Firstly, I wanted to drive your attention to the other two status code responses, 403 and 404.

We'll get a 403 if the request is made with an account that has **not authorized access** to the Eddystone platform as seen before. This means that we can't read or modify the beacon data related to Eddystone. Receiving a 404 means that the beacon itself is not registered with the platform.

{% img https://raw.githubusercontent.com/malmstein/eddystone-sample/master/art/beacons.png %}

If you take a look at the [BeaconConverter](https://github.com/malmstein/eddystone-sample/master/model/beaconConverter) you'll get to see all the different values that the platform sends us. Such as `advertisingId`, `description` and even more interesting things like `placeId` which gives is the Google Places Id which this beacon has been assigned to.

# Changing the data associated with a beacon

Once we've done all the dance of scanning, authenticating and receiving the beacons, we can finally look what's inside them! To make it easy I'm presenting the data in three separate `Cards`: **Info**, **Location** and **Attachments**.

Any time we want to modify the data of a beacon we'll need to make a call to `updateBeacon` from the REST API.

{% codeblock lang:java %}
private void updateRemoteBeacon() {
    JSONObject json = beaconConverter.toJson(beacon);   
    proximityBeacon.updateBeacon(updateBeaconCallback, beacon.getBeaconName(), json);

    Callback updateBeaconCallback = new Callback() {
    @Override
    public void onFailure(Request request, IOException e) {
        Log.d(TAG, "Failed request: " + request, e);      
    }

    @Override
    public void onResponse(Response response) throws IOException {
        String body = response.body().string();
        if (response.isSuccessful()) {
            bluetoothScanner.fetchBeaconStatus(beacon);
        } else {
            Log.d(TAG, "Unsuccessful updateBeacon request: " + body);          
        }
    }
};
}
{% endcodeblock %}

As you can see we are following the same pattern as before, there is a remote endpoint that we hit and a callback when we check the response. The difference here is that we need to pass a JSON object to the `updateMethod` call. The [BeaconConverter](https://github.com/malmstein/eddystone-sample/master/model/beaconConverter) class does that for us.

{% img https://raw.githubusercontent.com/malmstein/eddystone-sample/master/art/manage_beacon.png %}

## Adding Attachments to a Beacon

One of the most interesting features of Eddystone is the ability to add attachments to your beacons. This means that you could add some information there that your application users could read (not modify). Imagine a train station with 20 rail lines and several beacons placed next to each line. Your application could ready the attachment from one of those beacons and tell the user that he / she is standing next to the middle section of rail line number 12. The possibilities are endless!

When you create an attachment, you must populate two fields.
* namespacedType: a string made up of a namespace identifier, followed by a forward slash and the data type. The namespace identifier can't be modified and is assigned to your Google Developer Console project.
* data: a base64 encoded value of the data type defined in the namespacedType field.

You may have noticed that `updateBeacon` does not modify the attachments of a beacon. That's because there is a different endpoint for it. Each attachment is translated to it's own JSON Object and sent to the Proximity Beacon API.

{% codeblock lang:java %}
public void addAttachment(final ProximityBeacon proximityBeacon, String type, String data) {
    this.proximityBeacon = proximityBeacon;

    JSONObject body = buildCreateAttachmentJsonBody(namespace, type, data);

    Callback createAttachmentCallback = new Callback() {
        @Override
        public void onFailure(Request request, IOException e) {
            Log.d(TAG, "Failed request: " + request, e);
        }

        @Override
        public void onResponse(Response response) throws IOException {
            String body = response.body().string();
            if (response.isSuccessful()) {
                try {
                    JSONObject json = new JSONObject(body);
                    addAttachmentView(json);
                } catch (JSONException e) {
                    Log.d(TAG, "JSONException in building attachment data", e);
                }
            } else {
                Log.d(TAG, "Unsuccessful createAttachment request: " + body);
            }
        }
    };

    proximityBeacon.createAttachment(createAttachmentCallback, beacon.getBeaconName(), body);
}
{% endcodeblock %}

{% img https://raw.githubusercontent.com/malmstein/eddystone-sample/master/art/attachments.png %}

## Places API

Another interesting feature of Eddystone is the ability to assign a Beacon to a specific Google Places ID. The very handy [LatLng]() class from Play Services allow us to request a Place Picker which will return a [Place]() object which gives us all the information we want.

{% codeblock lang:java %}
private void startPlacePickerIntnet(double latitude, double longitude){
  LatLng latLng = new LatLng(latitude, longitude);

  PlacePicker.IntentBuilder builder = new PlacePicker.IntentBuilder();
  if (latLng != null) {
      builder.setLatLngBounds(new LatLngBounds(latLng, latLng));
  }
  startActivityForResult(builder.build(this), REQUEST_CODE_PLACE_PICKER);
}

@Override
public void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == REQUEST_CODE_PLACE_PICKER) {
        if (resultCode == Activity.RESULT_OK) {
            Place place = PlacePicker.getPlace(data, this);
            beaconLocationView.addPlace(place);
            beacon.setPlace(place);
            updateRemoteBeacon();
        }
    }
}

{% endcodeblock %}

{% img https://raw.githubusercontent.com/malmstein/eddystone-sample/master/art/places.gif %}

# Conclusion

Eddystone is a great platform that, once implemented, will bring a whole new set of experiences for our day to day life. There are literally thousands of use cases where the user will be the main beneficiary of this technology. I can't wait to see what developers and companies come up with!

If you are curious to see the whole implementation, all the source code is published on [Github](https://github.com/malmstein/eddystone-sample). As always, all the feedback is welcome!

# Next steps

You may have seen an empty Tab in the application, it corresponds to the `Nearby Messages API` which I'll be covering soon. Stay tuned!


{% codeblock lang:java %}

{% endcodeblock %}
