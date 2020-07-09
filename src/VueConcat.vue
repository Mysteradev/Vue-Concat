<template>
    <div class="player">
        <div id='audio-player'></div>
        <div id="video-player">
            <div id="progress" class="progress-seek">
                <button @click="playerButton()" id="playPauseBtn"><i class='fa fa-pause'></i></button>
                <input type="range" class="slider" id="seek-bar" value="0">
            </div>
        </div>
    </div>
</template>

<script>
    export default {
        name: 'VueConcat',
        data () {
            return {
                id: null,
                name: null,
                playerIndex: 0,
                players: [],
                currentPlayer: null,
                currentZIndex: 1000
            }
        },
        props: {
            playlist: {
                type: [Array, Object],
                required: true
            }
        },
        mounted: function () {
            let self = this;
            const videoPlayer = document.getElementById("video-player");
            const audioPlayer = document.getElementById("audio-player");
            let videoDuration = 0;

            // Get duration of each video
            for (let index=0; index < this.playlist.length; index++) {
                self.playlist[index].timeInPlaylist = [];
                self.playlist[index].duration = 0;
                let audio = document.createElement('audio');
                audio.setAttribute('id', 'audio' + index);
                audio.setAttribute('src', self.playlist[index].bucketRef);
                audioPlayer.appendChild(audio);
                audio.addEventListener('canplaythrough', function(e){
                    videoDuration = Math.round(e.currentTarget.duration);
                    self.playlist[index].duration = videoDuration;
                });
            }
            // Set time in playlist for each video
            let numOfTimes = 0;
            let keepCheckingForDuration = setInterval(function () {
                let playlistLength = 0;
                for (let index = 0; index < self.playlist.length; index++) {
                    self.playlist[index].timeInPlaylist[0] = playlistLength;
                    playlistLength = playlistLength + self.playlist[index].duration;
                    self.playlist[index].timeInPlaylist[1] = playlistLength;
                }
                if (++numOfTimes === 5) {
                    window.clearInterval(keepCheckingForDuration);
                }
            }, 1000);
            // Preload videos and play the first one
            for (let i = 0; i < self.playlist.length; i++) {
                let newPlayer = document.createElement('video');
                newPlayer.setAttribute('id', 'player' + i);

                // preload each video
                newPlayer.src = self.playlist[i].bucketRef;
                newPlayer.load();
                videoPlayer.appendChild(newPlayer);
                document.getElementById('player' + i).volume = 1;
                self.players[i] = document.getElementById('player' + i);
            }
            for (let i = 0; i < self.players.length; i++) {
                self.players[i].onended = function() { self.switchVideo() };
                self.players[i].onclick = function() { self.playerButton() };
                self.players[i].style.maxWidth = "100%";
                self.players[i].style.top = "0";
                self.players[i].style.left = "0";
                self.players[i].style.position = 'absolute';
            }
            self.currentPlayer = self.players[0];
            self.currentPlayer.style.zIndex = self.currentZIndex;
            let seekBar = document.getElementById("seek-bar");

            // Update the seek bar as the video plays
            self.currentPlayer.addEventListener("timeupdate", function() {
                // Calculate the slider value
                let value = (100 / self.getTotalDuration(self.playlist)) * self.currentPlayer.currentTime;

                // Update the slider value
                seekBar.value = value;
            });

            // Event listener for the seek bar
            seekBar.addEventListener("change", function() {
                // Calculate the new time
                let time = self.currentPlayer.duration * (seekBar.value / 100);

                let currentVideoIndex = self.playerIndex;
                self.checkSeekBarVideo(seekBar.value, currentVideoIndex, time);
            });
        },
        methods: {
            switchVideo() {
                if (this.playerIndex < (this.playlist.length - 1)) {
                    let self = this;
                    this.playerIndex++;
                    let seekBar = document.getElementById("seek-bar");
                    self.currentPlayer = self.players[self.playerIndex];
                    if(self.playerIndex > 0) {
                        self.players[self.playerIndex-1].style.zIndex = "0";
                    }
                    self.currentPlayer.style.zIndex = self.currentZIndex++;
                    // Update the seek bar as the video plays
                    self.currentPlayer.addEventListener("timeupdate", function() {
                        // Calculate the slider value
                        let value = (100 / self.getTotalDuration(self.playlist)) * (self.playlist[self.playerIndex].timeInPlaylist[0] + self.currentPlayer.currentTime);

                        // Update the slider value
                        seekBar.value = value;
                    });
                    // Event listener for the seek bar
                    seekBar.addEventListener("change", function() {
                        // Calculate the new time
                        let time = self.currentPlayer.duration * (seekBar.value / 100);
                        // Update the video time
                        let currentVideoIndex = self.playerIndex;
                        self.checkSeekBarVideo(seekBar.value, currentVideoIndex, time);
                    });
                    let playPromise = self.currentPlayer.play();
                    if (playPromise !== undefined) {
                        playPromise.then(() => {
                            self.currentPlayer.play()
                        })
                            .catch(() => {
                                // console.log(error);
                            });
                    }
                }
                else {
                    this.currentPlayer = this.players[0];
                    this.currentPlayer.style.zIndex = this.currentZIndex;
                    this.currentPlayer.currentTime = 0;
                }
            },
            playerButton() {
                let player = this.currentPlayer;
                if (player.paused === true) {
                    // Play the video
                    let playPromise = player.play();
                    if (playPromise !== undefined) {
                        playPromise.then(() => {
                            player.play()
                        })
                            .catch(() => {
                                // console.log(error);
                            });
                    }
                    document.getElementById('playPauseBtn').innerHTML="<i class='fa fa-pause'></i>";
                } else {
                    // Pause the video
                    let playPromise = player.pause();
                    if (playPromise !== undefined) {
                        playPromise.then(() => {
                            player.pause()
                        })
                            .catch(() => {
                                // console.log(error);
                            });
                    }
                    document.getElementById('playPauseBtn').innerHTML="<i class='fa fa-play'></i>";
                }
            },
            getTotalDuration(playlist) {
                let duration = 0;
                for (let i = 0; i < playlist.length; i++) {
                    duration += playlist[i].duration
                }
                return duration
            },
            checkSeekBarVideo(seekValue, index) {
                let seekBar = document.getElementById('seek-bar');
                // If current seek value is in the range of this particular video, rewind/forward it
                if ((seekValue < ((this.playlist[index].timeInPlaylist[1]) / this.getTotalDuration(this.playlist)) * 100) && (seekValue > (this.playlist[index].timeInPlaylist[0] / this.getTotalDuration(this.playlist)) * 100)) {
                    seekBar.value = seekValue;
                    this.currentPlayer.currentTime = ((this.getTotalDuration(this.playlist) * (seekValue / 100)) - this.playlist[index].timeInPlaylist[0]);
                    let playPromise = this.currentPlayer.play();
                    if (playPromise !== undefined) {
                        playPromise.then(() => {
                            this.currentPlayer.play();
                        })
                            .catch(() => {
                                // console.log(error);
                            });
                    }
                } // Otherwise, load a different video that's in this range
                else {
                    let seekTo = this.getTotalDuration(this.playlist) * (seekValue / 100);
                    for (let i = 0; i < this.playlist.length; i++) {
                        if ((seekTo < this.playlist[i].timeInPlaylist[1]) && (seekTo > this.playlist[i].timeInPlaylist[0])) {
                            let playPromise = this.currentPlayer.pause();
                            if (playPromise !== undefined) {
                                playPromise.then(() => {
                                    this.currentPlayer.pause();
                                })
                                    .catch(() => {
                                        // console.log(error);
                                    });
                            }
                            this.playerIndex = i;
                            this.currentPlayer = this.players[this.playerIndex];
                            this.currentPlayer.currentTime = seekTo - this.playlist[i].timeInPlaylist[0];
                            this.currentPlayer.style.zIndex = this.currentZIndex++;
                            let playPromise2 = this.currentPlayer.play();
                            if (playPromise2 !== undefined) {
                                playPromise2.then(() => {
                                    this.currentPlayer.play();
                                })
                                    .catch(() => {
                                        // console.log(error);
                                    });
                            }
                        }
                    }
                }
            },
        }
    }
</script>

<style>
    .player {
        max-width: 770px;
    }
    button#playPauseBtn {
        padding: 0;
        margin-right: 10px;
        color: rgba(255,255,255,0.85);
        border-radius: 0;
        background-color: rgba(0,0,0,0);
    }
    #progress {
        position: absolute;
        z-index: 2000;
        bottom: 0;
        width: 99.96%;
        background-color: rgba(0,0,0,0.7);
        left: 0;
    }

    input[type=range].slider {
        width: 90%;
        margin: 10px 0;
        background-color: rgba(0,0,0,0.1);
        border-radius: 0;
    }
    input[type=range].slider:focus {
        outline: none;
    }
    input[type=range].slider::-webkit-slider-runnable-track {
        width: 100%;
        height: 10px;
        cursor: pointer;
        /* box-shadow: 0px 0px 0.3px #000000, 0px 0px 0px #0d0d0d; */
        background: #484d4d;
        border-radius: 2.5px;
        border: 0px solid #010101;
    }
    input[type=range].slider::-webkit-slider-thumb {
        box-shadow: 0px 0px 1px #730000, 0px 0px 0px #8d0000;
        border: 0px solid rgba(108, 202, 149, 0.48);
        height: 28px;
        width: 13px;
        border-radius: 7px;
        background: rgba(208, 0, 1, 0.93);
        cursor: pointer;
        -webkit-appearance: none;
        margin-top: -3px;
    }
    input[type=range].slider:focus::-webkit-slider-runnable-track {
        background: #abb1b1;
    }
    input[type=range].slider::-moz-range-track {
        width: 100%;
        height: 10px;
        cursor: pointer;
        background: #484d4d;
        border: 0;
    }
    input[type=range].slider::-moz-range-thumb {
        box-shadow: 0px 0px 1px #730000, 0px 0px 0px #8d0000;
        border: 0px solid rgba(108, 202, 149, 0.48);
        height: 28px;
        width: 13px;
        border-radius: 7px;
        background: rgba(208, 0, 1, 0.93);
        cursor: pointer;
    }
    input[type=range].slider::-ms-track {
        width: 100%;
        height: 10px;
        cursor: pointer;
        background: transparent;
        border-color: transparent;
        color: transparent;
    }
    input[type=range].slider::-ms-fill-lower {
        background: #000000;
        border: 0px solid #010101;
        border-radius: 5px;
        box-shadow: 0px 0px 0.3px #000000, 0px 0px 0px #0d0d0d;
    }
    input[type=range].slider::-ms-fill-upper {
        background: #484d4d;
        border: 0px solid #010101;
        border-radius: 5px;
        box-shadow: 0px 0px 0.3px #000000, 0px 0px 0px #0d0d0d;
    }
    input[type=range].slider::-ms-thumb {
        box-shadow: 0px 0px 1px #730000, 0px 0px 0px #8d0000;
        border: 0px solid rgba(108, 202, 149, 0.48);
        height: 28px;
        width: 13px;
        border-radius: 7px;
        background: rgba(208, 0, 1, 0.93);
        cursor: pointer;
        height: 10px;
    }
    input[type=range].slider:focus::-ms-fill-lower {
        background: #484d4d;
    }
    input[type=range].slider:focus::-ms-fill-upper {
        background: #abb1b1;
    }
    #video-player {
        position: relative;
        height: 390px;
        width: 90%;
        margin: 0 auto;
    }
</style>