<template>
    <div class="scene" ref="container"/>
</template>

<script>
    import * as Three from "three";
    import {OrbitControls} from "three/examples/jsm/controls/OrbitControls"
    import {OBJLoader} from "three/examples/jsm/loaders/OBJLoader"

    class webSocketHandler {
        ws = null;

        constructor(wsuri) {
            var obj = this;
            // eslint-disable-next-line no-console
            console.log(wsuri);
            this.ws = new WebSocket(wsuri);
            this.ws.binaryType = 'arraybuffer';
            this.ws.onmessage = function (j) {
                obj.handleData(j);
            };
        }

        // eslint-disable-next-line no-unused-vars
        connect(wsconnect) {
            // eslint-disable-next-line no-console
            console.log("Websocket Open");
        }

        handleData(e) {
            // eslint-disable-next-line no-console
            console.log("web-socket-handler: Got Data!" + e.data);
            // data is big endian
            var dv = new DataView(e.data);
            var command = dv.getUint32(0);
            switch (command) {
                case 1:
                    // Position Update
                    this.positionUpdate(dv);
                    break;
                case 2:
                    // Dummy Command


                    break;
            }
        }

        positionUpdate(dv) {

            var transform = [];
            var rlink = dv.getUint32(4);
            for (var i = 0; i < 16; i++) {
                transform.push(dv.getFloat32((i + 2) * 4));
            }
            var m = new Three.Matrix4();
            m.elements = transform;
            //debugger;
            if (this.robot.linkObjects[rlink] != null) {

                this.robot.linkObjects[rlink].transform = m;
                this.robot.linkObjects[rlink].update = true;
                // eslint-disable-next-line no-console
                console.log("web-socket-handler: Position for link " + rlink + "! " + transform);
            }
        }

        // eslint-disable-next-line no-unused-vars
        dummyCommand(dv) {
            // eslint-disable-next-line no-console
            console.log("Dummy Command!");
        }
    }

    // robot object
    class robotLink {
        scene = null;
        loader = null;

        constructor(scene, loader, uri, index) {
            this.scene = scene;
            this.loader = loader;
            var obj = this;
            // eslint-disable-next-line no-console
            console.log("display-links: New Link '" + uri + "': " + index + "");
            // Load OBJ and register our selves as callback
            this.loader.load(uri, function (j) {
                obj.addToScene(j);
            });

            this.index = index;
        }

        index = null;
        sceneobject = null;
        transform = null;
        update = false;

        addToScene(mesh) {
            // eslint-disable-next-line no-console
            console.log("display-links: Object " + this.index + " loaded");

            //debugger;
            //debugger;
            // Set a material
            //mesh.children[0].material = new THREE.MeshBasicMaterial( { color: 0x00ff00 } );
            this.sceneobject = mesh;
            //debugger;
            // Add ourselves to the global scene.
            this.scene.add(mesh);
        }
    }

    class robot {
        // an array of robotLinks for future reference.
        linkObjects = [];
        scene = null;
        loader = null;

        constructor(scene, loader, uri) {
            this.scene = scene;
            this.loader = loader;
            // eslint-disable-next-line no-console
            console.log("display-links: New Robot '" + uri + "'");
            //kick off async request for robots file.
            fetch(uri).then(it => it.json()).then(it => this.loadCad(it));
            this.linkObjects = [];
        }

        loadCad(json) {
            // Request returned with valid JSON
            var cad = json.robots[0].cad;
            // eslint-disable-next-line no-console
            console.log("display-links: Loading " + cad.length + " CAD objects");

            // Iterate over create robotLink Objects
            for (var i = 0; i < cad.length; i++) {
                // eslint-disable-next-line no-console
                console.log("display-links: Adding Link " + i);
                // Pass URI and index to the constructor.
                var robotlink = new robotLink(this.scene, this.loader, cad[i], i);
                this.linkObjects.push(robotlink);

            }
        }

        applyTransforms() {
            var lojb = this.linkObjects;

            for (var i = 0; i < lojb.length; i++) {
                var l = lojb[i];
                if (l.transform != null && l.sceneobject != null && l.update) {
                    // eslint-disable-next-line no-console
                    console.log("display-links: Updating matrix");
                    if (l.sceneobject.matrixAutoUpdate != false)
                        l.sceneobject.matrixAutoUpdate = false;
                    l.sceneobject.matrix = lojb[i].transform;
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
                loader: null
            }
        },
        methods: {
            init: function () {
                let container = this.$refs.container;

                this.renderer = new Three.WebGLRenderer({antialias: true});
                this.renderer.setSize(container.clientWidth, container.clientHeight);
                container.appendChild(this.renderer.domElement);

                this.camera = new Three.PerspectiveCamera(75,
                    container.clientWidth / container.clientHeight, 0.01, 1000);
                this.camera.position.set(0, 0, 200);

                this.controls = new OrbitControls(this.camera, this.renderer.domElement);

                this.scene = new Three.Scene();
                this.scene.background = new Three.Color(0x777777);

                let ambientLight = new Three.AmbientLight(0x404040);
                this.scene.add(ambientLight);

                this.loader = new OBJLoader();

                this.robot = new robot(this.scene, this.loader, "/robots");
                let wsuri = ((window.location.protocol === "https:") ? "wss://" : "ws://") +
                    window.location.host + "/robot/socket/MyTestRobot";
                this.wsHandle = new webSocketHandler(wsuri);
            },
            animate: function () {
                this.robot.applyTransforms();
                requestAnimationFrame(this.animate);
                this.controls.update();
                this.renderer.render(this.scene, this.camera);
            }
        },
        mounted() {
            this.init();
            this.animate();
        },
        destroyed() {
            this.renderer.forceContextLoss();
            this.renderer.context = null;
            this.renderer.domElement = null;
            this.renderer = null;
        }
    }
</script>

<style scoped>
    .scene {
        width: 100%;
        height: 100%;
    }
</style>
