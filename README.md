# Meeting-Transcription-System-Extension-for-Popular-Video-Conferencing-Platforms
create a transcription system/app/chrome extension that accurately captures and transcribes meetings conducted on Google Meet, Zoom, and Microsoft Teams.

Can be something like Otter AI or Sembly where there is an agent that joins the call.
It can also be a Chrome Extension that uses transcriptions from the calls like TactiQ does. Here is the chrome extension approach MVP: https://github.com/thepersonalaicompany/amurex

Heck, it can even be a computer app for Win and macOS if you prefer that.

All I need is to get meeting transcriptions from Meet, Zoom and Teams to a specific API endpoint so it saves all conversations of a user in my database.
-----
To create a transcription system that can accurately capture and transcribe meetings conducted on Google Meet, Zoom, and Microsoft Teams, there are several approaches to consider. However, the most practical solution involves leveraging APIs and real-time transcription systems. Here’s an MVP (Minimum Viable Product) approach using a Chrome Extension to capture transcriptions and send them to a backend API.

The system will:

    Integrate with Google Meet, Zoom, and Microsoft Teams.
    Capture audio from the meeting in real-time.
    Transcribe the captured audio using a service like Google Cloud Speech-to-Text, Azure, or a custom transcription service.
    Send the transcriptions to a specific API endpoint.

Overview of the Architecture:

    Chrome Extension: Capture audio from Google Meet, Zoom, and Teams meetings and send it to the backend API.
    Backend API: Accept and save transcriptions in the database.
    Speech-to-Text: Use APIs like Google Cloud Speech-to-Text or Azure Speech API to convert the audio into text.

Approach

We'll build a Chrome Extension that:

    Detects when a meeting is active in Google Meet, Zoom, or Microsoft Teams.
    Captures the meeting audio using the browser’s capabilities (or an external agent for more control).
    Sends the captured audio to an API endpoint for transcription.
    Displays the transcription within the extension or stores it in the database.

Key Components:

    Chrome Extension: This will interact with the meeting platforms and record audio.
    Backend API: To handle storing the transcription data.
    Speech-to-Text API: For real-time audio transcription (Google, Azure, etc.).

Here’s how you can implement it:
1. Chrome Extension (Frontend)
manifest.json

{
  "manifest_version": 3,
  "name": "Meeting Transcription System",
  "version": "1.0",
  "description": "A transcription system for Google Meet, Zoom, and Microsoft Teams.",
  "permissions": [
    "activeTab",
    "storage",
    "microphone",
    "tabs"
  ],
  "background": {
    "service_worker": "background.js"
  },
  "content_scripts": [
    {
      "matches": [
        "https://meet.google.com/*",
        "https://zoom.us/*",
        "https://teams.microsoft.com/*"
      ],
      "js": ["content.js"]
    }
  ],
  "host_permissions": [
    "http://*/*",
    "https://*/*"
  ],
  "action": {
    "default_popup": "popup.html"
  },
  "icons": {
    "16": "icons/icon16.png",
    "48": "icons/icon48.png",
    "128": "icons/icon128.png"
  }
}

2. Background Script (background.js)

In the background script, we'll manage the recording of audio streams (either by using the Web Audio API or by utilizing a microphone capture).

chrome.runtime.onInstalled.addListener(() => {
  console.log("Meeting Transcription Extension Installed.");
});

chrome.runtime.onMessage.addListener((request, sender, sendResponse) => {
  if (request.action === "startRecording") {
    startRecording(request.meetingType, request.apiEndpoint);
    sendResponse({ status: "Recording started" });
  }
});

// Function to start recording
function startRecording(meetingType, apiEndpoint) {
  // You would use the Web Audio API or MediaRecorder to capture the audio
  navigator.mediaDevices.getUserMedia({ audio: true })
    .then(stream => {
      const mediaRecorder = new MediaRecorder(stream);
      mediaRecorder.start();

      mediaRecorder.ondataavailable = (event) => {
        // Send audio data to API for transcription
        sendAudioToAPI(event.data, apiEndpoint);
      };

      mediaRecorder.onstop = () => {
        console.log("Recording stopped");
        stream.getTracks().forEach(track => track.stop());
      };
    })
    .catch(error => {
      console.error("Error accessing microphone:", error);
    });
}

// Function to send audio to backend API for transcription
function sendAudioToAPI(audioBlob, apiEndpoint) {
  const formData = new FormData();
  formData.append("audio", audioBlob, "audio.webm");

  fetch(apiEndpoint, {
    method: 'POST',
    body: formData
  })
  .then(response => response.json())
  .then(data => {
    console.log("Transcription data:", data);
  })
  .catch(error => {
    console.error("Error sending audio to API:", error);
  });
}

3. Content Script (content.js)

This content script will detect if the meeting is ongoing and interact with the meeting UI for starting/stopping the recording. For now, it will only detect Google Meet, Zoom, and Microsoft Teams.

window.onload = function() {
  if (window.location.hostname === "meet.google.com") {
    startTranscription("Google Meet");
  } else if (window.location.hostname === "zoom.us") {
    startTranscription("Zoom");
  } else if (window.location.hostname === "teams.microsoft.com") {
    startTranscription("Microsoft Teams");
  }
};

function startTranscription(meetingType) {
  chrome.runtime.sendMessage({
    action: "startRecording",
    meetingType: meetingType,
    apiEndpoint: "https://your-backend-api.com/transcribe"
  }, (response) => {
    console.log(response.status);
  });
}

4. Popup HTML (popup.html)

Create a simple popup interface for the user to start and stop the transcription process.

<!DOCTYPE html>
<html>
  <head>
    <title>Meeting Transcription</title>
    <style>
      body {
        font-family: Arial, sans-serif;
        padding: 10px;
      }
      button {
        background-color: #25D366;
        color: white;
        padding: 10px;
        border: none;
        cursor: pointer;
        width: 100%;
      }
    </style>
  </head>
  <body>
    <h3>Transcribe Meeting</h3>
    <button id="startButton">Start Transcription</button>
    <button id="stopButton">Stop Transcription</button>

    <script>
      document.getElementById("startButton").addEventListener("click", () => {
        chrome.runtime.sendMessage({ action: "startRecording" }, (response) => {
          alert("Recording started");
        });
      });

      document.getElementById("stopButton").addEventListener("click", () => {
        chrome.runtime.sendMessage({ action: "stopRecording" }, (response) => {
          alert("Recording stopped");
        });
      });
    </script>
  </body>
</html>

5. Backend API for Transcription

You can use Google Cloud Speech-to-Text, Azure Speech API, or a custom transcription service to handle the audio files uploaded from the extension.
Node.js Example for API Endpoint:

const express = require("express");
const multer = require("multer");
const path = require("path");
const fs = require("fs");
const { SpeechClient } = require('@google-cloud/speech'); // Example for Google Cloud Speech

const app = express();
const port = 3000;

const upload = multer({ dest: "uploads/" });

const speechClient = new SpeechClient();

app.post("/transcribe", upload.single("audio"), async (req, res) => {
  const filePath = path.join(__dirname, req.file.path);
  
  const audioBytes = fs.readFileSync(filePath).toString('base64');
  const audio = {
    content: audioBytes,
  };

  const config = {
    encoding: 'WEBM_OPUS',
    sampleRateHertz: 16000,
    languageCode: 'en-US',
  };

  const request = {
    audio: audio,
    config: config,
  };

  try {
    const [response] = await speechClient.recognize(request);
    const transcription = response.results.map(result => result.alternatives[0].transcript).join("\n");
    res.json({ transcription: transcription });
  } catch (error) {
    console.error(error);
    res.status(500).send("Error transcribing audio");
  }

  // Delete the uploaded file after processing
  fs.unlinkSync(filePath);
});

app.listen(port, () => {
  console.log(`Server is running at http://localhost:${port}`);
});

Conclusion:

    Chrome Extension captures meeting audio via the browser.
    Backend API transcribes the audio using a speech-to-text API (Google, Azure, etc.).
    Database stores the transcriptions for later use.

Future Enhancements:

    Real-Time Transcription: Use WebSocket or Streaming APIs for live transcription.
    Multiple Language Support: Add support for multi-language transcription.
    User Interface: Display transcriptions directly within the Chrome extension.
    Privacy and Security: Implement encryption for storing sensitive data.

