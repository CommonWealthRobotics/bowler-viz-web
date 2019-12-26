<template>
    <div class="scene">
        <div class="scene" id="container" ref="container">
            <div id="stats" ref="stats"/>
        </div>
    </div>
</template>

<script>
    import * as Three from "three";
    import {OrbitControls} from "three/examples/jsm/controls/OrbitControls"
    import {OBJLoader} from "three/examples/jsm/loaders/OBJLoader"
    import Stats from "stats.js"

    class webSocketHandler {

        constructor(robot, wsuri) {
            const obj = this;
            this.robot = robot;
            // eslint-disable-next-line no-console
            console.log(wsuri);
            this.ws = new WebSocket(wsuri);
            this.ws.binaryType = 'arraybuffer';
            this.ws.onmessage = function (j) {
                obj.handleData(j);
            };
        }

        handleData(e) {
            // data is big endian
            const dv = new DataView(e.data);
            const command = dv.getUint32(0);
            switch (command) {
                case 1:
                    // Position Update
                    this.positionUpdate(dv);
                    break;
            }
        }

        positionUpdate(dv) {
            const rlink = dv.getUint32(4);
            const link = this.robot.linkObjects[rlink];

            if (link != null) {
                if (link.transform == null) {
                    link.transform = new Three.Matrix4();
                }

                for (let i = 0; i < 16; i++) {
                    link.transform.elements[i] = dv.getFloat32((i + 2) * 4);
                }

                link.update = true;
            }
        }
    }

    class robotLink {

        constructor(worldGroup, loader, uri, index) {
            // eslint-disable-next-line no-console
            console.log("display-links: New Link '" + uri + "': " + index + "");

            const obj = this;
            loader.load(uri, function (mesh) {
                mesh.matrixAutoUpdate = false;
                obj.mesh = mesh;
                worldGroup.add(mesh);

                // eslint-disable-next-line no-console
                console.log("display-links: Object " + index + " loaded");
            });
        }
    }

    class robot {

        constructor(worldGroup, loader, uri) {
            this.worldGroup = worldGroup;
            this.loader = loader;
            this.linkObjects = [];

            // eslint-disable-next-line no-console
            console.log("display-links: New Robot '" + uri + "'");

            //kick off async request for robots file.
            fetch(uri).then(it => it.json()).then(it => this.loadCad(it));
        }

        loadCad(json) {
            // Request returned with valid JSON
            const cad = json.robots[0].cad;

            // eslint-disable-next-line no-console
            console.log("display-links: Loading " + cad.length + " CAD objects");

            // Iterate over create robotLink Objects
            for (let i = 0; i < cad.length; i++) {
                // eslint-disable-next-line no-console
                console.log("display-links: Adding Link " + i);

                // Pass URI and index to the constructor.
                const robotlink = new robotLink(this.worldGroup, this.loader,
                    "http://localhost:8000" + cad[i], i);
                this.linkObjects.push(robotlink);
            }
        }

        applyTransforms() {
            for (let i = 0; i < this.linkObjects.length; i++) {
                const l = this.linkObjects[i];
                if (l.transform != null && l.mesh != null && l.update) {
                    l.mesh.matrix = this.linkObjects[i].transform;
                    l.update = false;
                }
            }
        }
    }

    export default {
        name: "RobotView3DScene",
        data() {
            return {
                camera: null,
                controls: null,
                scene: null,
                renderer: null,
                robot: null,
                wsHandle: null,
                loader: null,
                stats: null,
                worldGroup: null
            }
        },
        methods: {
            init: function () {
                const container = this.$refs.container;

                this.renderer = new Three.WebGLRenderer({antialias: false});
                this.renderer.setSize(container.clientWidth, container.clientHeight);
                container.appendChild(this.renderer.domElement);

                this.stats = new Stats();
                this.$refs.stats.appendChild(this.stats.domElement);

                this.camera = new Three.PerspectiveCamera(75,
                    container.clientWidth / container.clientHeight, 0.01, 10000);
                this.camera.position.set(300, 300, -300);

                this.controls = new OrbitControls(this.camera, this.renderer.domElement);

                this.scene = new Three.Scene();
                this.scene.background = new Three.Color(0x777777);

                this.worldGroup = new Three.Group();
                this.worldGroup.rotateX(-Math.PI / 2);
                this.scene.add(this.worldGroup);

                const gridHelper = new Three.GridHelper(1000, 10);
                gridHelper.rotateX(-Math.PI / 2);
                this.worldGroup.add(gridHelper);

                const axesHelper = new Three.AxesHelper(1000);
                axesHelper.material.linewidth = 2;
                this.worldGroup.add(axesHelper);

                const ambientLight = new Three.AmbientLight(0x404040);
                this.scene.add(ambientLight);

                const directionalLight = new Three.DirectionalLight(0xffffff, 5.0);
                directionalLight.position.set(10, 10, 10);
                this.scene.add(directionalLight);

                this.loader = new OBJLoader();

                const jvmAddr = "localhost:8000";
                this.robot = new robot(this.worldGroup, this.loader,
                    "http://" + jvmAddr + "/robots");

                // const wsuri = ((window.location.protocol === "https:") ? "wss://" : "ws://") +
                //     window.location.host + "/robot/socket/MyTestRobot";
                this.wsHandle = new webSocketHandler(this.robot,
                    "ws://" + jvmAddr + "/robot/socket/MyTestRobot");
            },
            animate: function () {
                this.stats.begin();
                this.robot.applyTransforms();
                this.controls.update();
                this.renderer.render(this.scene, this.camera);
                this.stats.end();
                requestAnimationFrame(this.animate);
            }
        },
        mounted() {
            this.init();
            this.animate();
        },
        destroyed() {
            this.renderer.forceContextLoss();
            this.renderer.domElement = null;
        }
    }
</script>

<style scoped>
    .scene {
        width: 100%;
        height: 100%;
    }

    #container {
        position: relative;
    }

    #stats {
        position: absolute;
        top: 0;
        left: 0;
    }
</style>
