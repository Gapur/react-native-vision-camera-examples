<p align="center">
  <img width="500" src="https://github.com/Gapur/react-native-vision-camera-examples/blob/main/assets/logo.png" />
</p>

# Using React Native VisionCamera

In a previous article, I wrote about [how to use React Native Camera](https://blog.logrocket.com/intro-to-react-native-camera/), which is now, unfortunately, deprecated. Today, I’m back to talk about other options we can use in our React Native projects to replace React Native Camera.

We’ll introduce React Native VisionCamera and assess some other alternatives to help you choose which camera library to use in your next application. In this piece, I’ll demonstrate what  VisionCamera can do by developing a QR code scanner. Let’s get started!

## Introducing React Native VisionCamera

[VisionCamera](https://mrousavy.com/react-native-vision-camera/) is a fully featured camera library for React Native. Some key benefits include:

- Rich developer support: VisionCamera has a large developer community and every feature is well documented and supported
- Feature-rich: VisionCamera offers all the features you’d want from a modern smartphone camera, plus full control over what device is being used — it can even adjust settings like frame rate and more
- Easy to use: You can easily start using the library with Hooks and functions with full control
Let’s explore a few of the key functions you’ll find yourself using quite often.

## Taking photos

First, we need to enable photo capture in order to take a photo:

```js
<Camera {...props} photo={true} />
```

Then, just use VisionCamera’s [takePhoto](https://mrousavy.com/react-native-vision-camera/docs/api/classes/Camera/#takephoto) method:

```js
const photo = await camera.current.takePhoto({
  flash: 'on'
})
```

<p align="center">
  <img src="https://github.com/Gapur/react-native-vision-camera-examples/blob/main/assets/example1.gif" />
</p>

## Taking snapshots

Take snapshots with VisionCamera’s [takeSnapshot(...)](https://mrousavy.com/react-native-vision-camera/docs/api/classes/Camera/#takesnapshot) function like so:

```js
const snapshot = await camera.current.takeSnapshot({
  quality: 85,
  skipMetadata: true
})
```

<p align="center">
  <img src="https://github.com/Gapur/react-native-vision-camera-examples/blob/main/assets/example2.png" />
</p>

## Recording video

To record video, you’ll first have to enable video capture for a video recording:

```js
<Camera
  {...props}
  video={true}
  audio={true} // <-- optional
/>
```
Then we can record a video via VisionCamera’s [startRecording(...)](https://mrousavy.com/react-native-vision-camera/docs/api/classes/Camera/#takesnapshot) function:

```js
camera.current.startRecording({
  flash: 'on',
  onRecordingFinished: (video) => console.log(video),
  onRecordingError: (error) => console.error(error),
})
```

Once we start recording, we can stop it like so:

```js
await camera.current.stopRecording()
```

<p align="center">
  <img src="https://github.com/Gapur/react-native-vision-camera-examples/blob/main/assets/example3.gif" />
</p>

# Building a QR code scanner with VisionCamera

## Setting up the project

Before we get started, we’ll need to [create a new React Native project](https://blog.logrocket.com/create-react-native-app-using-ignite-boilerplate/). We can do that with the following commands:

```sh
npx react-native init react_native_image_detector
cd react_native_image_detector
yarn ios
```

Great, now we can start installing dependencies! First and foremost, we need to install React Native VisionCamera with the following commands:

```sh
yarn add react-native-vision-camera
npx pod-install 
```

Now, in order to use the camera or microphone, we must add the iOS permissions to the `Info.plist`:

```sh
<key>NSCameraUsageDescription</key>
<string>$(PRODUCT_NAME) needs access to your Camera.</string>

<!-- optionally, if reac want to record audio: -->
<key>NSMicrophoneUsageDescription</key>
<string>$(PRODUCT_NAME) needs access to your Microphone.</string>
```

For Android, we will add the following lines of code to our `AndroidManifest.xml` file in the `<manifest>` tag:

```sh
<uses-permission android:name="android.permission.CAMERA" />

<!-- optionally, if you want to record audio: -->
<uses-permission android:name="android.permission.RECORD_AUDIO" />
```

We will also use the [vision-camera-code-scanner](https://github.com/rodgomesc/vision-camera-code-scanner) plugin to scan the code using ML Kit’s barcode scanning API. Let’s install it with the following command:

```sh
yarn add vision-camera-code-scanner
```

To label a camera QR code as text, we need to install React Native Reanimated by running the following command:

```sh
yarn add react-native-reanimated
```

Lastly, let’s update our `babel.config.js` file:

```js
module.exports = {
  presets: ['module:metro-react-native-babel-preset'],
  plugins: [
    [
      'react-native-reanimated/plugin',
      {
        globals: ['__scanCodes'],
      },
    ],
  ],
};
```

Cool, now we’re ready to code!

## Creating the QR scanner screen

When you create a React Native project with a source codebase, you will have an app screen with default UI components. So, I’m going to update our `App.js` file to work with VisionCamera:

```js
export default function App() {
  return (
    <View style={styles.screen}>
      <SafeAreaView style={styles.saveArea}>
        <View style={styles.header}>
          <Text style={styles.headerText}>React Native Camera Libraries</Text>
        </View>
      </SafeAreaView>

      <View style={styles.caption}>
        <Text style={styles.captionText}>
          Welcome To React-Native-Vision-Camera Tutorial
        </Text>
      </View>
    </View>
  );
}
```

Next, to record videos or capture photos, we will use the camera’s physical or virtual devices. Let’s quickly go over both:

- Physical: The camera lens on a phone. All physical cameras have different characteristics and capabilities, like formats, frame rate, focal length, and more. In addition, some phones have multiple physical cameras
- Virtual: A combination of one or more physical camera devices

We can get all the camera devices with the `useCameraDevices` Hook. Let’s update `App.js` and add the following code:

```js
const devices = useCameraDevices();
const cameraDevice = devices.back;
```

Above, we have a list of physical device types. For a single physical camera device, this property is always an array of one element; we can also check whether this camera is back- or front-facing.

But before we go any further, we need to enable permissions for the microphone and camera. We can request the user to give your app permission to use the camera or microphone through the following functions:

```js
const cameraPermission = await Camera.requestCameraPermission();
const microphonePermission = await Camera.requestMicrophonePermission();
```

The permission request status can be:

- Authorized: Your app is authorized and has permission
- Denied: The user explicitly denied the permission request and your app does not have permission
Restricted (iOS only): Your app’s camera or microphone is restricted, so you can’t use them

Let’s modify our `App.js` file with the requested permission and camera device:

```js
export default function App() {
  const [cameraPermission, setCameraPermission] = useState();

  useEffect(() => {
    (async () => {
      const cameraPermissionStatus = await Camera.requestCameraPermission();
      setCameraPermission(cameraPermissionStatus);
    })();
  }, []);

  console.log(`Camera permission status: ${cameraPermission}`);

  const devices = useCameraDevices();
  const cameraDevice = devices.back;

  const renderDetectorContent = () => {
    if (cameraDevice && cameraPermission === 'authorized') {
      return (
        <Camera
          style={styles.camera}
          device={cameraDevice}
          isActive={true}
        />
      );
    }
    return <ActivityIndicator size="large" color="#1C6758" />;
  };

  return (
    <View style={styles.screen}>
      <SafeAreaView style={styles.saveArea}>
        <View style={styles.header}>
          <Text style={styles.headerText}>React Native Image Detector</Text>
        </View>
      </SafeAreaView>

      <View style={styles.caption}>
        <Text style={styles.captionText}>
          Welcome To React-Native-Vision-Camera Tutorial
        </Text>
      </View>

      {renderDetectorContent()}
    </View>
  );
}
```

In the above code, we received the camera permission inside the `useEffect` Hook and got the device’s rear camera. Then, we checked whether `cameraDevice` exists and whether camera permission is `authorized`, after which the app will show the camera component.

## Using frame processor plugins

If you want to scan or detect an image with a camera, you’ll need to use a frame processor. Frame processors are simple and powerful functions written in JavaScript to process camera frames. By using frame processors, we can:

- Build AI and facial recognition features
- Use TensorFlow, ML Kit, Apple Vision, or other libraries
- Enable real-time video chats using WebRTC
- Detect and label images
- Create Snapchat-like filters
- Use color filters with depth detection

When we need to satisfy specific use cases, frame processor plugins come in very handy. In this case, we’ll use vision-camera-code-scanner to scan the QR code through ML Kit. We already installed it in the previous section, so let’s use it now:

```js
const detectorResult = useSharedValue('');

const frameProcessor = useFrameProcessor(frame => {
  'worklet';
  const imageLabels = labelImage(frame);

  console.log('Image labels:', imageLabels);
  detectorResult.value = imageLabels[0]?.label;
}, []);
```

Here, we scanned the QR code and assigned barcode strings with a React Native Reanimated Shared Value. Awesome — that’s it!

This is the final app:


<p align="center">
  <img src="https://github.com/Gapur/react-native-vision-camera-examples/blob/main/assets/example4.gif" />
</p>

## React Native camera library alternatives

It’s worth noting that there are some other alternatives to VisionCamera, so I’ve taken the liberty of including a quick list below. As with any tool, whatever works best for you and your use case is the one you should choose!

- [React Native Camera Kit](https://github.com/teslamotors/react-native-camera-kit): A robust, high-performance, lightweight, and easy-to-use camera library for React Native with 1.9K stars on GitHub at the time of writing. It’s worth noting that it doesn’t support video, so if that’s crucial for your use case, you’ll want to find an alternative
- [React Native Video](https://github.com/react-native-video/react-native-video): A <Video/> component for React Native with 6.4K stars on GitHub at the time of writing. In contrast to Camera Kit, it only supports videos, so keep that in mind
- [React Native Image Picker](https://github.com/react-native-image-picker/react-native-image-picker): This library allows you to select a photo/video from a device library or camera using its native UI. It has 7.9K stars on GitHub at the time of writing

# Conclusion

React Native Camera was the perfect option if your app required access to the device camera to take photos or record videos. Unfortunately, because it was deprecated due to a lack of maintainers and increased code complexity, we’ve had to move on to other options.

If the camera plays a big role in your application and you need full control over the camera, then React Native VisionCamera is probably the choice for you. React Native Image Picker may be mature and popular, but it has limited camera capabilities.

Thanks for reading. I hope you found this piece useful. Happy coding!

## Article on LogRocket

[Using React Native VisionCamera: Demo and alternatives](https://blog.logrocket.com/react-native-visioncamera-demo-alternatives/)

## How to contribute?

1. Fork this repo
2. Clone your fork
3. Code 🤓
4. Test your changes
5. Submit a PR!
