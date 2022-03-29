<template>
  <f7-page name="home">
    <!-- Top Navbar -->
    <f7-navbar>
      <f7-nav-title>Microphone check</f7-nav-title>
    </f7-navbar>
    <!-- Page content-->
    <div class="content-container">
      <p>
        First approach - plays audio immediately only on phone (uses Cordova
        plugin audioInput):
      </p>
      <f7-button
        class="audio_btn"
        large
        small
        fill
        @click="listenAndRepeatAudio()"
        >Listen and repeat sound
      </f7-button>


      <p>
        Second approach - save audio and press play (uses MediaRecorder on PC
        and Cordova plugin audioInput on phone):
      </p>
      <f7-button large small fill @click="recordSound()"
        >Record and save sound
      </f7-button>
      <f7-button class="audio_btn" large small fill @click="playSound()"
        >Play sound</f7-button
      >

      <p>
        Third approach - plays audio immediately (uses
        navigator.mediaDevices.getUserMedia):
      </p>
      <f7-button large small fill @click="getUserMediaAudio()"
        >Record with getUserMedia
      </f7-button>
      <p>After starting recording, play the sound:</p>
      <audio class="audio_btn" id="player" controls></audio>

      <p>
        Fourth approach - plays audio immediately (uses cordova-plugin-media):
      </p>
      <f7-button large small fill @click="getAudioCordovaPluginMedia()"
        >Record with cordova-plugin-media
      </f7-button>
    </div>
  </f7-page>
</template>

<script>
export default {
  setup() {
    let audio;
    let fileSystem;

    const isAndroid = () => {
      const userAgent = navigator.userAgent || navigator.vendor || window.opera;
      if (/android/i.test(userAgent)) return true;
      return false;
    };

    const isIos = () => {
      const userAgent = navigator.userAgent || navigator.vendor || window.opera;
      if (/iPad|iPhone|iPod/i.test(userAgent)) return true;
      return false;
    };

    const isMobile = () => window.cordova && (isAndroid() || isIos());

    // This should capture voice and play it instantly through speakers
    const listenAndRepeatAudio = () => {
      audioinput.start({
        streamToWebAudio: true,
      });

      console.log(audioinput.getAudioContext().destination);
      console.log(audioinput.getAudioContext());
      console.log(JSON.stringify(audioinput.getAudioContext()));
      console.log(JSON.stringify(audioinput.getAudioContext().destination));

      // Connect the audioinput to the device speakers in order to hear the captured sound.
      audioinput.connect(audioinput.getAudioContext().destination);

      setTimeout(() => {
        audioinput.stop(() => {
          console.log("stopped");
        });
      }, 5000);
    };

    const recordSound = () => {
      if (isMobile() === true) {
        // Get access to the file system
        window.requestFileSystem(
          window.TEMPORARY,
          5 * 1024 * 1024,
          function (fs) {
            console.log("Got file system: " + fs.name);
            console.log("Got file system: " + JSON.stringify(fs));
            console.log("Got file system: " + fs);
            fileSystem = fs;

            // Now you can initialize audio, telling it about the file system you want to use.
            var captureCfg = {
              sampleRate: 16000,
              bufferSize: 8192,
              channels: 1,
              format: audioinput.FORMAT.PCM_16BIT,
              audioSourceType: audioinput.AUDIOSOURCE_TYPE.DEFAULT,
              fileUrl: cordova.file.cacheDirectory,
            };

            // Initialize the audioinput plugin.
            window.audioinput.initialize(captureCfg, function () {
              // Now check whether we already have permission to access the microphone.
              window.audioinput.checkMicrophonePermission(function (
                hasPermission
              ) {
                if (hasPermission) {
                  console.log("Already have permission to record.");
                } else {
                  // Ask the user for permission to access the microphone
                  window.audioinput.getMicrophonePermission(function (
                    hasPermission,
                    message
                  ) {
                    if (hasPermission) {
                      console.log("User granted permission to record.");
                    } else {
                      console.warn("User denied permission to record.");
                    }
                  });
                }
              });
            });
          },
          function (e) {
            console.log("Couldn't access file system: " + e.message);
          }
        );

        // Later, when we want to record to a file...
        var captureCfg = {
          fileUrl: cordova.file.cacheDirectory + "temp.wav",
        };

        console.log(captureCfg.fileUrl);

        // Start the capture.
        audioinput.start(captureCfg);

        setTimeout(() => {
          // ...and when we're ready to stop recording.
          audioinput.stop(function (url) {
            console.log(url);
            console.log(JSON.stringify(url));

            // Now you have the URL (which might be different to the one passed in to audioinput.start())
            // You might, for example, read the data into a blob.
            window.resolveLocalFileSystemURL(
              captureCfg.fileUrl,
              function (tempFile) {
                tempFile.file(function (tempWav) {
                  var reader = new FileReader();
                  reader.onloadend = function (e) {
                    // Create the blob from the result.
                    var blob = new Blob([new Uint8Array(this.result)], {
                      type: "audio/wav",
                    });
                    // Delete the temporary file.
                    tempFile.remove(function (e) {
                      console.log("temporary WAV deleted");
                    }, fileError);
                    // Do something with the blob.
                    console.log(blob);
                    doSomethingWithWAVData(blob);
                  };
                  reader.readAsArrayBuffer(tempWav);
                });
              },
              function (e) {
                console.log(
                  "Could not resolveLocalFileSystemURL: " + JSON.stringify(e)
                );
              }
            );
          });
        }, 5000);
      } else {
        navigator.mediaDevices.getUserMedia({ audio: true }).then((stream) => {
          const mediaRecorder = new MediaRecorder(stream);
          mediaRecorder.start();

          const audioChunks = [];

          mediaRecorder.addEventListener("dataavailable", (event) => {
            audioChunks.push(event.data);
          });

          mediaRecorder.addEventListener("stop", () => {
            const audioBlob = new Blob(audioChunks);
            const audioUrl = URL.createObjectURL(audioBlob);
            audio = new Audio(audioUrl);
          });

          setTimeout(() => {
            mediaRecorder.stop();
            console.log(audioChunks);
          }, 5000);
        });
      }
    };

    const playSound = () => {
      if (isMobile() == true) {
        console.log("play mobile");
      } else {
        try {
          audio.play();
        } catch (error) {
          console.warn(
            "Something went wrong, probably audio file has not been saved yet"
          );
          console.warn("Error: " + error);
        }
      }
    };

    // Capturing sound by navigator.mediaDevices.getUserMedia
    const getUserMediaAudio = () => {
      const handleSuccess = function (stream) {
        const audioTracks = stream.getAudioTracks();
        console.log("Using audio device: " + audioTracks[0].label);
        stream.oninactive = function () {
          console.log("Stream ended");
        };

        if (window.URL) {
          player.srcObject = stream;
        } else {
          player.src = stream;
        }
      };

      navigator.mediaDevices
        .getUserMedia({ audio: true, video: false })
        .then(handleSuccess)
        .catch((e) => {
          console.log(e.message);
          console.log(JSON.stringify(e));
        });
    };

    const getAudioCordovaPluginMedia = () => {
      let src = "myrecording.mp3";
      var mediaRec = new Media(
        src,
        // success callback
        function () {
          console.log("recordAudio():Audio Success");
        },

        // error callback
        function (err) {
          console.log("recordAudio():Audio Error: " + err.code);
        }
      );

      // Record audio
      mediaRec.startRecord();

      // Stop recording after 10 seconds
      setTimeout(function () {
        mediaRec.stopRecord();
        mediaRec.play();
      }, 5000);
    };

    return {
      recordSound,
      listenAndRepeatAudio,
      playSound,
      getUserMediaAudio,
      getAudioCordovaPluginMedia,
    };
  },
};
</script>
