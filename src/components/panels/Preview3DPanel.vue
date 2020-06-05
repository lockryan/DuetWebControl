<style scoped>
.loading {
	background-color: black;
}
.loading h1{
	color: white;
}
.loading,
.canvas {
	border-radius: 8px 0 0 8px;
}
.legend {
	border-radius: 0 8px 8px 0;
}
canvas {
	display: flex;
	min-height: 480px;
}
.no-cursor {
	pointer-events: none;
}
</style>

<template>
	<div class="component">
		<v-card class="card">
			<v-container ref="container" class="pa-1" fluid>
				<v-layout row wrap fill-height>
					<v-flex class="preview3D-container pa-2" xs12 sm12 md9 lg10 xl10>
						<v-layout ref="parentElement" row fill-height>
                            <v-layout align-center>
                                <v-flex tag="h1" class="text-xs-center">
                                    {{ loading ? $t('generic.loading') : (errorMessage ? errorMessage : $t('panel.heightmap.notAvailable')) }}
                                </v-flex>
                            </v-layout>
							<v-flex shrink>
								<canvas ref="canvas" class="canvas" @mousemove="canvasMouseMove"></canvas>
							</v-flex>
						</v-layout>
					</v-flex>
				</v-layout>
			</v-container>
		</v-card>
	</div>
</template>

<script>
'use strict'

import { mapState, mapGetters, mapActions } from 'vuex'

import * as THREE from 'three'
import OrbitControls from 'three-orbitcontrols'

import { drawLegend, setFaceColors, generateIndicators, generateMeshGeometry } from '../../utils/3d.js'
import Path from '../../utils/path.js'


import colornames from 'colornames';
import Toolpath from 'gcode-toolpath';
//import * as THREE from 'three';
// import log from 'app/lib/log';

const defaultColor = new THREE.Color(colornames('lightgrey'));
const motionColor = {
    'G0': new THREE.Color(colornames('green')),
    'G1': new THREE.Color(colornames('blue')),
    'G2': new THREE.Color(colornames('deepskyblue')),
    'G3': new THREE.Color(colornames('deepskyblue'))
};

// class GCodeVisualizer {
//     constructor() {

//         return this;
//     }


//     setFrameIndex(frameIndex) {
//         if (this.frames.length === 0) {
//             return;
//         }

//         frameIndex = Math.min(frameIndex, this.frames.length - 1);
//         frameIndex = Math.max(frameIndex, 0);

//         const v1 = this.frames[this.frameIndex].vertexIndex;
//         const v2 = this.frames[frameIndex].vertexIndex;

//         // Completed path is grayed out
//         if (v1 < v2) {
//             const workpiece = this.group.children[0];
//             for (let i = v1; i < v2; ++i) {
//                 workpiece.geometry.colors[i] = defaultColor;
//             }
//             workpiece.geometry.colorsNeedUpdate = true;
//         }

//         // Restore the path to its original colors
//         if (v2 < v1) {
//             const workpiece = this.group.children[0];
//             for (let i = v2; i < v1; ++i) {
//                 workpiece.geometry.colors[i] = this.geometry.colors[i];
//             }
//             workpiece.geometry.colorsNeedUpdate = true;
//         }

//         this.frameIndex = frameIndex;
//     }
// }

const scaleZ = 0.5, maxVisualizationZ = 0.25
const indicatorColor = 0xFFFFFF, indicatorOpacity = 0.4, indicatorOpacityHighlighted = 1.0

let threeInstances = []
window.addEventListener('resize', function() {
	threeInstances.forEach(instance => instance.resize())
})

export default {
	beforeCreate() {
        //console.log("beforeCreate");
		this.three = {						// non-reactive data
			scene: null,
			camera: null,
			renderer: null,
			orbitControls: null,
			raycaster: null,

			hasHelpers: false,
			meshGeometry: null,
			meshPlane: null,
			meshIndicators: null,
			lastIntersection: null
		}

        this.group = new THREE.Object3D();
        this.geometry = new THREE.Geometry();


        // Example
        // [
        //   {
        //     code: 'G1 X1',
        //     vertexIndex: 2
        //   }
        // ]
        this.frames = []; // Example
        this.frameIndex = 0;
	},
	computed: {
		...mapGetters(['isConnected']),
		...mapState('settings', ['language'])
	},
	data() {
		return {
			isActive: true,
			ready: false,
			loading: false,
			errorMessage: "",

			colorScheme: 'terrain',
			tooltip: {
				coord: {
					x: 0,
					y: 0,
					z: 0
				},
				x: undefined,
				y: undefined,
				shown: false
			},
			unsubscribe: undefined
		}
	},
	methods: {
		...mapActions('machine', ['download']),
		init() {
			// Perform initial resize
            //console.log("INIT");
			const size = this.resize();

			// Create THREE instances
			this.three.scene = new THREE.Scene();
			this.three.camera = new THREE.PerspectiveCamera(45, size.width / size.height, 0.1, 1000);
			this.three.camera.position.set(0, 0, 10);
			this.three.camera.up = new THREE.Vector3(0, 1, 0);
			this.three.renderer = new THREE.WebGLRenderer({ canvas: this.$refs.canvas });
			this.three.renderer.setSize(size.width, size.height);
			this.three.orbitControls = new OrbitControls(this.three.camera, this.three.renderer.domElement);
			this.three.orbitControls.enableKeys = false;
			this.three.raycaster = new THREE.Raycaster();
    
            this.three.scene.background = new THREE.Color("rgb(200, 200, 200)");
			// Register this instance in order to deal with size changes
			threeInstances.push(this);
//			if (this.isConnected) {
				this.preview();
//			}
            //console.log("inited");
		},
		resize() {
            //console.log("resize");
			if (!this.isActive) {
				return;
			}

			// Resize canvas elements
			// const containerOffset = this.$refs.container.scrollWidth - this.$refs.container.offsetWidth;
			// const width = this.$refs.parentElement.offsetWidth - this.$refs.legend.offsetWidth - containerOffset;
			// const height = this.$refs.parentElement.offsetHeight;
			// this.$refs.legend.height = height;
			// this.$refs.canvas.width = width;
			// this.$refs.canvas.height = height;

            const width = 640;
            const height = 480;

			// Resize the 3D height map
			if (this.three.renderer) {
				this.three.camera.aspect = width / height;
				this.three.camera.updateProjectionMatrix();
				this.three.renderer.setSize(width, height);
			}

			// Redraw the legend and return the canvas size
			//drawLegend(this.$refs.legend, maxVisualizationZ, this.colorScheme);
			return { width, height };
		},
        renderGCode(gcode) {
            //console.log("renderGCode");
            const toolpath = new Toolpath({
                    position: { x: 10, y: 10, z: 0 },

                    // Initial modal state (optional)
                    modal: {
                        motion: 'G0', // G0, G1, G2, G3, G38.2, G38.3, G38.4, G38.5, G80
                        wcs: 'G54', // G54, G55, G56, G57, G58, G59
                        plane: 'G17', // G17: xy-plane, G18: xz-plane, G19: yz-plane
                        units: 'G21', // G20: Inches, G21: Millimeters
                        distance: 'G90', // G90: Absolute, G91: Relative
                        feedrate: 'G94', // G93: Inverse time mode, G94: Units per minute, G95: Units per rev
                        program: 'M0', // M0, M1, M2, M30
                        spindle: 'M5', // M3, M4, M5
                        coolant: 'M9', // M7, M8, M9
                        tool: 0
                    },


                // @param {object} modal The modal object.
                // @param {object} v1 A 3D vector of the start point.
                // @param {object} v2 A 3D vector of the end point.
                addLine: (modal, v1, v2) => {
                    const { motion } = modal;
                    const color = motionColor[motion] || defaultColor;
                    this.geometry.vertices.push(new THREE.Vector3(v2.x, v2.y, v2.z));
                    this.geometry.colors.push(color);
                },
                // @param {object} modal The modal object.
                // @param {object} v1 A 3D vector of the start point.
                // @param {object} v2 A 3D vector of the end point.
                // @param {object} v0 A 3D vector of the fixed point.
                addArcCurve: (modal, v1, v2, v0) => {
                    const { motion, plane } = modal;
                    const isClockwise = (motion === 'G2');
                    const radius = Math.sqrt(
                        ((v1.x - v0.x) ** 2) + ((v1.y - v0.y) ** 2)
                    );
                    let startAngle = Math.atan2(v1.y - v0.y, v1.x - v0.x);
                    let endAngle = Math.atan2(v2.y - v0.y, v2.x - v0.x);

                    // Draw full circle if startAngle and endAngle are both zero
                    if (startAngle === endAngle) {
                        endAngle += (2 * Math.PI);
                    }

                    const arcCurve = new THREE.ArcCurve(
                        v0.x, // aX
                        v0.y, // aY
                        radius, // aRadius
                        startAngle, // aStartAngle
                        endAngle, // aEndAngle
                        isClockwise // isClockwise
                    );
                    const divisions = 30;
                    const points = arcCurve.getPoints(divisions);
                    const color = motionColor[motion] || defaultColor;

                    for (let i = 0; i < points.length; ++i) {
                        const point = points[i];
                        const z = ((v2.z - v1.z) / points.length) * i + v1.z;

                        if (plane === 'G17') { // XY-plane
                            this.geometry.vertices.push(new THREE.Vector3(point.x, point.y, z));
                        } else if (plane === 'G18') { // ZX-plane
                            this.geometry.vertices.push(new THREE.Vector3(point.y, z, point.x));
                        } else if (plane === 'G19') { // YZ-plane
                            this.geometry.vertices.push(new THREE.Vector3(z, point.x, point.y));
                        }
                        this.geometry.colors.push(color);
                    }
                }
            });

            //console.log("ToolPathOK");

            while (this.group.children.length > 0) {
                const child = this.group.children[0];
                this.group.remove(child);
                child.geometry.dispose();
            }

            //console.log("LoadToolPath");
            toolpath.loadFromStringSync(gcode, (line, index) => {
                this.frames.push({
                    data: line,
                    vertexIndex: this.geometry.vertices.length // remember current vertex index
                });
            });
            //console.log("LoadToolPathOK");

            const workpiece = new THREE.Line(
                new THREE.Geometry(),
                new THREE.LineBasicMaterial({
                    color: defaultColor,
                    linewidth: 1,
                    vertexColors: THREE.VertexColors,
                    opacity: 0.5,
                    transparent: true
                })
            );
            workpiece.geometry.vertices = this.geometry.vertices.slice();
            workpiece.geometry.colors = this.geometry.colors.slice();

            this.group.add(workpiece);

            //console.log({
            //     workpiece: workpiece,
            //     frames: this.frames,
            //     frameIndex: this.frameIndex
            // });

            //console.log("renderGCodeDONE");

            return this.group;
        },
        myRender() {
            //console.log("myRender");
            //console.log("myRender0");
            // Generate stats
            let xMin = -5.0;
            let xMax = 5.0;
            let yMin = -5.0;
            let yMax = 5.0;

            //console.log("myRender01");
            this.numPoints = 0;
            this.minDiff = 10;
            this.maxDiff = 10;
            this.meanError = 0;
            this.rmsError = 0;

            //console.log("myRender02");
            // Make indicators and add them
            //console.log("myRender03");

            if (!this.three.hasHelpers) {
                //console.log("Helpers");
                // Make axis arrows for XYZ
                this.three.scene.add(new THREE.ArrowHelper(new THREE.Vector3(1, 0, 0), new THREE.Vector3(0, 0, 0), 0.5, 0xFF0000));
                this.three.scene.add(new THREE.ArrowHelper(new THREE.Vector3(0, 1, 0), new THREE.Vector3(0, 0, 0), 0.5, 0x00FF00));
                this.three.scene.add(new THREE.ArrowHelper(new THREE.Vector3(0, 0, 1), new THREE.Vector3(0, 0, 0), 0.5, 0x0000FF));

                //console.log("Grid");
                // Make grid on XY plane
                const grid = new THREE.GridHelper(100, 50);
                grid.rotation.x = -Math.PI / 2;
                this.three.scene.add(grid);
                //console.log("GridDONE");
                //console.log(grid);

                // Don't add these helpers again
                this.three.hasHelpers = true;
                //console.log("HelpersDone");
            }

            //console.log("myRender04");
            const GCODE = [
                'N1 T2 G17 G20 G90 G94 G54',
                'N2 G0 Z0.25',
                'N3 X-0.5 Y0.',
                'N4 Z0.1',
                'N5 G01 Z0. F5.',
                'N6 G02 X0. Y0.5 I0.5 J0. F2.5',
                'N7 X0.5 Y0. I0. J-0.5',
                'N8 X0. Y-0.5 I-0.5 J0.',
                'N9 X-0.5 Y0. I0. J0.5',
                'N10 G01 Z0.1 F5.',
                'N11 G00 X0. Y0. Z0.25'
            ].join('\n');

            let group = this.renderGCode(GCODE);
            this.three.scene.add(group);

            // Render scene
            // this.ready = true;
            // this.resize();
            this.render();
            //console.log("myRenderDONE");
        },
		render() {
			if (this.three.renderer) {
                // //console.log("render");
				requestAnimationFrame(this.render);
				this.three.renderer.render(this.three.scene, this.three.camera);
			}
		},
		canvasMouseMove(e) {
            // //console.log("canvasMouseMove");
			if (!e.clientX || !this.three.meshGeometry) {
				return;
			}

			// Try to get the Z value below the cursor
			// For that we need normalized X+Y coordinates between -1.0 and 1.0
			const rect = e.target.getBoundingClientRect();
			const mouse = new THREE.Vector2();
			mouse.x = (e.clientX - rect.left) / e.target.clientWidth * 2 - 1;
			mouse.y = -(e.clientY - rect.top) / e.target.clientHeight * 2 + 1;

			// Is the cursor on a point indicator?
			this.three.raycaster.setFromCamera(mouse, this.three.camera);
			const intersection = this.three.raycaster.intersectObjects(this.three.meshIndicators);
			if (this.three.lastIntersection && (intersection.length === 0 || intersection[0] !== this.three.lastIntersection)) {
				this.three.lastIntersection.object.material.opacity = indicatorOpacity;
				this.three.lastIntersection = undefined;
			}

			let intersectionPoint;
			if (intersection.length > 0) {
				if (intersection[0] !== this.three.lastIntersection) {
					intersection[0].object.material.opacity = indicatorOpacityHighlighted;
					this.three.lastIntersection = intersection[0];
				}
				intersectionPoint = intersection[0].object.coords;
			}

			// Show or hide the tooltip
			if (intersectionPoint) {
				this.tooltip.coord.x = intersectionPoint.x;
				this.tooltip.coord.y = intersectionPoint.y;
				this.tooltip.coord.z = intersectionPoint.z;
				this.tooltip.x = e.clientX;
				this.tooltip.y = e.clientY;
				this.tooltip.shown = true;
			} else {
				this.tooltip.shown = false;
			}
		},
		topView() {
            //console.log("topView");
			this.three.camera.position.set(0, 0, 1.5);
			this.three.camera.rotation.set(0, 0, 0);
			this.three.camera.updateProjectionMatrix();
		},

		preview() {
            //console.log("preview");
			this.ready = true;
			this.loading = false;
            this.myRender();
        }
	},
	activated() {
        //console.log("activated");
		this.isActive = true;
		this.resize();
	},
	deactivate() {
        //console.log("deactivated");
		this.isActive = false;
	},
	mounted() {
        //console.log("mounted");
        this.init();
		this.preview();
	},
	beforeDestroy() {
        //console.log("beforeDestroy");
		this.unsubscribe();

		threeInstances = threeInstances.filter(item => item !== this);
		if (this.three.renderer) {
			this.three.renderer.forceContextLoss();
			this.three.renderer = null;
		}
	},
	watch: {
		colorScheme(to) {
			if (this.three.meshGeometry) {
				setFaceColors(this.three.meshGeometry, scaleZ, to, maxVisualizationZ);
			}
			//drawLegend(this.$refs.legend, maxVisualizationZ, to);
		},
		isConnected(to) {
			if (to) {
                //console.log("isConnected");
				this.preview();
			}
		},
		language() {
			//drawLegend(this.$refs.legend, maxVisualizationZ, this.colorScheme);
		}
	}
}

</script>
