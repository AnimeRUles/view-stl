<script>
import { ref, onMounted, watch, onUnmounted } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { STLLoader } from 'three/examples/jsm/loaders/STLLoader'

export default {
    name: 'ViewStl',

    props: {
        uuid: null,
        file: null,
        rotateX: null,
        rotateY: null,
        rotateZ: null,
    },

    emits: ['ready', 'error', 'loading'],

    setup(props, ctx) {
        const viewRef = ref(null)

        let isUnmounted = false
        let rafId = null
        let isReady = false
        let isLoadEnd = false

        let scene = null
        let material = null
        let mesh = null
        let controls = null
        let renderer = null
        let camera = null
        let light = null
        let dirLight = null
        let middle = null
        let stats = null

        const init = async () => {
            const view = viewRef.value

            scene = new THREE.Scene()
            scene.background = new THREE.Color( 0x323232 )
            // scene.add(new THREE.AxesHelper(5))

            light = new THREE.AmbientLight( 0x000099, 0.1 )
            scene.add( light )
            dirLight = new THREE.DirectionalLight( 0xffff00, 0.3 )
            scene.add( dirLight )

            camera = new THREE.PerspectiveCamera(75,
                view.clientWidth / view.clientHeight,
                0.1, 1000)

            // camera.position.x = 0
            // camera.position.y = -90
            // camera.position.z = 0

            renderer = new THREE.WebGLRenderer({
                antialias: true,
                // alpha: true,
                preserveDrawingBuffer: true
            })
            renderer.outputEncoding = THREE.sRGBEncoding
            renderer.setSize(view.clientWidth, view.clientHeight)
            view.appendChild(renderer.domElement)

            controls = new OrbitControls(camera, renderer.domElement)
            controls.addEventListener('change', () => {
                light.position.copy(camera.position)
                dirLight.position.copy(camera.position)
            })
            controls.enableDamping = true
            // controls.rotateSpeed = 0.5
            // controls.dampingFactor = 0.5
            controls.enableZoom = true
            // controls.autoRotate = true
            // controls.autoRotateSpeed = .75
            // controls.enableDamping = true

            material = new THREE.MeshPhongMaterial({
                color: 0xf15533,
                specular: 100,
                shininess: 100
            })

            const render = () => {
                renderer.render(scene, camera)
            }

            const onWindowResize = () => {
                camera.aspect = view.clientWidth / view.clientHeight
                camera.updateProjectionMatrix()
                renderer.setSize(view.clientWidth, view.clientHeight)
                render()
            }

            window.addEventListener('resize', onWindowResize, false)

            // stats = new Stats()
            // view.appendChild(stats.dom)

            const animate = () => {
                controls.update()
                render()
                // stats.update()
                if (!isUnmounted) {
                    rafId = requestAnimationFrame(animate)
                    if (isLoadEnd && !isReady) {
                        isReady = true
                        ctx.emit('ready', { uuid: props.uuid })
                    }
                }
            }
            rafId = requestAnimationFrame(animate)

            load(props.file, props.rotateX, props.rotateY, props.rotateZ)
        }

        // watch(() => props.uuid, async (file) => {
        //     load(await getDataUrl(file))
        // })

        const getImg = () => {
            return renderer.domElement.toDataURL("image/png")
        }

        const getDataUrl = (file) => {
            return new Promise((resolve, reject) => {
                if (typeof file === 'string') {
                    resolve(file)
                } else if (typeof file === 'object') {
                    resolve(URL.createObjectURL(file))
                }
            })
        }

        const loader = new STLLoader()

        const load = async (file, x = null, y = null, z = null) => {
            isLoadEnd = false
            isReady = false
            loader.load(
                await getDataUrl(file),
                (geometry) => {
                    scene.remove(mesh)
                    mesh = new THREE.Mesh(geometry, material)

                    if (x !== null) {
                        mesh.rotateX(x)
                    }
                    if (y !== null) {
                        mesh.rotateY(y)
                    }
                    if (z !== null) {
                        mesh.rotateZ(z)
                    }
                    // mesh.rotateX(-45)
                    // mesh.rotateZ(45)
                    // mesh.rotateY(90)

                    scene.add(mesh)

                    middle = new THREE.Vector3()
                    geometry.computeBoundingBox()
                    geometry.boundingBox.getCenter(middle)
                    mesh.geometry.applyMatrix4(
                        new THREE.Matrix4().makeTranslation(
                            -middle.x, -middle.y, -middle.z))

                    let largestDimension = Math.max(
                        geometry.boundingBox.max.x,
                        geometry.boundingBox.max.y,
                        geometry.boundingBox.max.z)
                    camera.position.z = largestDimension * 2;

                    isLoadEnd = true
                },
                (progressEvent) => {
                    ctx.emit('loading', { uuid: props.uuid, progressEvent: progressEvent })
                    // console.log((xhr.loaded / xhr.total) * 100 + '% loaded')
                },
                (error) => {
                    console.log(error)
                    ctx.emit('error', { uuid: props.uuid, error: error })
                }
            )
        }

        onMounted(() => {
            init()
        })

        onUnmounted(() => {
            isUnmounted = true
        })

        return {
            viewRef,
            renderer,
            getImg,
            load
        }
    }
}
</script>

<template>
    <div ref="viewRef"></div>
</template>
