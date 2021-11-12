<script setup>
  import { ref, onMounted, reactive } from 'vue';
  const AWS = require('aws-sdk');
  import * as Chime from 'aws-sdk/clients/chime';
  import {
    ConsoleLogger,
    DefaultDeviceController,
    DefaultMeetingSession,
    LogLevel,
    MeetingSessionConfiguration
} from 'amazon-chime-sdk-js';
  
  const { v4: uuid } = require('uuid');

  AWS.config.update({
    accessKeyId: "AKIAQNS5IHH2VLCVD4OS",
    secretAccessKey: "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",
    "region": "us-west-2"
  }); // for simplicity. In prod, use loadConfigFromFile, or env variables

  // You must use "us-east-1" as the region for Chime API and set the endpoint.
  const chime = new Chime({ region: 'us-east-1' });
  chime.endpoint = new AWS.Endpoint('https://service.chime.aws.amazon.com');

  const meetingObj = null;
  let attendeeObj = null;
  let meetingSession = null;
  const attendeesPresent =  reactive(new Set());

  const logger = new ConsoleLogger('MyLogger', LogLevel.INFO);
  const deviceController = new DefaultDeviceController(logger);

  let videoDevices = [];
  let audioDevices = [];

  const createMeeting = async () => {
    console.log('Creating Meeting');
    meetingOngoing.value = true;

    meetingObj.value = await chime.createMeeting({
      ClientRequestToken: uuid(),
      MediaRegion: 'us-west-2' // Specify the region in which to create the meeting.
    }).promise();

  };

  const joinMeeting = async () => {
    console.log('Joining meeting');

    const mainVideo = document.getElementById('main-video');
    const localVideo = document.getElementById('local-video');
    const audio = document.getElementById('meeting-audio');

    attendeeObj = await chime.createAttendee({
      MeetingId: meetingObj.Meeting.MeetingId,
      ExternalUserId: uuid() // Link the attendee to an identity managed by your application.
    }).promise();

    // // set the selected audio and video devices
    await deviceController.chooseVideoInputDevice(videoDevices[0].deviceId);
    await deviceController.listAudioInputDevices() // For some reason this required to be executed before the `chooseAudioInputDevice()`, or else an error is encountered
      .then(async () => {
        await deviceController.chooseAudioInputDevice(audioDevices[0].deviceId);
      });

    const configuration = new MeetingSessionConfiguration(meetingObj, attendeeObj);
    meetingSession = new DefaultMeetingSession(configuration, logger, deviceController);

    /* =======
      VIDEO
    ======= */

    // Handling for Audio/Video events, see https://aws.github.io/amazon-chime-sdk-js/interfaces/audiovideoobserver.html
    const videoObserver = {
      // videoTileDidUpdate is called whenever a new tile is created or tileState changes.
      videoTileDidUpdate: (tileState) => {
        console.log("%cVideoTileDidUpdate()", "color: green", tileState);
        // Ignore a tile without attendee ID and other attendee's tile.
        if (!tileState.boundAttendeeId && !tileState.localTile) {
          console.log("No Attendee Id");
          return;
        }
        // Checking whether to render as local preview or main video
        let videoObj = tileState.localTile ? localVideo : mainVideo;
        meetingSession.audioVideo.bindVideoElement(tileState.tileId, videoObj.el);
        videoObj.tileId = tileState.active ? tileState.tileId : null;
      },
      videoTileWasRemoved: (tileId) => { // This seems to give a more accurate callback to a video losing connection to the meeting
        console.log(`%cvideoTileWasRemoved. [tileId=${tileId}]`, "color: orange");
        // Set the corresponding video 'isActive' flag to false
        const videoObj = tileId === mainVideo.tileId ? mainVideo : localVideo;
        videoObj.isActive = false;
      },
    };
    const deviceChangeObserver = {
      // videoInputsChanged is called whenever a new webcam is connected to the computer
      videoInputsChanged: async (freshVideoInputDeviceList) => {
        // TODO Load the audio/video input devices to vuex store
        console.log('videoInputsChanged: ', freshVideoInputDeviceList);
      }
    };
    meetingSession.audioVideo.addDeviceChangeObserver(deviceChangeObserver);

    meetingSession.audioVideo.startLocalVideoTile();
    meetingSession.audioVideo.addObserver(videoObserver);

    /* =======
      AUDIO
    ======= */

    await meetingSession.audioVideo.bindAudioElement(audio);

    const audioObserver = {
      audioVideoDidStart: () => {
        console.log('AudioVideo Started');
      }
    };
    meetingSession.audioVideo.addObserver(audioObserver);

    // Handlers for meeting events, see https://aws.github.io/amazon-chime-sdk-js/modules/meetingevents.html#meeting-events
    const meetingObserver = {
      eventDidReceive: (name, attributes) => {
        console.log('Meeting event: ', name)
        switch (name) {
          case 'meetingStartSucceeded':
            // TODO callback when meeting has started, maybe enable/show the video controls at this point?
            console.log('Meeting started: ', attributes);
            break;
          case 'audioInputFailed':
            console.error(`Failed to choose microphone: ${attributes.audioInputErrorMessage} in `, attributes);
            break;
          case 'videoInputFailed':
            console.error(`Failed to choose camera: ${attributes.videoInputErrorMessage} in `, attributes);
            break;
          case 'meetingStartFailed':
            console.error(`Failed to start a meeting: ${attributes.meetingErrorMessage} in `, attributes);
            break;
          case 'meetingFailed':
            console.error(`Failed during a meeting: ${attributes.meetingErrorMessage} in `, attributes);
            break;
          case 'meetingStartRequested':

            break;
          default:
            break;
        }
        // Meeting is ended
        if (Object.prototype.hasOwnProperty.call(attributes, 'meetingStatus')) {
          // TODO handling for the meeting ending.
          console.log(`The meeting ended with the status: ${attributes.meetingStatus} in `, attributes);
        }
      }
    };
    meetingSession.audioVideo.addObserver(meetingObserver);

    // Subscribe to Attendees Joining/Leaving the meeting
    const attendeesCallback = (presentAttendeeId, present) => {
      console.log(`Attendee ID: ${presentAttendeeId} Present: ${present}`);
      // TODO handling for attendees joining/leaving
      if (present) { // An attendee joins the call
        attendeesPresent.add(presentAttendeeId);
      } else { // An attendee leaves the call
        attendeesPresent.delete(presentAttendeeId);
      }
    };
    meetingSession.audioVideo.realtimeSubscribeToAttendeeIdPresence(attendeesCallback);

    await meetingSession.audioVideo.start(); // Start the video meeting

  };

  const meetingOngoing = ref(false);

  onMounted(async () => {
    videoDevices = await deviceController.listVideoInputDevices();
    audioDevices = await deviceController.listAudioInputDevices();

    console.log(videoDevices);
    console.log(audioDevices);
  });

</script>

<template>
  <input type="button" @click="createMeeting" value="Create Video Call" :disabled="meetingOngoing"/>
  <br>
  <br>
  <div>Or</div>
  <br>
  <div>
    <label for="meeting-obj-input">Meeting Object:</label><br>
    <textarea id="meeting-obj-input" type="text" v-model="meetingObj"></textarea>
    <div>
      <input id="join-btn" type="button" value="Join" :disabled="meetingOngoing || !meetingObj" @click="joinMeeting">
    </div>
  </div>
  
  <div v-if="meetingObj">Meeting object: {{meetingObj}}</div>

  <div>
    <video id="main-video" />
    <video id="local-video" />
    <audio id="meeting-audio" />
  </div>
</template>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
  #main-video,
  #local-video {
    background-color: grey;
    margin: 10px;
  }

  #local-video {
    max-width: 200px;
  }

  input,
  textarea {
    margin: 0 5px;
  }
</style>
