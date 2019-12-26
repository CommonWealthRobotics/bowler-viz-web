<template>
    <div class="scene" ref="container"/>
</template>

<script>
    import * as Three from "three";
    import {OrbitControls} from "three/examples/jsm/controls/OrbitControls"
    import {OBJLoader} from "three/examples/jsm/loaders/OBJLoader"

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

            if (this.robot.linkObjects[rlink] != null) {
                const transform = [];
                for (let i = 0; i < 16; i++) {
                    transform.push(dv.getFloat32((i + 2) * 4));
                }

                if (this.robot.linkObjects[rlink].transform == null) {
                    this.robot.linkObjects[rlink].transform = new Three.Matrix4();
                }

                this.robot.linkObjects[rlink].transform.elements = transform;
                this.robot.linkObjects[rlink].update = true;
            }
        }
    }

    class robotLink {

        constructor(scene, loader, uri, index) {
            // eslint-disable-next-line no-console
            console.log("display-links: New Link '" + uri + "': " + index + "");

            const obj = this;
            loader.load(uri, function (mesh) {
                mesh.matrixAutoUpdate = false;
                obj.mesh = mesh;
                scene.add(mesh);

                // eslint-disable-next-line no-console
                console.log("display-links: Object " + index + " loaded");
            });
        }
    }

    class robot {

        constructor(scene, loader, uri) {
            this.scene = scene;
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
                const robotlink = new robotLink(this.scene, this.loader,
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
                loader: null
            }
        },
        methods: {
            init: function () {
                const container = this.$refs.container;

                this.renderer = new Three.WebGLRenderer({antialias: false});
                this.renderer.setSize(container.clientWidth, container.clientHeight);
                container.appendChild(this.renderer.domElement);

                this.camera = new Three.PerspectiveCamera(75,
                    container.clientWidth / container.clientHeight, 0.01, 1000);
                this.camera.position.set(0, 0, 200);

                this.controls = new OrbitControls(this.camera, this.renderer.domElement);

                this.scene = new Three.Scene();
                this.scene.background = new Three.Color(0x777777);

                const groundPlane = new Three.LineSegments(
                    new Three.WireframeGeometry(
                        new Three.PlaneGeometry(
                            1000, 1000, 10, 10)),
                    new Three.LineBasicMaterial({color: 0xffffff, linewidth: 2})
                );
                groundPlane.rotateX(1.570796);
                this.scene.add(groundPlane);

                this.createAxis(0xff0000, new Three.Vector3(1000, 0, 0));
                this.createAxis(0x00ff00, new Three.Vector3(0, 1000, 0));
                this.createAxis(0x0000ff, new Three.Vector3(0, 0, 1000));

                const ambientLight = new Three.AmbientLight(0x404040);
                this.scene.add(ambientLight);

                const directionalLight = new Three.DirectionalLight(0xffffff, 5.0);
                directionalLight.position.set(10, 10, 10);
                this.scene.add(directionalLight);

                this.loader = new OBJLoader();

                const jvmAddr = "localhost:8000";
                this.robot = new robot(this.scene, this.loader,
                    "http://" + jvmAddr + "/robots");

                // const wsuri = ((window.location.protocol === "https:") ? "wss://" : "ws://") +
                //     window.location.host + "/robot/socket/MyTestRobot";
                this.wsHandle = new webSocketHandler(this.robot,
                    "ws://" + jvmAddr + "/robot/socket/MyTestRobot");
            },
            animate: function () {
                this.robot.applyTransforms();
                requestAnimationFrame(this.animate);
                this.controls.update();
                this.renderer.render(this.scene, this.camera);
            },
            createAxis: function(color, endPoint) {
                const xAxisGeom = new Three.Geometry();
                xAxisGeom.vertices.push(new Three.Vector3(0, 0, 0));
                xAxisGeom.vertices.push(endPoint);
                const xAxis = new Three.Line(
                    xAxisGeom,
                    new Three.LineBasicMaterial({color: color, linewidth: 2})
                );
                this.scene.add(xAxis);
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
</style>
