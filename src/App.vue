<template>
    <div id="app" :style="{height: '100vh', width: '100vw'}">
        <!-- <input type="button" value="Start webcam feed" @click="init" /> -->
        <canvas
            ref="sence"
            :height="window.h"
            :width="window.w"
            :style="{height: '100vh', width: '100vw'}"
        ></canvas>
        <canvas ref="canvas" width="640" height="480" style="display: none"></canvas>
    </div>
</template>

<script>
import Eye from "./assets/Eye";

const pageHeight = window.innerHeight;
const pageWidth = window.innerWidth;

export default {
    data() {
        return {
            initialized: false,
            window: {
                h: pageHeight,
                w: pageWidth
            }
        };
    },
    computed: {
        sence() {
            return this.$refs["sence"].getContext("2d");
        }
    },
    mounted() {
        // this.init();
    },
    methods: {
        init() {
            let self = this;
            //	(0) check whether we're already running face detection
            // if yes, then do not initialize everything again
            if (this.initialized) return;
            //	(1) initialize the pico.js face detector
            // we will use the detecions of the last 5 frames
            let update_memory = pico.instantiate_detection_memory(5);
            fetch("/facefinder").then(response => {
                response.arrayBuffer().then(buffer => {
                    let bytes = new Int8Array(buffer);
                    this.facefinder_classify_region = pico.unpack_cascade(
                        bytes
                    );
                    console.log("* facefinder loaded");
                });
            });
            //	(2) initialize the lploc.js library with a pupil localizer
            fetch("/puploc.bin").then(response => {
                response.arrayBuffer().then(buffer => {
                    let bytes = new Int8Array(buffer);
                    this.do_puploc = lploc.unpack_localizer(bytes);
                    console.log("* puploc loaded");
                });
            });
            //	(3) get the drawing context on the canvas and define a function to transform an RGBA image to grayscale
            let ctx = this.$refs["canvas"].getContext("2d");

            //	(4) this function is called each time a video frame becomes available
            let processfn = function(video, dt) {
                // render the video frame to the canvas element and extract RGBA pixel data
                ctx.drawImage(video, 0, 0);
                var rgba = ctx.getImageData(0, 0, 640, 480).data;
                // prepare input to `run_cascade`
                const image = {
                    pixels: self.rgbaToGrayscale(rgba, 480, 640),
                    nrows: 480,
                    ncols: 640,
                    ldim: 640
                };
                const params = {
                    shiftfactor: 0.1, // move the detection window by 10% of its size
                    minsize: 100, // minimum size of a face
                    maxsize: 1000, // maximum size of a face
                    scalefactor: 1.1 // for multiscale processing: resize the detection window by 10% when moving to the higher scale
                };
                // run the cascade over the frame and cluster the obtained detections
                // dets is an array that contains (r, c, s, q) quadruplets
                // (representing row, column, scale and detection score)
                let dets = pico.run_cascade(
                    image,
                    self.facefinder_classify_region,
                    params
                );
                dets = update_memory(dets);
                dets = pico.cluster_detections(dets, 0.2); // set IoU threshold to 0.2

                // draw detections
                for (i = 0; i < dets.length; ++i)
                    // check the detection score
                    // if it's above the threshold, draw it
                    // (the constant 50.0 is empirical: other cascades might require a different one)
                    if (dets[i][3] > 50.0) {
                        let r, c, s;
                        // 脸部轮廓
                        // self.drawFaceOutline(ctx, dets);
                        //
                        // find the eye pupils for each detected face.
                        // starting regions for localization are initialized
                        // based on the face bounding box.
                        // (parameters are set empirically)

                        let eye1, eye2;

                        // first eye
                        r = dets[i][0] - 0.075 * dets[i][2];
                        c = dets[i][1] - 0.175 * dets[i][2];
                        s = 0.35 * dets[i][2];
                        [r, c] = self.do_puploc(r, c, s, 63, image);
                        if (r >= 0 && c >= 0) {
                            // self.drawEye(ctx, c, r);
                            eye1 = new Eye(c, r);
                        }
                        // second eye
                        r = dets[i][0] - 0.075 * dets[i][2];
                        c = dets[i][1] + 0.175 * dets[i][2];
                        s = 0.35 * dets[i][2];
                        [r, c] = self.do_puploc(r, c, s, 63, image);
                        if (r >= 0 && c >= 0) {
                            // self.drawEye(ctx, c, r);
                            eye2 = new Eye(c, r);
                        }

                        if (eye1 && eye2) {
                            self.drawBasicDot(eye1, eye2);
                        }
                    }
            };
            //	(5) instantiate camera handling (see https://github.com/cbrandolino/camvas)
            let mycamvas = new camvas(ctx, processfn);
            //	(6) it seems that everything went well
            this.initialized = true;
        },
        rgbaToGrayscale(rgba, nrows, ncols) {
            let gray = new Uint8Array(nrows * ncols);
            for (let r = 0; r < nrows; ++r) {
                for (let c = 0; c < ncols; ++c) {
                    gray[r * ncols + c] =
                        (2 * rgba[r * 4 * ncols + 4 * c + 0] +
                            7 * rgba[r * 4 * ncols + 4 * c + 1] +
                            1 * rgba[r * 4 * ncols + 4 * c + 2]) /
                        10;
                }
            }
            return gray;
        },
        drawFaceOutline(ctx, dets) {
            ctx.beginPath();
            ctx.arc(
                dets[i][1],
                dets[i][0],
                dets[i][2] / 2,
                0,
                2 * Math.PI,
                false
            );
            ctx.lineWidth = 3;
            ctx.strokeStyle = "red";
            ctx.stroke();
        },
        drawEye(ctx, c, r) {
            ctx.beginPath();
            ctx.arc(c, r, 1, 0, 2 * Math.PI, false);
            ctx.lineWidth = 3;
            ctx.strokeStyle = "red";
            ctx.stroke();
        },
        drawLine(ctx, eye1, eye2) {
            ctx.beginPath();
            ctx.moveTo(eye1.x, eye1.y);
            ctx.lineTo(eye2.x, eye2.y);
            ctx.lineWidth = 3;
            ctx.strokeStyle = "red";
            ctx.stroke();
            ctx.closePath();
        },
        drawDotBetweenEyes(ctx, eye1, eye2) {
            const x = (eye1.x + eye2.x) / 2;
            const y = (eye1.y + eye2.y) / 2;
            this.drawEye(ctx, x, y);
        },
        drawBasicDot(eye1, eye2) {
            const x = this.$refs["sence"].width / 2;
            const y = (eye1.y + eye2.y) / 2;
            const ctx = this.sence;
            ctx.clearRect(
                0,
                0,
                this.$refs["sence"].width,
                this.$refs["sence"].height
            );
            ctx.beginPath();
            ctx.arc(x, y, 1, 0, 2 * Math.PI, false);
            ctx.lineWidth = 10;
            ctx.strokeStyle = "black";
            ctx.stroke();
        },
        do_puploc(r, c, s, nperturbs, pixels, nrows, ncols, ldim) {
            return [-1.0, -1.0];
        },
        facefinder_classify_region(r, c, s, pixels, ldim) {
            return -1.0;
        }
    }
};
</script>

<style lang="stylus">
* {
    margin: 0;
    padding: 0;
}
</style>