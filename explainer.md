# Document Picture-in-Picture Explained

2022-01-13

## What's all this then?

There currently exists a Web API for putting an HTMLVideoElement into a
Picture-in-Picture window (HTMLVideoElement.requestPictureInPicture()). This
limits a website's ability to provide a custom picture-in-picture experience. We
want to expand upon that functionality with a new method on the Window object
(window.requestPictureInPictureWindow()) which opens a picture-in-picture (i.e.
always-on-top) window with a blank document that can be populated with arbitrary
HTMLElements instead of only a single HTMLVideoElement.

This new window will be much like a blank same-origin window opened via the
existing window.open() API, with some limitations for security reasons:

- The PiP window will never outlive the opening window
- The PiP window can only be opened as a response to a user gesture
- The website cannot set the position of the PiP window
- The PiP window cannot be navigated (any window.history or window.location
  calls that change to a new document will close the PiP window)
- The PiP window cannot open more windows
- The PiP window cannot download files
- The PiP window must be populated via JS (i.e. cannot be loaded via URL)
- The UA can restrict the size of the PiP window
- The UA can restrict input on the window

### Proposed Spec

    // We will add a new |requestPictureInPictureWindow()| method that opens an
    // always-on-top window with the given aspect ratio (subject to restrictions
    // by the UA) and an option to force the window to keep its original aspect
    // ratio when resizing.
    // We will also add a method to exit the PictureInPicture window as well as
    // events for detecting when picture-in-picture is entered or exited.
    partial interface Window {
      Promise<PictureInPictureWindow> requestPictureInPictureWindow(
          PictureInPicturWindowOptions options);
      attribute EventHandler onenterpictureinpicture;
      attribute EventHandler onexitpictureinpicture;
    };

    dictionary PictureInPictureWindowOptions {
      long width;
      long height;
      boolean constrainAspectRatio = false;
    };

    // The current PictureInPictureWindow object has no document accessor (since
    // it wasn't a window with a Document before), so we will add a document
    // accessor so the website can use it to populate the PiP window. We will
    // also add the aspect ratio options so they can be updated by the website
    // if e.g. a new video is loaded with a different aspect ratio.
    partial interface PictureInPictureWindow {
      readonly attribute Document? Document;
      Promise<void> setAspectRatio(float aspectRatio);
      boolean constrainAspectRatio;
    };


### Goals

- Allow a website to display arbitrary HTMLElements in an always-on-top window
- To be simple for web developers to use and understand. Note that while
  allowing websites to call requestPictureInPicture on any element would be the
  simplest way, for reasons described below, this isn't feasible.

### Non-goals

- This API is not attempting to handle placeholder content for elements that are
  moved out of the page (that is the responsibility of the website to handle)
- Allowing websites to open always-on-top widgets that outlive the webpage (the
  PiP window will close when the webpage is closed)

## Example code

### HTML

    <body>
      <div id="player-container">
        <div id="player">
          <video id="video" src="foo.webm"></video>
          <!-- more player elements here -->
        </div>
      </div>
      <input type="button" onclick="enterPiP();" value="Enter PiP" />
    </body>


### JavaScript

    // Handle to the picture-in-picture window.
    let pipWindow = null;

    function enterPiP() {
      const player = document.getElementById('player');
      let pipOptions = { width: player.clientWidth, height: player.clientHeight,
                         constrainAspectRatio: true };
      window.requestPictureInPictureWindow(pipOptions).then(pipWin => {
        pipWindow = pipWin;

        // Style remaining container to imply the player is in PiP.
        playerContainer.classList.add('pip-mode');

        // Add styles to the PiP window.
        const styleLink = document.createElement('link');
        styleLink.href = 'pip.css';
        styleLink.rel = 'stylesheet';
        styleLink.type = 'text/css';
        pipWindow.document.body.appendChild(styleLink);

        // Add player to the PiP window.
        pipWindow.document.body.appendChild(player);

        // Listen for the PiP closing event to put the video back.
        window.addEventListener('exitpictureinpicture', onExitPiP,
            { once: true });
      });
    }

    // Called when the PiP window has closed.
    function onExitPiP(event) {
        // Remove pip styling from the container.
        const playerContainer = document.getElementById('player-container');
        playerContainer.classList.remove('pip-mode');

        // Add the player back to the main window.
        const player = event.pictureInPictureWindow.document
                            .getElementById('player');
        playerContainer.appendChild(player);

        pipWindow = null;
    }

## Key scenarios

### Accessing elements on the PiP window

The document attribute provides access to the DOM of the PictureInPictureWindow object:

    const video = pipWindow.document.getElementById('video');
    video.loop = true;


### Listening to events on the PiP window

As part of creating an improved picture-in-picture experience, websites will
often want custom buttons and controls that need to respond to user input events
such as clicks.

    const video = pipWindow.document.getElementById('video');
    const muteButton = pipWindow.document.createElement('button');
    muteButton.innerHTML = 'toggle mute';
    muteButton.addEventListener('click', _ => { video.muted = !video.muted; });
    pipWindow.document.body.appendChild(muteButton);


### Exiting PiP

The website may decide to close the PictureInPictureWindow themselves. The
existing HTMLVideoElement PiP functionality uses the exitPictureInPicture method
on the Document object to accomplish this. For consistency, we will still use
this method to close the PiP window:

    // This will close the PiP window and trigger our existing onExitPiP()
    // listener.
    document.exitPictureInPicture();


### Changing Aspect Ratio

Sometimes the website will want to change the aspect ratio after the PiP window
is open (e.g. because a new video is playing with a different aspect ratio). The
website can change it via the aspectRatio attribute on the
PictureInPictureWindow:

    const newVideo = document.createElement('video');
    newVideo.id='video';
    newVideo.src = 'newvideo.webm';
    newVideo.addEventListener('loadedmetadata', async _ => {
      const aspectRatio = newVideo.videoWidth / newVideo.videoHeight;
      const player = pipWindow.document.getElementById('player');
      const oldVideo  = pipWindow.document.getElementById('video');
      player.removeChild(oldVideo);
      player.appendChild(newVideo);
      await pipWindow.setAspectRatio(aspectRatio);
    });
    newVideo.load();


### Getting elements out of the PiP window when it closes

When the PiP window is closed for any reason (either because the website
initiated it or the user closed it), the website will often want to get the
elements back out of the PiP window. The website can perform this in an event
handler for the onexitpictureinpicture event on the Window object. The UA must
guarantee that the PiP document will live long enough for the event handlers to
be run. This is shown in the onExitPiP() handler in the "Example code" section
above and is copied below:

    // Called when the PiP window has closed.
    function onExitPiP(event) {
        // Remove pip styling from the container.
        const playerContainer = document.getElementById('player-container');
        playerContainer.classList.remove('pip-mode');

        // Add the player back to the main window.
        const player = event.pictureInPictureWindow.document.getElementById('player');
        playerContainer.appendChild(player);
    }

## Detailed design discussion

### Why not extend the HTMLVideoElement.requestPictureInPicture() idea to allow it to be called on any HTMLElement?

Any API where the UA is taking elements out of the page and then reinserting
them ends up with tricky questions on what to show in the current document when
those elements are gone (do elements shift around? Is there a placeholder? What
magic needs to happen when things resize? etc). By leaving it up to websites to
move their own elements, the API contract between the UA and website is much
clearer and simpler to understand.

### Since this is pretty close to window.open(), why not just add an "alwaysOnTop" flag to window.open()?

The main reason we decided to have a completely separate API is to make it
easier for websites to detect it (since in most cases, falling back to a
standard window would be undesirable and websites would rather use
HTMLVideoElement PiP instead). Additionally, it also works differently enough
from window.open (e.g. never outliving the opener) that having it separate
makes sense.

### Why not give the website more control over the size/position of the window?

Giving websites less control over the size/position of the window will help
prevent e.g. phishing attacks where a website pops a small always-on-top window
over an input element to steal your password.

## Considered alternatives

Surface Element was a proposal where the website would wrap PiP-able content in
advance with a new type of iframe-like element that could be pulled out into a
separate window when requested. This had some downsides including always
requiring the overhead of a separate document (even in the most common case of
never entering picture-in-picture).

We also considered a similar approach to the one in this document, but with no
input allowed in the DOM (only allowlisted controls from a predetermined list in
a similar fashion to the existing HTMLVideoElement PiP). One issue with this
approach is that it really didn't help websites do much more than they already
can today, since a website can draw anything in a canvas element and PiP a video
with the canvas as a source. Having HTMLElements that can actually be interacted
with is what makes the Document Picture-in-Picture feature worth implementing.

## References & acknowledgements

Many thanks to Frank Liberato, Mark Foltz, Klaus Weidner, François Beaufort,
Charlie Reis, and Joe DeBlasio for their comments and contributions to this
document and to the discussions that have informed it.
