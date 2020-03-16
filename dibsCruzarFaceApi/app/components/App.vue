<template>
<Page class="page" actionBarHidden="true">
  <Actionbar title="FaceDetection" class="actionBar"/>
  <StackLayout>
    <Label class="data" text="My Mood"></Label>
    <ScrollView class="card">
      <StackLayout>
        <Label
          horizontalAlignment="center"
          textWrap="true"
          class="data"
        >Take a Selfie and Match your Mood </Label>
        <StackLayout row="1">
          <Button class="button" text="Take a Pic" @tap="runFaceDetect" />
          <Image v-if = selfie class="cameraPic" :src="selfie"></Image>
          <Image v-if = imageAsset class="cameraPic" :src="imageAsset"></Image>
          <Label v-for="emotion in emotions" :key="emotion">{{ emotion }}</Label>
        </StackLayout>
        <StackLayout v-if="!complete">
          <ActivityIndicator row="1" #activityIndicator busy="true" width="30" height="30"></ActivityIndicator>
        </StackLayout>
       <!-- <StackLayout class="inner-card" v-if="selfiePoem">
          <Label textWrap="true" :text="selfiePoem" />
        </StackLayout>-->
      </StackLayout>
    </ScrollView>
  </StackLayout>
</Page>
</template>
<script>

import * as camera from "nativescript-camera";
import { ImageAsset } from "tns-core-modules/image-asset/image-asset";
import { ImageSource } from "tns-core-modules/image-source";
import { File, knownFolders, path } from "tns-core-modules/file-system";
import { isAndroid } from "tns-core-modules/platform";
import { session } from "nativescript-background-http";

const http = require("http");
export default {
  data() {
    return {
      imageAsset:null,
      selfie: null,
      result: [],
      emotions: [],
      myEmotion: 0.5,
      complete: true
    };
  },
   methods: {
   // ...mapActions(["getSelfiePoem", "clearSelfiePoem"]),
    runFaceDetect() {
      camera.requestPermissions();
     // this.clearSelfiePoem();
      this.complete = false;
      const imageAsset =  camera.takePicture({
        width: 300,
        height: 500,
        keepAspectRatio: true,
        saveToGallery: false,
        cameraFacing: "front"
      });
      //process the asset
      const filePath = this.getFilePath(imageAsset);
      const imageFile = File.fromPath(filePath);
      this.selfie = imageAsset;
      //send it to be processed
      this.result =  this.http.sendRequest(imageFile);
    },

    getFilePath(asset) {
      if (isAndroid) {
        return asset.android;
      }

      const source =  new ImageSource().fromAsset(asset);

      const folder = knownFolders.documents().path;
      const fileName = `${Date.now()}.png`;
      const filePath = path.join(folder, fileName);
      const saved = source.saveToFile(filePath, "png");
      if (saved) {
        console.log("Image ready for upload!");
      }

      return filePath;
    },

    determineScore(score) {
      const myScore = score.split("-");
      const mood = myScore[1].trim();
      switch (mood) {
        case "sadness":
          this.myEmotion = 0.1;
          break;
        case "anger":
          this.myEmotion = 0.2;
          break;
        case "contempt":
          this.myEmotion = 0.3;
          break;
        case "disgust":
          this.myEmotion = 0.3;
          break;
        case "fear":
          this.myEmotion = 0.4;
          break;
        case "neutral":
          this.myEmotion = 0.5;
          break;
        case "happiness":
          this.myEmotion = 0.7;
          break;
        case "surprise":
          this.myEmotion = 0.8;
          break;
        default:
          this.myEmotion = 0.5;
          break;
      }
     this.myEmotion = emotions;
      this.complete = true;
    },

    sendRequest(file) {
      return new Promise((resolve, reject) => {
        const ses = session("image-upload");

        const request = {
          url:
            "https://dibsface.cognitiveservices.azure.com/face/v1.0/detect?returnFaceLandmarks=false&returnFaceAttributes=emotion",
          method: "POST",
          headers: {
            "Content-Type": "application/octet-stream",
            "Ocp-Apim-Subscription-Key": "4aa538d8109a4122bb1e8351a1640ccf"
          },
          description: "Uploading " + file.name
        };

        const task = ses.uploadFile(file.path, request);

        task.on("error", e => reject("error " + e.responseCode + " code."));

        task.on("responded", e => {
          //clear the array
          this.emotions = [];
          if (e.data !== null) {
            const data = JSON.parse(e.data);
            const emotion = data.faceAttributes.emotion;

            for (var i in emotion) this.emotions.push(emotion[i] + " - " + i);

            this.emotions.sort(function(a, b) {
              return parseFloat(b) - parseFloat(a);
            });
            //grab the top element and equate it to a poem score
            this.determineScore(this.emotions[0]);
          } else {
            alert("Sorry, there was a problem");
          }
        });
      });
    }
  }
};
</script>

<style>
.cameraPic {
  border-radius: 5;
  margin: 10;
}

    .page {
        background-image: linear-gradient(to right, #4D7C8A, #7F9C96);
    }

    .actionBar {
        background-color: #1B4079;
        color:
            #ffffff;
    }
    .data {
        border-radius: 15px;
        font-size: 22;
        font-weight: bold;
        text-align: center;
    }
     .button {
        margin-bottom: 50px;
    }

</style>