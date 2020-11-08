<template>
    <div>
        <canvas ref="canvas" width="640" height="480"></canvas>

        <div>
            <p>                
            口距離: {{mouseDistance}} <br/>
            開口閾値: {{mouseDistanceThreshold}} <input type="range" v-model="mouseDistanceThreshold" max=1000>
            </p>
            <p>                
            左目: {{leftEyeDistance}} <br/>
            左瞬き閾値: {{leftEyeDistanceThreshold}} <input type="range" v-model="leftEyeDistanceThreshold">
            </p>
                        <p>                
            右目: {{rightEyeDistance}} <br/>
            右瞬き閾値: {{rightEyeDistanceThreshold}} <input type="range" v-model="rightEyeDistanceThreshold">
            </p>
        </div>
    </div>

</template>

<script>
import Vue from "vue";
import * as three from "three";
// import * as facemesh from "@tensorflow-models/facemesh";
import * as faceLandmarkDetection from "@tensorflow-models/face-landmarks-detection";
import * as tf from "@tensorflow/tfjs-core";
import "@tensorflow/tfjs-backend-webgl";
import "@tensorflow/tfjs-backend-cpu";
export default Vue.extend({
    data() {
        return {
            model: null,
            canvas: null,
            video: null,
            renderer: null,
            camera: null,
            textures: null,
            mouseDistance: 0,
            mouseDistanceThreshold: 95,
            leftEyeDistance: 0,
            leftEyeDistanceThreshold: 10,
            rightEyeDistance: 0,
            rightEyeDistanceThreshold: 10,
        }
    },
    async mounted() {
        await tf.setBackend('webgl');
        // await tf.setBackend('cpu');

        this.video = document.createElement("video");

        this.textures = [];
        for (let i = 0; i < 8; i++) {
            const url = "./poop/" + ("000" + i.toString(2)).slice(-3) + ".png";

            const loadTexture = new Promise((resolve, reject) => {
                const loader = new three.TextureLoader();
                loader.load(url, resolve);
            })
            const tx = await loadTexture;
            this.textures.push(tx);           
        }

        this.video.onloadedmetadata = async () => {
            const width = this.video.videoWidth;
            const height = this.video.videoHeight;
            this.$refs.canvas.width = width;
            this.$refs.canvas.height = height;

            this.renderer = new three.WebGLRenderer({
                canvas: this.$refs.canvas,
                alpha: true,
            });
            this.renderer.setSize(width, height);

            this.camera = new three.PerspectiveCamera(45, width / height);
            this.camera.position.set(0,0, 1000);


            this.canvas = document.createElement('canvas');
            this.canvas.width = width;
            this.canvas.height = height;

            this.model = await faceLandmarkDetection.load(
                faceLandmarkDetection.SupportedPackages.mediapipeFacemesh,
                {maxFaces: 1});

            console.log("START");
            await this.faceDetection();
        }

        await this.startVideo();
    },
    methods: {
        async startVideo() {
            const stream = await navigator.mediaDevices.getUserMedia({video: true, audio: false})
            this.video.srcObject = stream;
            this.video.play();
        },
        async faceDetection() {
            const scene = new three.Scene();
            const predictions = await this.model.estimateFaces({
                input: this.video,
                predictIrises: true,
            });
            const geom = new three.Geometry();
            if (predictions.length > 0) {
                const mesh = predictions[0].scaledMesh;

                const geom2 = new three.PlaneGeometry(1, 1, 1, 1);
                geom2.computeVertexNormals();
                geom2.computeFaceNormals();
                geom2.vertices[1] = new three.Vector3(-mesh[71][0] + 320, -mesh[71][1] + 240, mesh[71][2])
                geom2.vertices[3] = new three.Vector3(-mesh[135][0] + 320, -mesh[135][1] + 240, mesh[135][2])
                geom2.vertices[2] = new three.Vector3(-mesh[367][0] + 320, -mesh[367][1] + 240, mesh[367][2])
                geom2.vertices[0] = new three.Vector3(-mesh[301][0] + 320, -mesh[301][1] + 240, mesh[301][2])

                geom2.__dirtyVertices = true;
                geom2.__dirtyNormals = true;

                let faceflag = 1;
                const staticSizeMesh = predictions[0].mesh;
                this.mouseDistance = (new three.Vector3(staticSizeMesh[13][0], staticSizeMesh[13][1], staticSizeMesh[13][2])).distanceToSquared(new three.Vector3(staticSizeMesh[14][0], staticSizeMesh[14][1], staticSizeMesh[14][2]));
                if (this.mouseDistance >= this.mouseDistanceThreshold) {
                    faceflag -= 1;
                }
                this.leftEyeDistance = (new three.Vector3(staticSizeMesh[386][0], staticSizeMesh[386][1], staticSizeMesh[386][2])).distanceToSquared(new three.Vector3(staticSizeMesh[374][0], staticSizeMesh[374][1], staticSizeMesh[374][2]));
                if (this.leftEyeDistance <= this.leftEyeDistanceThreshold) {
                    faceflag += 4;
                }
                this.rightEyeDistance = (new three.Vector3(staticSizeMesh[159][0], staticSizeMesh[159][1], staticSizeMesh[159][2])).distanceToSquared(new three.Vector3(staticSizeMesh[145][0], staticSizeMesh[145][1], staticSizeMesh[145][2]));
                if (this.rightEyeDistance <= this.rightEyeDistanceThreshold) {
                    faceflag += 2;
                }

                const material2 = new three.MeshBasicMaterial({map: this.textures[faceflag], side: three.DoubleSide});
                scene.add(new three.Mesh(geom2, material2));
            }
            const material = new three.PointsMaterial({size: 10, color: 0xffffff})
            scene.add(new three.Points(geom, material))
            this.renderer.render(scene, this.camera);
            requestAnimationFrame(this.faceDetection);
        }
    }

})
</script>

<style lang="stylus">
.outofscreen {
    position: fixed;
    top: -1px;
    left: -1px;
    overflow:hidden;
    width: 1px;
    height: 1px;
}
body {
    background-color: green;
}
</style>