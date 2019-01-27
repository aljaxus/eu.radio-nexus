<template>
  <v-app dark>
    <v-content>
      <v-container fluid fill-height>
        <v-layout align-center justify-center row wrap>
          <v-flex xs12 text-xs-center>
            <img src="./assets/img/fox.svg" alt="radio-nexus.eu logo" style="max-height:500px;max-width:100%;height:50vh;">
          </v-flex>
          <v-flex xs12 sm10 md8 lg6 text-xs-center>
            <v-flex xs12>
              <v-slider
                v-model="volumeValue"
                append-icon="volume_up"
                :prepend-icon="volumeValue<=50?'volume_mute':'volume_down'"
                @click:append="volumeValue = (volumeValue>=950 ? 1000 : volumeValue+50)"
                @click:prepend="volumeValue = (volumeValue<=50 ? 0 : volumeValue-50)"
                :max="1000"
              ></v-slider>
            </v-flex>
            <v-flex xs12>
              <v-card style="background-image:linear-gradient(110deg,rgba(56,54,54,0.76),rgba(74,74,74,0.68),rgba(68,68,68,0.94),rgba(74,73,73,0.52),rgba(51,51,51,0.81));background-color:unset;">
                <v-card-text>
                  <v-layout row wrap text-xs-left>
                    <v-flex v-if="!playing" xs12 class="text-xs-center">
                      <v-btn 
                        outline 
                        color="success" 
                        @click="playerStart"
                        :loading="playingLoading"
                        :disabled="playingLoading"
                      >
                        Start the player
                      </v-btn>
                    </v-flex>
                    <v-flex v-if="playing" xs2 lg1 class="text-xs-center">
                      <img :src="nowPlaying.art" alt="radio-nexus.eu logo" class="nowPlayingImage">
                    </v-flex>
                    <v-flex v-if="playing" xs10 lg11 class="pl-3">
                      <v-btn
                        absolute
                        dark
                        fab
                        small
                        right
                        outline
                        color="info"
                        @click="loadRequests(requestPage); request.showDialog=true;"
                      >
                        <v-icon>queue_music</v-icon>
                      </v-btn>
                      <h5 v-if="nowPlaying.title!=null && nowPlaying.title.length > 0" style="font-size:17px;font-weight:400;color:#fff5ee;">Title: {{ nowPlaying.title }}</h5>
                      <h5 v-if="nowPlaying.artist!=null && nowPlaying.artist.length > 0" style="font-size:13px;font-weight:300;color:#fff5ee;">Artist: {{ nowPlaying.artist }}</h5>
                      <h5 v-if="nowPlaying.edit_by!=null && nowPlaying.edit_by.length > 0" style="font-size:13px;font-weight:300;color:#fff5ee;">Edited by: {{ nowPlaying.edit_by }}</h5>
                      <h5 style="font-size:13px;font-weight:300;color:#fff5ee;">{{ elapsedStr + ' / ' + durationStr }}</h5>
                    </v-flex>
                    <v-flex v-if="playing" xs12>
                      <v-progress-linear
                        background-color="black"
                        background-opacity="0.2"
                        color="info"
                        height="4"
                        :value="(nowPlaying.elapsed / nowPlaying.duration) * 100"
                      ></v-progress-linear>
                    </v-flex>
                    <v-flex v-if="playing" xs12>
                      <v-switch
                        :label="`Stream type: ${streamHQ==true?'radio - 256kbps':'mobile - 128kbps'}`"
                        v-model="streamHQ"
                      ></v-switch>
                    </v-flex>

                    <audio 
                      id="audio" 
                      ref="audio"
                      type="audio/mpeg"
                      preload="auto"
                      :src="audioSrc"
                    >
                      <p>Your browser does not support the <code>audio</code> element.</p>
                    </audio>
                  </v-layout>
                </v-card-text>
              </v-card>
            </v-flex>
          </v-flex>
        </v-layout>
      </v-container>
      <v-snackbar
        v-model="snack.stillMuted"
        bottom
        left
        color="info"
      >
        Do not forget to unmute the volume.
        <v-btn
          color="info"
          flat
          :timeout="10000"
          @click="snack.stillMuted = false"
        >
          Close
        </v-btn>
      </v-snackbar>
      <v-snackbar
        v-model="snack.connErr"
        bottom
        :timeout="0"
        color="error"
      >
        Radio server could not be reached
      </v-snackbar>
      <v-snackbar
        v-model="request.submitSnack.shown"
        bottom
        :timeout="6000"
        :color="request.submitSnack.type"
      >
        {{ request.submitSnack.txt }}
      </v-snackbar>
      <v-dialog
        v-model="request.showDialog"
        width="840"
        dark
      >
        <v-card>
          <v-card-title
            class="headline"
          >
            Request a song
          </v-card-title>

          <v-card-text>
            <v-layout v-if="request.dataLoaded" row wrap>
              <v-flex v-for="row in request.data.rows" :key="row.request_id" xs12 class="my-1">
                <v-layout row wrap>
                  <v-flex md1 v-if="$vuetify.breakpoint.mdAndUp">
                    <img 
                      src="./assets/img/transparent-512px.png"
                      :style="'background-image:url('+row.song.art+')'"
                      alt="Request art"
                      class="requestImage"
                    >
                  </v-flex>
                  <v-flex xs10 sm10 md9 class="px-2">
                    <h5 v-if="row.song.title!=null && row.song.title.length > 0" style="font-size:14px;font-weight:400;color:#fff5ee;">Song: {{ row.song.title }}</h5>
                    <h5 v-if="row.song.artist!=null && row.song.artist.length > 0" style="font-size:13px;font-weight:300;color:#fff5ee;">By: {{ row.song.artist }}</h5>
                    <h5 v-if="$vuetify.breakpoint.mdAndUp && row.song.custom_fields.edit_by!=null && row.song.custom_fields.edit_by.length > 0" style="font-size:13px;font-weight:300;color:#fff5ee;">Edit: {{ row.song.custom_fields.edit_by }}</h5>
                  </v-flex>
                  <v-flex 
                    xs2 
                    sm2 
                    md2
                    class="text-xs-right"
                  >
                    <v-btn 
                    outline 
                    color="info" 
                    @click="requestSong(row.request_url)" 
                    :style="'margin-left:0;margin-right:0;'+($vuetify.breakpoint.xsOnly?'min-width:16px;':'')"
                  >
                      <v-icon v-if="!$vuetify.breakpoint.mdAndUp">arrow_forward</v-icon>
                      <span v-if="$vuetify.breakpoint.mdAndUp">Request</span>
                    </v-btn>
                  </v-flex>
                </v-layout>
              </v-flex>
              <v-flex xs12 class="text-xs-center">
                <v-pagination
                  v-model="requestPage"
                  :length="request.data.total_pages"
                ></v-pagination>
              </v-flex>
            </v-layout>
            <v-layout v-if="!request.dataLoaded" row wrap>
              <v-flex xs12>
                <v-progress-linear :indeterminate="true"></v-progress-linear>
              </v-flex>
            </v-layout>
          </v-card-text>

          <v-divider></v-divider>

          <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn
              color="info"
              flat
              @click="request.showDialog = false"
            >
              Close
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
    </v-content>
    <v-footer
      color="transparent"
      fixed
      app
    >
      <v-spacer></v-spacer>
      <div>
        <v-btn
          style="float:right;"
          dark
          small
          outline 
          color="purple accent-1"
          href="https://aljaxus.eu"
          target="_blank"
        >
          &copy; {{ new Date().getFullYear() }} - Aljaxus
        </v-btn>
      </div>
    </v-footer>
  </v-app>
</template>

<script>

/*

nowplaying card:

background-color: #ffffff3d;
background-image: linear-gradient(to right, #ffffff3d, #d8d8d83d, #6b6b6ba3, #d8d8d83d, #ffffff3d);


*/

// eslint-disable-next-line
const axios = require('axios');

export default {
  name: 'App',
  data() {
    return {
      volumeValue: 50,
      playing: false,
      playingLoading: false,
      nowPlaying: {
        title: 'some Title',
        artist: 'some Artist',
        edit_by: 'some Editor',
        art: './assets/img/art-placeholder.jpg',
        duration: 0,
        elapsed: 0
      },
      request: {
        data: {
          rows: {}
        },
        dataLoaded: false,
        submitSnack: {
          shown: false,
          type: 'info',
          txt: ''
        },
        showDialog: false
      },
      snack: {
        stillMuted: false,
        connErr: false
      },
      requestPage: 1,
      elapsedCounter: 0,
      streamHQ: true
    }
  },
  watch: {
    volume (val) {
      document.getElementById('audio').volume = val/100;
      this.snack.stillMuted = false;
    },
    requestPage (val) {
      this.loadRequests(val);
    },
    audioSrc (oldUrl, newUrl) {
      if (oldUrl != newUrl) {
        this.playing = false;
        this.playingLoading = true;
        try {
          document.getElementById('audio').load();
          document.getElementById('audio').addEventListener('canplaythrough', () => { 
            document.getElementById('audio').play();
            this.playing = true;
            this.playingLoading = false;
            this.loadNowPlaying();
          }, false);
        } catch (error) {
          // eslint-disable-next-line
          console.log(error);
        }
      }
    }
  },
  computed: {
    audioSrc () {
      // return 'https://radio.aljaxus.eu/radio/8000/'+(this.$vuetify.breakpoint.smAndDown?'mobile':'radio')+'.mp3';
      let type = this.streamHQ ? 'radio' : 'mobile';
      return 'https://radio.aljaxus.eu/radio/8000/'+type+'.mp3';
    },
    volume () {
      return this.volumeValue / 10;
    },
    elapsedStr () {
      return this.secToStr(this.nowPlaying.elapsed)
    },
    durationStr () {
      return this.secToStr(this.nowPlaying.duration)
    }
  },
  mounted() {
    document.getElementById('audio').volume = this.volume/100;
    setInterval(()=>{
      axios.get('https://radio.aljaxus.eu/api/station/1')
        .then(() => {
          this.snack.connErr = false;
        })
        .catch(() => {
          this.snack.connErr = true;
        });
    }, 15000);
  },
  methods: {
    playerStart () {
      this.loadNowPlaying();
      this.playingLoading = true;
      document.getElementById('audio').load();
      document.getElementById('audio').addEventListener('canplaythrough', () => { 
        document.getElementById('audio').play();
        setTimeout(() => { if (this.volume < 1) this.snack.stillMuted = true; }, 15000);
        this.playing = true;
        this.playingLoading = false;
      }, false);
    },
    loadRequests (pageNum = 1) {
      // https://radio.aljaxus.eu/api/station/1/requests
      // https://radio.aljaxus.eu/api/station/1/request/REQUESTID
      this.request.dataLoaded = false;
      setTimeout(()=>{
        axios.get('https://radio.aljaxus.eu/api/station/1/requests', { params: {
            page: pageNum
          }})
          .then(response => {
            this.request.dataErr = false;
            this.request.dataLoaded = true;
            this.request.data = response.data;
          })
          .catch(function (error) {
            this.request.dataErr = error;
          });
      }, Math.floor(Math.random()*(250-50+1)+50));
    },
    requestSong (requestUrl) {
      this.request.showDialog = false;
      this.request.submitSnack.shown = false;
      axios.get('https://radio.aljaxus.eu'+requestUrl)
        .then(response => {
          // eslint-disable-next-line
          console.log(response);
          this.request.submitSnack.type = 'success';
          this.request.submitSnack.txt = response.data;
          this.request.submitSnack.shown = true;
        })
        .catch(error => {
          // eslint-disable-next-line
          console.log(error);
          this.request.submitSnack.type = 'error';
          this.request.submitSnack.txt = 'An error occured!';
          this.request.submitSnack.shown = true;
        }); 
    },
    loadNowPlaying () {
      axios.get('https://radio.aljaxus.eu/api/nowplaying/1')
        .then(response => {
          let dat = response.data.now_playing;
          this.nowPlaying.title = dat.song.title;
          this.nowPlaying.artist = dat.song.artist;
          this.nowPlaying.edit_by = dat.song.custom_fields.edit_by;
          this.nowPlaying.art = dat.song.art;
          this.nowPlaying.duration = dat.duration;
          this.nowPlaying.elapsed = dat.elapsed;

          clearInterval(this.elapsedCounter);
          this.elapsedCounter = setInterval(() => { if (this.nowPlaying.elapsed <= this.nowPlaying.duration) {this.nowPlaying.elapsed = this.nowPlaying.elapsed+1;} }, 1000);

          let delay = dat.remaining < 10 ? (dat.remaining > 0 ? dat.remaining + 1 : 5) : 30;
          setTimeout(() => { this.loadNowPlaying() }, (delay * 1000));
        })
        .catch(function (error) {
          // eslint-disable-next-line
          console.log(error);
        });
    },
    secToStr(sec_int) {
        let string = '';
        let hours   = Math.floor(sec_int / 3600);
        let minutes = Math.floor((sec_int - (hours * 3600)) / 60);
        let seconds = sec_int - (hours * 3600) - (minutes * 60);

        string += hours >= 1 ? (hours < 10 ? "0"+hours+':' : hours+':') : '';
        string += minutes >= 1 ? (minutes < 10 ? "0"+minutes+':' : minutes+':') : '00:';
        string += seconds >= 1 ? (seconds < 10 ? "0"+seconds : seconds) : '00';

        return string;
    }
  }
}
</script>
<style>
  @import './assets/master.css';
  .nowPlayingImage,
  .requestImage {
    background-size: cover;
    background-repeat: no-repeat;
    background-image: url('./assets/img/art-placeholder.jpg');
    max-width: 100%;
    min-width: 100%;
    height: auto;
  }
</style>
