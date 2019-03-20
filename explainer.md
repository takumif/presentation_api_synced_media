# Synced media in Presentation API

## Problem

We have two web standards,
[Presentation API](https://w3c.github.io/presentation-api/) and
[Remote Playback API](https://w3c.github.io/remote-playback/), for showing media
contents on a remote device, each with distinct strengths.

Benefits of Presentation API (for developers of media web apps):
* Control over receiver-side HTML/CSS and JavaScript that allows:
  * Flinging MSE and EME videos
  * Changing the look-and-feel of the media player
  * Showing a splash screen
  * Drawing custom subtitles, ads, etc. over a video element
* Bi-directional communication between the controller and the receiver that
  allows, among other things:
  * Custom commands such as queueing videos and toggling settings
  * Multiple controllers controlling a receiver

Benefits of Remote Playback API (for developers of media web apps):
* Automatic synchronization of playback state (seek, volume change, play/pause,
  etc.) between controller and receiver media elements.

Currently, a developer cannot use both the APIs at the same time, and must
choose between the two. Remote Playback API works fine for simple synchronized
media playback, but if the developer wants to fling MSE/EME videos or customize
the UI shown on the receiver page, they must use Presentation API instead and
implement media playback control from scratch using `PresentationConnection`
messages.

## Proposal

We propose to extend the `PresentationConnection` and the `PresentationReceiver`
interfaces in Presentation API.

### Controller side
```
partial interface PresentationConnection {
  attribute HTMLMediaElement? controllingMedia;
};
```

When an `HTMLMediaElement` is set as `controllingMedia`, its `remote` attribute
(of the
[RemotePlayback interface](https://w3c.github.io/remote-playback/#remoteplayback-interface))
will have the following events fired:
* `connecting` fired when the element is set as `controllingMedia`
* `connect` fired when `controlledMedia` on the receiver side is also set
* `disconnect` fired when one of the following happens:
  * `controllingMedia` is unassigned or reassigned
  * `controlledMedia` is unassigned or reassigned
  * The presentation connection is closed or terminated

`controllingMedia.remote.state` will reflect these state changes.

### Receiver side
```
partial interface PresentationReceiver {
  attribute HTMLMediaElement? controlledMedia;
};
```

When an `HTMLMediaElement` is set as `controlledMedia`, its `remote` attribute
will have the following events fired:
* `connecting` fired when the element is set as `controlledMedia`
* `connect` fired when `controllingMedia` on the controller side is also set
* `disconnect` fired when one of the following happens:
  * `controllingMedia` is unassigned or reassigned
  * `controlledMedia` is unassigned or reassigned
  * The presentation connection is closed or terminated

`controlledMedia.remote.state` will reflect these state changes.

### Synchronized media playback
Once `controllingMedia` and `controlledMedia` are set, the playback states of
the two media elements are synchronized until one of the `disconnect` events
listed above occurs. The implementation of the sync is specific to the user
agent.

## Sample code

Controller page:
```html
<video id="sender-video" src="https://example.com/file.mp4"></video>

<script>
  const request = new PresentationRequest('https://example.com/receiver.html');
  const connection = await request.start();
  const videoElement = document.querySelector('#sender-video');
  connection.controllingMedia = videoElement;
  // |videoElement.remote.state| is now 'connecting'.

  // Event handlers can be set to be called once both sides are in sync or get
  // disconnected:
  videoElement.remote.onconnected = () => {
    console.log('connected, playback sync is on.');
    // Pausing the controller-side element now pauses the receiver-side
    // element as well.
    videoElement.pause();
  };
  videoElement.remote.ondisconnected = () => {
    console.log('disconnected, sync is off.');
  };
</script>
```

Receiver page:
```html
<video id="receiver-video" src="https://example.com/file.mp4"></video>

<script>
  const remoteVideoElement = document.querySelector('#receiver-video');
  navigator.presentation.receiver.controlledMedia = remoteVideoElement;
  // |remoteVideoElement.remote.state| is now 'connecting'.

  remoteVideoElement.remote.onconnected = () => {
    console.log('connected, playback sync is on.');
  };
</script>
```
