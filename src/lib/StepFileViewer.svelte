<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import { fade } from 'svelte/transition';
	
	let container: HTMLDivElement;
	let fileInput: HTMLInputElement;
	let scene: any;
	let camera: any;
	let renderer: any;
	let controls: any;
	let model: any;
	let animationId: number;
	
	let isLoading = false;
	let loadError = '';
	let fileName = '';
	let showDemo = false;
	let viewerReady = false;
	
	// Stats
	let modelStats = {
		vertices: 0,
		faces: 0,
		boundingBox: { x: 0, y: 0, z: 0 }
	};
	
	onMount(async () => {
		if (typeof window !== 'undefined') {
			// Load Three.js
			const script = document.createElement('script');
			script.src = 'https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js';
			script.async = true;
			document.head.appendChild(script);
			
			script.onload = async () => {
				// Load OrbitControls
				const orbitScript = document.createElement('script');
				orbitScript.textContent = `
					// OrbitControls for Three.js r128
					THREE.OrbitControls = function(object, domElement) {
						if (domElement === undefined) console.warn('THREE.OrbitControls: The second parameter "domElement" is now mandatory.');
						if (domElement === document) console.error('THREE.OrbitControls: "document" should not be used as the target "domElement". Please use "renderer.domElement" instead.');
						
						this.object = object;
						this.domElement = domElement;
						this.domElement.style.touchAction = 'none';
						
						this.enabled = true;
						this.target = new THREE.Vector3();
						
						this.minDistance = 0;
						this.maxDistance = Infinity;
						
						this.minZoom = 0;
						this.maxZoom = Infinity;
						
						this.minPolarAngle = 0;
						this.maxPolarAngle = Math.PI;
						
						this.minAzimuthAngle = -Infinity;
						this.maxAzimuthAngle = Infinity;
						
						this.enableDamping = false;
						this.dampingFactor = 0.05;
						
						this.enableZoom = true;
						this.zoomSpeed = 1.0;
						
						this.enableRotate = true;
						this.rotateSpeed = 1.0;
						
						this.enablePan = true;
						this.panSpeed = 1.0;
						this.screenSpacePanning = true;
						this.keyPanSpeed = 7.0;
						
						this.autoRotate = false;
						this.autoRotateSpeed = 2.0;
						
						this.keys = { LEFT: 'ArrowLeft', UP: 'ArrowUp', RIGHT: 'ArrowRight', BOTTOM: 'ArrowDown' };
						
						this.mouseButtons = { LEFT: THREE.MOUSE.ROTATE, MIDDLE: THREE.MOUSE.DOLLY, RIGHT: THREE.MOUSE.PAN };
						
						this.touches = { ONE: THREE.TOUCH.ROTATE, TWO: THREE.TOUCH.DOLLY_PAN };
						
						this.target0 = this.target.clone();
						this.position0 = this.object.position.clone();
						this.zoom0 = this.object.zoom;
						
						this.getPolarAngle = function() {
							return spherical.phi;
						};
						
						this.getAzimuthalAngle = function() {
							return spherical.theta;
						};
						
						this.saveState = function() {
							scope.target0.copy(scope.target);
							scope.position0.copy(scope.object.position);
							scope.zoom0 = scope.object.zoom;
						};
						
						this.reset = function() {
							scope.target.copy(scope.target0);
							scope.object.position.copy(scope.position0);
							scope.object.zoom = scope.zoom0;
							scope.object.updateProjectionMatrix();
							scope.dispatchEvent(changeEvent);
							scope.update();
							state = STATE.NONE;
						};
						
						this.update = function() {
							var offset = new THREE.Vector3();
							var quat = new THREE.Quaternion().setFromUnitVectors(object.up, new THREE.Vector3(0, 1, 0));
							var quatInverse = quat.clone().inverse();
							var lastPosition = new THREE.Vector3();
							var lastQuaternion = new THREE.Quaternion();
							
							return function update() {
								var position = scope.object.position;
								offset.copy(position).sub(scope.target);
								offset.applyQuaternion(quat);
								spherical.setFromVector3(offset);
								
								if (scope.autoRotate && state === STATE.NONE) {
									rotateLeft(getAutoRotationAngle());
								}
								
								if (scope.enableDamping) {
									spherical.theta += sphericalDelta.theta * scope.dampingFactor;
									spherical.phi += sphericalDelta.phi * scope.dampingFactor;
								} else {
									spherical.theta += sphericalDelta.theta;
									spherical.phi += sphericalDelta.phi;
								}
								
								var min = scope.minAzimuthAngle;
								var max = scope.maxAzimuthAngle;
								
								if (isFinite(min) && isFinite(max)) {
									if (min < -Math.PI) min += Math.PI * 2;
									else if (min > Math.PI) min -= Math.PI * 2;
									if (max < -Math.PI) max += Math.PI * 2;
									else if (max > Math.PI) max -= Math.PI * 2;
									if (min <= max) {
										spherical.theta = Math.max(min, Math.min(max, spherical.theta));
									} else {
										spherical.theta = (spherical.theta > (min + max) / 2) ?
											Math.max(min, spherical.theta) :
											Math.min(max, spherical.theta);
									}
								}
								
								spherical.phi = Math.max(scope.minPolarAngle, Math.min(scope.maxPolarAngle, spherical.phi));
								spherical.makeSafe();
								spherical.radius *= scale;
								spherical.radius = Math.max(scope.minDistance, Math.min(scope.maxDistance, spherical.radius));
								
								if (scope.enableDamping === true) {
									scope.target.addScaledVector(panOffset, scope.dampingFactor);
								} else {
									scope.target.add(panOffset);
								}
								
								offset.setFromSpherical(spherical);
								offset.applyQuaternion(quatInverse);
								position.copy(scope.target).add(offset);
								scope.object.lookAt(scope.target);
								
								if (scope.enableDamping === true) {
									sphericalDelta.theta *= (1 - scope.dampingFactor);
									sphericalDelta.phi *= (1 - scope.dampingFactor);
									panOffset.multiplyScalar(1 - scope.dampingFactor);
								} else {
									sphericalDelta.set(0, 0, 0);
									panOffset.set(0, 0, 0);
								}
								
								scale = 1;
								
								if (zoomChanged ||
									lastPosition.distanceToSquared(scope.object.position) > EPS ||
									8 * (1 - lastQuaternion.dot(scope.object.quaternion)) > EPS) {
									scope.dispatchEvent(changeEvent);
									lastPosition.copy(scope.object.position);
									lastQuaternion.copy(scope.object.quaternion);
									zoomChanged = false;
									return true;
								}
								return false;
							};
						}();
						
						this.dispose = function() {
							scope.domElement.removeEventListener('contextmenu', onContextMenu);
							scope.domElement.removeEventListener('pointerdown', onPointerDown);
							scope.domElement.removeEventListener('wheel', onMouseWheel);
							scope.domElement.removeEventListener('touchstart', onTouchStart);
							scope.domElement.removeEventListener('touchend', onTouchEnd);
							scope.domElement.removeEventListener('touchmove', onTouchMove);
							scope.domElement.ownerDocument.removeEventListener('pointermove', onPointerMove);
							scope.domElement.ownerDocument.removeEventListener('pointerup', onPointerUp);
							if (scope.domElement.tabIndex !== -1) scope.domElement.removeEventListener('keydown', onKeyDown);
						};
						
						var scope = this;
						var changeEvent = { type: 'change' };
						var startEvent = { type: 'start' };
						var endEvent = { type: 'end' };
						
						var STATE = {
							NONE: -1,
							ROTATE: 0,
							DOLLY: 1,
							PAN: 2,
							TOUCH_ROTATE: 3,
							TOUCH_PAN: 4,
							TOUCH_DOLLY_PAN: 5,
							TOUCH_DOLLY_ROTATE: 6
						};
						
						var state = STATE.NONE;
						var EPS = 0.000001;
						var spherical = new THREE.Spherical();
						var sphericalDelta = new THREE.Spherical();
						var scale = 1;
						var panOffset = new THREE.Vector3();
						var zoomChanged = false;
						var rotateStart = new THREE.Vector2();
						var rotateEnd = new THREE.Vector2();
						var rotateDelta = new THREE.Vector2();
						var panStart = new THREE.Vector2();
						var panEnd = new THREE.Vector2();
						var panDelta = new THREE.Vector2();
						var dollyStart = new THREE.Vector2();
						var dollyEnd = new THREE.Vector2();
						var dollyDelta = new THREE.Vector2();
						
						function getAutoRotationAngle() {
							return 2 * Math.PI / 60 / 60 * scope.autoRotateSpeed;
						}
						
						function getZoomScale() {
							return Math.pow(0.95, scope.zoomSpeed);
						}
						
						function rotateLeft(angle) {
							sphericalDelta.theta -= angle;
						}
						
						function rotateUp(angle) {
							sphericalDelta.phi -= angle;
						}
						
						var panLeft = function() {
							var v = new THREE.Vector3();
							return function panLeft(distance, objectMatrix) {
								v.setFromMatrixColumn(objectMatrix, 0);
								v.multiplyScalar(-distance);
								panOffset.add(v);
							};
						}();
						
						var panUp = function() {
							var v = new THREE.Vector3();
							return function panUp(distance, objectMatrix) {
								if (scope.screenSpacePanning === true) {
									v.setFromMatrixColumn(objectMatrix, 1);
								} else {
									v.setFromMatrixColumn(objectMatrix, 0);
									v.crossVectors(scope.object.up, v);
								}
								v.multiplyScalar(distance);
								panOffset.add(v);
							};
						}();
						
						var pan = function() {
							var offset = new THREE.Vector3();
							return function pan(deltaX, deltaY) {
								var element = scope.domElement;
								if (scope.object.isPerspectiveCamera) {
									var position = scope.object.position;
									offset.copy(position).sub(scope.target);
									var targetDistance = offset.length();
									targetDistance *= Math.tan((scope.object.fov / 2) * Math.PI / 180.0);
									panLeft(2 * deltaX * targetDistance / element.clientHeight, scope.object.matrix);
									panUp(2 * deltaY * targetDistance / element.clientHeight, scope.object.matrix);
								} else if (scope.object.isOrthographicCamera) {
									panLeft(deltaX * (scope.object.right - scope.object.left) / scope.object.zoom / element.clientWidth, scope.object.matrix);
									panUp(deltaY * (scope.object.top - scope.object.bottom) / scope.object.zoom / element.clientHeight, scope.object.matrix);
								} else {
									console.warn('WARNING: OrbitControls.js encountered an unknown camera type - pan disabled.');
									scope.enablePan = false;
								}
							};
						}();
						
						function dollyIn(dollyScale) {
							if (scope.object.isPerspectiveCamera) {
								scale /= dollyScale;
							} else if (scope.object.isOrthographicCamera) {
								scope.object.zoom = Math.max(scope.minZoom, Math.min(scope.maxZoom, scope.object.zoom * dollyScale));
								scope.object.updateProjectionMatrix();
								zoomChanged = true;
							} else {
								console.warn('WARNING: OrbitControls.js encountered an unknown camera type - dolly/zoom disabled.');
								scope.enableZoom = false;
							}
						}
						
						function dollyOut(dollyScale) {
							if (scope.object.isPerspectiveCamera) {
								scale *= dollyScale;
							} else if (scope.object.isOrthographicCamera) {
								scope.object.zoom = Math.max(scope.minZoom, Math.min(scope.maxZoom, scope.object.zoom / dollyScale));
								scope.object.updateProjectionMatrix();
								zoomChanged = true;
							} else {
								console.warn('WARNING: OrbitControls.js encountered an unknown camera type - dolly/zoom disabled.');
								scope.enableZoom = false;
							}
						}
						
						function handleMouseDownRotate(event) {
							rotateStart.set(event.clientX, event.clientY);
						}
						
						function handleMouseDownDolly(event) {
							dollyStart.set(event.clientX, event.clientY);
						}
						
						function handleMouseDownPan(event) {
							panStart.set(event.clientX, event.clientY);
						}
						
						function handleMouseMoveRotate(event) {
							rotateEnd.set(event.clientX, event.clientY);
							rotateDelta.subVectors(rotateEnd, rotateStart).multiplyScalar(scope.rotateSpeed);
							var element = scope.domElement;
							rotateLeft(2 * Math.PI * rotateDelta.x / element.clientHeight);
							rotateUp(2 * Math.PI * rotateDelta.y / element.clientHeight);
							rotateStart.copy(rotateEnd);
							scope.update();
						}
						
						function handleMouseMoveDolly(event) {
							dollyEnd.set(event.clientX, event.clientY);
							dollyDelta.subVectors(dollyEnd, dollyStart);
							if (dollyDelta.y > 0) {
								dollyIn(getZoomScale());
							} else if (dollyDelta.y < 0) {
								dollyOut(getZoomScale());
							}
							dollyStart.copy(dollyEnd);
							scope.update();
						}
						
						function handleMouseMovePan(event) {
							panEnd.set(event.clientX, event.clientY);
							panDelta.subVectors(panEnd, panStart).multiplyScalar(scope.panSpeed);
							pan(panDelta.x, panDelta.y);
							panStart.copy(panEnd);
							scope.update();
						}
						
						function handleMouseWheel(event) {
							if (event.deltaY < 0) {
								dollyOut(getZoomScale());
							} else if (event.deltaY > 0) {
								dollyIn(getZoomScale());
							}
							scope.update();
						}
						
						function handleKeyDown(event) {
							var needsUpdate = false;
							switch (event.code) {
								case scope.keys.UP:
									pan(0, scope.keyPanSpeed);
									needsUpdate = true;
									break;
								case scope.keys.BOTTOM:
									pan(0, -scope.keyPanSpeed);
									needsUpdate = true;
									break;
								case scope.keys.LEFT:
									pan(scope.keyPanSpeed, 0);
									needsUpdate = true;
									break;
								case scope.keys.RIGHT:
									pan(-scope.keyPanSpeed, 0);
									needsUpdate = true;
									break;
							}
							if (needsUpdate) {
								event.preventDefault();
								scope.update();
							}
						}
						
						function handleTouchStartRotate() {
							if (pointers.length === 1) {
								rotateStart.set(pointers[0].pageX, pointers[0].pageY);
							} else {
								var x = 0.5 * (pointers[0].pageX + pointers[1].pageX);
								var y = 0.5 * (pointers[0].pageY + pointers[1].pageY);
								rotateStart.set(x, y);
							}
						}
						
						function handleTouchStartPan() {
							if (pointers.length === 1) {
								panStart.set(pointers[0].pageX, pointers[0].pageY);
							} else {
								var x = 0.5 * (pointers[0].pageX + pointers[1].pageX);
								var y = 0.5 * (pointers[0].pageY + pointers[1].pageY);
								panStart.set(x, y);
							}
						}
						
						function handleTouchStartDolly() {
							var dx = pointers[0].pageX - pointers[1].pageX;
							var dy = pointers[0].pageY - pointers[1].pageY;
							var distance = Math.sqrt(dx * dx + dy * dy);
							dollyStart.set(0, distance);
						}
						
						function handleTouchStartDollyPan() {
							if (scope.enableZoom) handleTouchStartDolly();
							if (scope.enablePan) handleTouchStartPan();
						}
						
						function handleTouchStartDollyRotate() {
							if (scope.enableZoom) handleTouchStartDolly();
							if (scope.enableRotate) handleTouchStartRotate();
						}
						
						function handleTouchMoveRotate(event) {
							if (pointers.length == 1) {
								rotateEnd.set(event.pageX, event.pageY);
							} else {
								var position = getSecondPointerPosition(event);
								var x = 0.5 * (event.pageX + position.x);
								var y = 0.5 * (event.pageY + position.y);
								rotateEnd.set(x, y);
							}
							rotateDelta.subVectors(rotateEnd, rotateStart).multiplyScalar(scope.rotateSpeed);
							var element = scope.domElement;
							rotateLeft(2 * Math.PI * rotateDelta.x / element.clientHeight);
							rotateUp(2 * Math.PI * rotateDelta.y / element.clientHeight);
							rotateStart.copy(rotateEnd);
						}
						
						function handleTouchMovePan(event) {
							if (pointers.length === 1) {
								panEnd.set(event.pageX, event.pageY);
							} else {
								var position = getSecondPointerPosition(event);
								var x = 0.5 * (event.pageX + position.x);
								var y = 0.5 * (event.pageY + position.y);
								panEnd.set(x, y);
							}
							panDelta.subVectors(panEnd, panStart).multiplyScalar(scope.panSpeed);
							pan(panDelta.x, panDelta.y);
							panStart.copy(panEnd);
						}
						
						function handleTouchMoveDolly(event) {
							var position = getSecondPointerPosition(event);
							var dx = event.pageX - position.x;
							var dy = event.pageY - position.y;
							var distance = Math.sqrt(dx * dx + dy * dy);
							dollyEnd.set(0, distance);
							dollyDelta.set(0, Math.pow(dollyEnd.y / dollyStart.y, scope.zoomSpeed));
							dollyIn(dollyDelta.y);
							dollyStart.copy(dollyEnd);
						}
						
						function handleTouchMoveDollyPan(event) {
							if (scope.enableZoom) handleTouchMoveDolly(event);
							if (scope.enablePan) handleTouchMovePan(event);
						}
						
						function handleTouchMoveDollyRotate(event) {
							if (scope.enableZoom) handleTouchMoveDolly(event);
							if (scope.enableRotate) handleTouchMoveRotate(event);
						}
						
						function onPointerDown(event) {
							if (scope.enabled === false) return;
							if (pointers.length === 0) {
								scope.domElement.setPointerCapture(event.pointerId);
								scope.domElement.addEventListener('pointermove', onPointerMove);
								scope.domElement.addEventListener('pointerup', onPointerUp);
							}
							addPointer(event);
							if (event.pointerType === 'touch') {
								onTouchStart(event);
							} else {
								onMouseDown(event);
							}
						}
						
						function onPointerMove(event) {
							if (scope.enabled === false) return;
							if (event.pointerType === 'touch') {
								onTouchMove(event);
							} else {
								onMouseMove(event);
							}
						}
						
						function onPointerUp(event) {
							removePointer(event);
							if (pointers.length === 0) {
								scope.domElement.releasePointerCapture(event.pointerId);
								scope.domElement.removeEventListener('pointermove', onPointerMove);
								scope.domElement.removeEventListener('pointerup', onPointerUp);
							}
							scope.dispatchEvent(endEvent);
							state = STATE.NONE;
						}
						
						function onMouseDown(event) {
							var mouseAction;
							switch (event.button) {
								case 0:
									mouseAction = scope.mouseButtons.LEFT;
									break;
								case 1:
									mouseAction = scope.mouseButtons.MIDDLE;
									break;
								case 2:
									mouseAction = scope.mouseButtons.RIGHT;
									break;
								default:
									mouseAction = -1;
							}
							switch (mouseAction) {
								case THREE.MOUSE.DOLLY:
									if (scope.enableZoom === false) return;
									handleMouseDownDolly(event);
									state = STATE.DOLLY;
									break;
								case THREE.MOUSE.ROTATE:
									if (event.ctrlKey || event.metaKey || event.shiftKey) {
										if (scope.enablePan === false) return;
										handleMouseDownPan(event);
										state = STATE.PAN;
									} else {
										if (scope.enableRotate === false) return;
										handleMouseDownRotate(event);
										state = STATE.ROTATE;
									}
									break;
								case THREE.MOUSE.PAN:
									if (event.ctrlKey || event.metaKey || event.shiftKey) {
										if (scope.enableRotate === false) return;
										handleMouseDownRotate(event);
										state = STATE.ROTATE;
									} else {
										if (scope.enablePan === false) return;
										handleMouseDownPan(event);
										state = STATE.PAN;
									}
									break;
								default:
									state = STATE.NONE;
							}
							if (state !== STATE.NONE) {
								scope.dispatchEvent(startEvent);
							}
						}
						
						function onMouseMove(event) {
							if (scope.enabled === false) return;
							event.preventDefault();
							switch (state) {
								case STATE.ROTATE:
									if (scope.enableRotate === false) return;
									handleMouseMoveRotate(event);
									break;
								case STATE.DOLLY:
									if (scope.enableZoom === false) return;
									handleMouseMoveDolly(event);
									break;
								case STATE.PAN:
									if (scope.enablePan === false) return;
									handleMouseMovePan(event);
									break;
							}
						}
						
						function onMouseWheel(event) {
							if (scope.enabled === false || scope.enableZoom === false || (state !== STATE.NONE && state !== STATE.ROTATE)) return;
							event.preventDefault();
							scope.dispatchEvent(startEvent);
							handleMouseWheel(event);
							scope.dispatchEvent(endEvent);
						}
						
						function onKeyDown(event) {
							if (scope.enabled === false || scope.enablePan === false) return;
							handleKeyDown(event);
						}
						
						function onTouchStart(event) {
							trackPointer(event);
							switch (pointers.length) {
								case 1:
									switch (scope.touches.ONE) {
										case THREE.TOUCH.ROTATE:
											if (scope.enableRotate === false) return;
											handleTouchStartRotate();
											state = STATE.TOUCH_ROTATE;
											break;
										case THREE.TOUCH.PAN:
											if (scope.enablePan === false) return;
											handleTouchStartPan();
											state = STATE.TOUCH_PAN;
											break;
										default:
											state = STATE.NONE;
									}
									break;
								case 2:
									switch (scope.touches.TWO) {
										case THREE.TOUCH.DOLLY_PAN:
											if (scope.enableZoom === false && scope.enablePan === false) return;
											handleTouchStartDollyPan();
											state = STATE.TOUCH_DOLLY_PAN;
											break;
										case THREE.TOUCH.DOLLY_ROTATE:
											if (scope.enableZoom === false && scope.enableRotate === false) return;
											handleTouchStartDollyRotate();
											state = STATE.TOUCH_DOLLY_ROTATE;
											break;
										default:
											state = STATE.NONE;
									}
									break;
								default:
									state = STATE.NONE;
							}
							if (state !== STATE.NONE) {
								scope.dispatchEvent(startEvent);
							}
						}
						
						function onTouchMove(event) {
							trackPointer(event);
							switch (state) {
								case STATE.TOUCH_ROTATE:
									if (scope.enableRotate === false) return;
									handleTouchMoveRotate(event);
									scope.update();
									break;
								case STATE.TOUCH_PAN:
									if (scope.enablePan === false) return;
									handleTouchMovePan(event);
									scope.update();
									break;
								case STATE.TOUCH_DOLLY_PAN:
									if (scope.enableZoom === false && scope.enablePan === false) return;
									handleTouchMoveDollyPan(event);
									scope.update();
									break;
								case STATE.TOUCH_DOLLY_ROTATE:
									if (scope.enableZoom === false && scope.enableRotate === false) return;
									handleTouchMoveDollyRotate(event);
									scope.update();
									break;
								default:
									state = STATE.NONE;
							}
						}
						
						function onTouchEnd(event) {
							if (scope.enabled === false) return;
							scope.dispatchEvent(endEvent);
							state = STATE.NONE;
						}
						
						function onContextMenu(event) {
							if (scope.enabled === false) return;
							event.preventDefault();
						}
						
						var pointers = [];
						var pointerPositions = {};
						
						function addPointer(event) {
							pointers.push(event);
						}
						
						function removePointer(event) {
							delete pointerPositions[event.pointerId];
							for (var i = 0; i < pointers.length; i++) {
								if (pointers[i].pointerId == event.pointerId) {
									pointers.splice(i, 1);
									return;
								}
							}
						}
						
						function trackPointer(event) {
							var position = pointerPositions[event.pointerId];
							if (position === undefined) {
								position = new THREE.Vector2();
								pointerPositions[event.pointerId] = position;
							}
							position.set(event.pageX, event.pageY);
						}
						
						function getSecondPointerPosition(event) {
							var pointer = (event.pointerId === pointers[0].pointerId) ? pointers[1] : pointers[0];
							return pointerPositions[pointer.pointerId];
						}
						
						scope.domElement.addEventListener('contextmenu', onContextMenu);
						scope.domElement.addEventListener('pointerdown', onPointerDown);
						scope.domElement.addEventListener('wheel', onMouseWheel);
						scope.domElement.addEventListener('touchstart', onTouchStart);
						scope.domElement.addEventListener('touchend', onTouchEnd);
						scope.domElement.addEventListener('touchmove', onTouchMove);
						if (scope.domElement.tabIndex === -1) scope.domElement.tabIndex = 0;
						scope.domElement.addEventListener('keydown', onKeyDown);
						
						this.update();
					};
					
					THREE.OrbitControls.prototype = Object.create(THREE.EventDispatcher.prototype);
					THREE.OrbitControls.prototype.constructor = THREE.OrbitControls;
				`;
				document.head.appendChild(orbitScript);
				
				// Initialize Three.js scene
				initScene();
				viewerReady = true;
			};
		}
	});
	
	onDestroy(() => {
		if (animationId) {
			cancelAnimationFrame(animationId);
		}
		if (renderer) {
			renderer.dispose();
		}
	});
	
	function initScene() {
		// @ts-ignore
		const THREE = window.THREE;
		
		// Scene setup
		scene = new THREE.Scene();
		scene.background = new THREE.Color(0xf5f5f5);
		
		// Camera setup
		camera = new THREE.PerspectiveCamera(
			75,
			container.clientWidth / container.clientHeight,
			0.1,
			1000
		);
		camera.position.set(5, 5, 5);
		
		// Renderer setup
		renderer = new THREE.WebGLRenderer({ antialias: true });
		renderer.setSize(container.clientWidth, container.clientHeight);
		renderer.shadowMap.enabled = true;
		renderer.shadowMap.type = THREE.PCFSoftShadowMap;
		container.appendChild(renderer.domElement);
		
		// Controls setup
		controls = new THREE.OrbitControls(camera, renderer.domElement);
		controls.enableDamping = true;
		controls.dampingFactor = 0.05;
		controls.screenSpacePanning = false;
		controls.minDistance = 1;
		controls.maxDistance = 50;
		controls.maxPolarAngle = Math.PI / 2;
		
		// Lighting setup
		const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
		scene.add(ambientLight);
		
		const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
		directionalLight.position.set(10, 10, 5);
		directionalLight.castShadow = true;
		directionalLight.shadow.camera.near = 0.1;
		directionalLight.shadow.camera.far = 50;
		directionalLight.shadow.camera.left = -10;
		directionalLight.shadow.camera.right = 10;
		directionalLight.shadow.camera.top = 10;
		directionalLight.shadow.camera.bottom = -10;
		scene.add(directionalLight);
		
		// Grid helper
		const gridHelper = new THREE.GridHelper(10, 10, 0x888888, 0xcccccc);
		scene.add(gridHelper);
		
		// Axes helper
		const axesHelper = new THREE.AxesHelper(2);
		scene.add(axesHelper);
		
		// Start animation loop
		animate();
		
		// Handle window resize
		window.addEventListener('resize', onWindowResize);
	}
	
	function onWindowResize() {
		if (!camera || !renderer || !container) return;
		
		camera.aspect = container.clientWidth / container.clientHeight;
		camera.updateProjectionMatrix();
		renderer.setSize(container.clientWidth, container.clientHeight);
	}
	
	function animate() {
		animationId = requestAnimationFrame(animate);
		
		if (controls) {
			controls.update();
		}
		
		if (renderer && scene && camera) {
			renderer.render(scene, camera);
		}
	}
	
	function handleFileUpload(event: Event) {
		const input = event.target as HTMLInputElement;
		const file = input.files?.[0];
		
		if (file) {
			fileName = file.name;
			loadSTEPFile(file);
		}
	}
	
	async function loadSTEPFile(file: File) {
		isLoading = true;
		loadError = '';
		
		try {
			// For demo purposes, we'll create a simple geometry
			// In a real implementation, you would parse the STEP file
			await createDemoGeometry();
			
		} catch (error) {
			loadError = 'Error loading file. Please ensure it\'s a valid STEP file.';
			console.error('Load error:', error);
		} finally {
			isLoading = false;
		}
	}
	
	async function loadDemoFile() {
		showDemo = true;
		fileName = 'demo-bracket.step';
		isLoading = true;
		loadError = '';
		
		try {
			await createDemoGeometry();
		} catch (error) {
			loadError = 'Error loading demo file.';
			console.error('Demo load error:', error);
		} finally {
			isLoading = false;
		}
	}
	
	async function createDemoGeometry() {
		// @ts-ignore
		const THREE = window.THREE;
		
		// Remove existing model
		if (model) {
			scene.remove(model);
			model.geometry.dispose();
			if (model.material) {
				if (Array.isArray(model.material)) {
					model.material.forEach((mat: any) => mat.dispose());
				} else {
					model.material.dispose();
				}
			}
		}
		
		// Create a demo mechanical part (bracket-like shape)
		const shape = new THREE.Shape();
		shape.moveTo(0, 0);
		shape.lineTo(4, 0);
		shape.lineTo(4, 1);
		shape.lineTo(3, 1);
		shape.lineTo(3, 3);
		shape.lineTo(4, 3);
		shape.lineTo(4, 4);
		shape.lineTo(0, 4);
		shape.lineTo(0, 3);
		shape.lineTo(1, 3);
		shape.lineTo(1, 1);
		shape.lineTo(0, 1);
		shape.closePath();
		
		// Create holes
		const hole1 = new THREE.Path();
		hole1.moveTo(0.5, 3.5);
		hole1.absarc(0.5, 3.5, 0.25, 0, Math.PI * 2, true);
		shape.holes.push(hole1);
		
		const hole2 = new THREE.Path();
		hole2.moveTo(3.5, 3.5);
		hole2.absarc(3.5, 3.5, 0.25, 0, Math.PI * 2, true);
		shape.holes.push(hole2);
		
		const hole3 = new THREE.Path();
		hole3.moveTo(2, 0.5);
		hole3.absarc(2, 0.5, 0.25, 0, Math.PI * 2, true);
		shape.holes.push(hole3);
		
		// Extrude settings
		const extrudeSettings = {
			steps: 2,
			depth: 0.5,
			bevelEnabled: true,
			bevelThickness: 0.05,
			bevelSize: 0.05,
			bevelOffset: 0,
			bevelSegments: 2
		};
		
		const geometry = new THREE.ExtrudeGeometry(shape, extrudeSettings);
		
		// Create material
		const material = new THREE.MeshPhongMaterial({
			color: 0x2d5a2d,
			specular: 0x111111,
			shininess: 200,
			side: THREE.DoubleSide
		});
		
		// Create mesh
		model = new THREE.Mesh(geometry, material);
		model.castShadow = true;
		model.receiveShadow = true;
		
		// Center the geometry
		geometry.computeBoundingBox();
		const box = geometry.boundingBox;
		const center = new THREE.Vector3();
		box.getCenter(center);
		geometry.translate(-center.x, -center.y, -center.z);
		
		// Add to scene
		scene.add(model);
		
		// Update camera position
		const size = new THREE.Vector3();
		box.getSize(size);
		const maxDim = Math.max(size.x, size.y, size.z);
		const fov = camera.fov * (Math.PI / 180);
		let cameraZ = Math.abs(maxDim / 2 / Math.tan(fov / 2));
		cameraZ *= 2; // Zoom out a bit
		
		camera.position.set(cameraZ, cameraZ, cameraZ);
		camera.lookAt(0, 0, 0);
		
		// Update stats
		modelStats = {
			vertices: geometry.attributes.position.count,
			faces: geometry.index ? geometry.index.count / 3 : geometry.attributes.position.count / 3,
			boundingBox: {
				x: parseFloat(size.x.toFixed(2)),
				y: parseFloat(size.y.toFixed(2)),
				z: parseFloat(size.z.toFixed(2))
			}
		};
	}
	
	function resetView() {
		if (!camera || !controls) return;
		
		camera.position.set(5, 5, 5);
		camera.lookAt(0, 0, 0);
		controls.target.set(0, 0, 0);
		controls.update();
	}
	
	function clearModel() {
		if (model) {
			scene.remove(model);
			model.geometry.dispose();
			if (model.material) {
				if (Array.isArray(model.material)) {
					model.material.forEach((mat: any) => mat.dispose());
				} else {
					model.material.dispose();
				}
			}
			model = null;
		}
		
		fileName = '';
		showDemo = false;
		modelStats = {
			vertices: 0,
			faces: 0,
			boundingBox: { x: 0, y: 0, z: 0 }
		};
		
		if (fileInput) {
			fileInput.value = '';
		}
	}
</script>

<div class="max-w-7xl mx-auto">
	<div class="mb-8">
		<h2 class="text-3xl font-bold text-gray-900 mb-2">STEP File Viewer</h2>
		<p class="text-gray-600">Upload and view 3D CAD models with full rotation and zoom controls</p>
		{#if !viewerReady}
			<p class="text-sm text-amber-600 mt-2">⚠️ 3D viewer loading...</p>
		{/if}
	</div>
	
	<div class="grid grid-cols-1 lg:grid-cols-3 gap-8">
		<!-- Controls Panel -->
		<div class="lg:col-span-1 space-y-6">
			<!-- Upload Section -->
			<div class="bg-white border-2 border-gray-300 rounded-lg p-6">
				<h3 class="text-lg font-semibold text-gray-900 mb-4">File Upload</h3>
				
				<input
					type="file"
					accept=".step,.stp,.STEP,.STP"
					on:change={handleFileUpload}
					bind:this={fileInput}
					disabled={!viewerReady}
					class="hidden"
					id="step-upload"
				/>
				
				<label
					for="step-upload"
					class="block w-full text-center px-4 py-3 bg-white border-2 border-dashed border-gray-300 rounded-lg cursor-pointer hover:border-logo-green transition-colors {!viewerReady ? 'opacity-50 cursor-not-allowed' : ''}"
				>
					<svg class="w-8 h-8 text-gray-400 mx-auto mb-2" fill="none" viewBox="0 0 24 24" stroke="currentColor">
						<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12" />
					</svg>
					<p class="text-sm text-gray-600">Upload STEP file</p>
					<p class="text-xs text-gray-400 mt-1">.step or .stp files</p>
				</label>
				
				{#if fileName}
					<div class="mt-3 p-2 bg-gray-50 rounded text-sm text-gray-700 flex items-center justify-between">
						<span class="truncate">{fileName}</span>
						<button
							on:click={clearModel}
							class="ml-2 text-gray-500 hover:text-red-600"
							aria-label="Remove file"
						>
							<svg class="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
								<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
							</svg>
						</button>
					</div>
				{/if}
				
				<div class="mt-4 text-center">
					<button
						on:click={loadDemoFile}
						disabled={!viewerReady || showDemo}
						class="text-sm text-logo-green hover:text-green-700 disabled:text-gray-400 disabled:cursor-not-allowed transition-colors"
					>
						{showDemo ? 'Demo loaded' : 'Load demo file'}
					</button>
				</div>
			</div>
			
			<!-- Model Stats -->
			{#if model && modelStats.vertices > 0}
				<div class="bg-gray-50 border border-gray-200 rounded-lg p-6" transition:fade>
					<h3 class="text-lg font-semibold text-gray-900 mb-4">Model Information</h3>
					<dl class="space-y-3 text-sm">
						<div class="flex justify-between">
							<dt class="text-gray-600">Vertices:</dt>
							<dd class="font-mono text-gray-900">{modelStats.vertices.toLocaleString()}</dd>
						</div>
						<div class="flex justify-between">
							<dt class="text-gray-600">Faces:</dt>
							<dd class="font-mono text-gray-900">{modelStats.faces.toLocaleString()}</dd>
						</div>
						<div class="border-t pt-3">
							<dt class="text-gray-600 mb-2">Bounding Box:</dt>
							<dd class="font-mono text-gray-900 text-xs">
								X: {modelStats.boundingBox.x} units<br>
								Y: {modelStats.boundingBox.y} units<br>
								Z: {modelStats.boundingBox.z} units
							</dd>
						</div>
					</dl>
				</div>
			{/if}
			
			<!-- View Controls -->
			<div class="bg-blue-50 border border-blue-200 rounded-lg p-6">
				<h3 class="text-lg font-semibold text-gray-900 mb-4">View Controls</h3>
				<ul class="space-y-2 text-sm text-gray-700">
					<li class="flex items-start">
						<svg class="w-5 h-5 text-blue-600 mr-2 mt-0.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
							<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 15l-2 5L9 9l11 4-5 2zm0 0l5 5M7.188 2.239l.777 2.897M5.136 7.965l-2.898-.777M13.95 4.05l-2.122 2.122m-5.657 5.656l-2.12 2.122" />
						</svg>
						<span><strong>Left click + drag:</strong> Rotate view</span>
					</li>
					<li class="flex items-start">
						<svg class="w-5 h-5 text-blue-600 mr-2 mt-0.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
							<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 9l4-4 4 4m0 6l-4 4-4-4" />
						</svg>
						<span><strong>Right click + drag:</strong> Pan view</span>
					</li>
					<li class="flex items-start">
						<svg class="w-5 h-5 text-blue-600 mr-2 mt-0.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
							<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0zM10 7v3m0 0v3m0-3h3m-3 0H7" />
						</svg>
						<span><strong>Scroll wheel:</strong> Zoom in/out</span>
					</li>
					<li class="flex items-start">
						<svg class="w-5 h-5 text-blue-600 mr-2 mt-0.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
							<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 11.5V14m0-2.5v-6a1.5 1.5 0 113 0m-3 6a1.5 1.5 0 00-3 0v2a7.5 7.5 0 0015 0v-5a1.5 1.5 0 00-3 0m-6-3V11m0-5.5v-1a1.5 1.5 0 013 0v1m0 0V11m0-5.5a1.5 1.5 0 013 0v3m0 0V11" />
						</svg>
						<span><strong>Touch:</strong> Pinch to zoom</span>
					</li>
				</ul>
				
				{#if model}
					<button
						on:click={resetView}
						class="mt-4 w-full px-4 py-2 bg-blue-600 hover:bg-blue-700 text-white rounded-lg transition-colors text-sm font-medium"
					>
						Reset View
					</button>
				{/if}
			</div>
		</div>
		
		<!-- 3D Viewer -->
		<div class="lg:col-span-2">
			<div class="bg-white border-2 border-gray-300 rounded-lg overflow-hidden relative" style="height: 600px;">
				<div bind:this={container} class="w-full h-full">
					{#if !viewerReady}
						<div class="absolute inset-0 flex items-center justify-center bg-gray-50">
							<div class="text-center">
								<div class="w-12 h-12 border-4 border-logo-green border-t-transparent rounded-full animate-spin mx-auto mb-4"></div>
								<p class="text-gray-600">Initializing 3D viewer...</p>
							</div>
						</div>
					{:else if !model && !isLoading}
						<div class="absolute inset-0 flex items-center justify-center bg-gray-50">
							<div class="text-center">
								<svg class="w-24 h-24 text-gray-300 mx-auto mb-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
									<path stroke-linecap="round" stroke-linejoin="round" stroke-width="1" d="M20 7l-8-4-8 4m16 0l-8 4m8-4v10l-8 4m0-10L4 7m8 4v10M4 7v10l8 4" />
								</svg>
								<h3 class="text-lg font-medium text-gray-900 mb-2">No Model Loaded</h3>
								<p class="text-sm text-gray-600 mb-4">Upload a STEP file or load the demo</p>
							</div>
						</div>
					{/if}
					
					{#if isLoading}
						<div class="absolute inset-0 bg-black bg-opacity-30 flex items-center justify-center">
							<div class="bg-white rounded-lg p-6 shadow-xl">
								<div class="w-8 h-8 border-3 border-logo-green border-t-transparent rounded-full animate-spin mx-auto mb-3"></div>
								<p class="text-gray-700">Loading model...</p>
							</div>
						</div>
					{/if}
					
					{#if loadError}
						<div class="absolute bottom-4 left-4 right-4 bg-red-50 border border-red-200 rounded-lg p-4">
							<p class="text-sm text-red-700">{loadError}</p>
						</div>
					{/if}
				</div>
			</div>
			
			<!-- Info Box -->
			<div class="mt-4 p-4 bg-amber-50 border border-amber-200 rounded-lg">
				<p class="text-sm text-amber-800">
					<strong>Note:</strong> This is a demonstration viewer. In production, full STEP file parsing would be implemented to handle complex CAD models from various CAD software.
				</p>
			</div>
		</div>
	</div>
</div>