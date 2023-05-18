<script>
import { ref, onBeforeUpdate, reactive, onMounted, nextTick } from 'vue'
import { forEach, toLower, size } from 'lodash'
import { v4 as uuidv4 } from 'uuid';
import ViewStl from './components/ViewStl.vue'

export default {
    props: {},

    components: {
        ViewStl
    },

    setup(props) {
        const viewList = ref([])
        const viewScan = ref(null)
        const viewScanRef = ref(null)
        const viewRef = ref(null)

        const changeView = (uuid, data, dataJoin) => {
            forEach(viewList.value, (viewItem, key) => {
                if (viewItem.uuid !== uuid) return

                const viewNew = { ...viewItem, ...data }

                if (dataJoin) {
                    forEach(dataJoin, (value, name) => {
                        if (Array.isArray(value)) {
                            viewNew[name] = [...viewNew[name], value]
                        } else {
                            viewNew[name] = value
                        }
                    })
                }

                viewList.value[key] = viewNew

                return false
            })
            viewList.value = [...viewList.value]
        }

        const handleFileUpload = (event) => {
            if (!event.target.files.length) return

            const viewListTemp = []

            forEach(event.target.files, (file) => {
                const ext = file.name.split('.').pop()
                const isStl = (toLower(ext) === 'stl')

                if (!isStl) return

                const view = {
                    uuid: uuidv4(),
                    file: file,
                    path: file.webkitRelativePath,
                    ext: ext,
                    isStl,
                    isShow3d: false,
                    img: null,
                    imgList: [],
                    rotateX: -45,
                    rotateY: null,
                    rotateZ: 45,
                    error: null,

                }
                viewListTemp.push(view)
            })

            viewList.value = viewListTemp

            startScan()
        }

        const hide3dAll = () => {
            forEach(viewList.value, (viewItem) => {
                viewItem.isShow3d = false
            })
            viewList.value = [...viewList.value]
        }

        const hide3d = (view) => {
            changeView(view.uuid, { img: viewRef.value[0].getImg() })

            hide3dAll()
        }

        const show3d = (view) => {
            hide3dAll()

            view.isShow3d = true
        }

        const startScan = async() => {
            let viewCurrent = null
            forEach(viewList.value, (view) => {
                if (view.img || view.error || !view.isStl) return

                viewCurrent = view

                return false
            })

            viewScan.value = viewCurrent
        }

        const reloadScan = (view) => {
            changeView(view.uuid, { error: null })
            viewScan.value = view
        }

        const loadingScan = ({ uuid, progressEvent }) => {
            // changeView(uuid, { error: error })
        }

        const readyScan = ({ uuid }) => {
            changeView(uuid, { img: viewScanRef.value.getImg() })

            startScan()

            // changeView(uuid, null, { imgList: [viewScanRef.value.getImg()] })
            // await viewScanRef.value.load(viewScan.value.file, -45, 0, 45)
        }

        const rotateScan = ({ uuid }) => {
            changeView(uuid, { img: viewScanRef.value.getImg() })

            startScan()
            // changeView(uuid, null, { imgList: [viewScanRef.value.getImg()] })
        }

        const setErrorScan = ({ uuid, error }) => {
            changeView(uuid, { error: error })

            startScan()
        }

        if (import.meta.env.DEV) {
            (() => {
                viewList.value.push({
                    uuid: uuidv4(),
                    file: './Bone_Giant_01.stl',
                    path: './Bone_Giant_01.stl',
                    ext: 'stl',
                    isStl: true,
                    isShow3d: false,
                    img: null,
                    imgList: [],
                    rotateX: -45,
                    rotateY: null,
                    rotateZ: 45,
                    error: null
                })
                viewList.value.push({
                    uuid: uuidv4(),
                    file: './FloatingTree_A_RockL.stl',
                    path: './FloatingTree_A_RockL.stl',
                    ext: 'stl',
                    isStl: true,
                    isShow3d: false,
                    img: null,
                    imgList: [],
                    rotateX: -45,
                    rotateY: null,
                    rotateZ: 45,
                    error: null
                })
                // viewList.value.push({
                //     uuid: uuidv4(),
                //     file: './FloatingTree_A_RockMid.stl',
                //     path: './FloatingTree_A_RockMid.stl',
                //     ext: 'stl',
                //     isStl: true,
                //     isShow3d: false,
                //     img: null,
                //     imgList: [],
                //     error: null
                // })
                // viewList.value.push({
                //     uuid: uuidv4(),
                //     file: './FloatingTree_A_RockR.stl',
                //     path: './FloatingTree_A_RockR.stl',
                //     ext: 'stl',
                //     isStl: true,
                //     isShow3d: false,
                //     img: null,
                //     imgList: [],
                //     error: null
                // })
                startScan()
            })()
        }

        return {
            viewList,
            viewScan,
            viewScanRef,
            viewRef,
            handleFileUpload,
            hide3d,
            show3d,
            setErrorScan,
            loadingScan,
            reloadScan,
            readyScan,
            rotateScan
        }
    }
}
</script>

<template>
    <ViewStl v-if="viewScan"
             style="position: absolute; left: -1000px; top: -1000px"
             ref="viewScanRef"
             :key="viewScan.uuid"
             :uuid="viewScan.uuid"
             :file="viewScan.file"
             :rotateX="viewScan.rotateX"
             :rotateY="viewScan.rotateY"
             :rotateZ="viewScan.rotateZ"
             class="view-scan"
             @ready="readyScan"
             @error="setErrorScan"/>

    <div style="margin: 20px 0; position: sticky; top: 0">
        <input type="file"
               multiple="multiple"
               webkitdirectory mozdirectory msdirectory odirectory directory
               @change="handleFileUpload">
    </div>

    <div class="view-list">
        <div v-for="view in viewList"
             :key="view.uuid"
             class="view-item"
        >
            <div style="display: flex; gap: 10px; flex-direction: column">
                <div>{{ view.path }}</div>

                <div>
                    <template v-if="view.error">
                        <div class="button"
                             @click="reloadScan(view)"
                        >Перезагрузить</div>
                    </template>

                    <template v-else>
                        <div v-if="view.isShow3d"
                             class="button"
                             @click="hide3d(view)"
                        >Скрыть 3d</div>

                        <div v-else
                             class="button"
                             @click="show3d(view)"
                        >Показать 3d</div>
                    </template>
                </div>
            </div>

            <ViewStl v-if="view.isShow3d && view.isStl"
                     ref="viewRef"
                     :uuid="view.uuid"
                     :file="view.file"
                     :rotateX="view.rotateX"
                     :rotateY="view.rotateY"
                     :rotateZ="view.rotateZ"
                     class="view-item-stl"
                     :class="{
                         show3d: view.isShow3d,
                         error: view.error,
                     }"/>

            <img v-else-if="view.img"
                 :src="view.img"
                 class="view-item-stl"/>

            <div v-else
                 class="view-item-stl"
                 :class="{error: view.error}"></div>
        </div>
    </div>
</template>

<style lang="scss">
.view-scan {
    border: 1px solid orange;
    width: 500px;
    height: 500px;
}

.view-list {
    display : flex;
    flex-direction: column;
    gap: 10px;

    .view-item {
        padding-top: 20px;
        display: flex;
        flex-direction: column;
        gap: 20px;

        .button {
            padding: 5px;
            border-radius: 5px;
            cursor: pointer;
            color: #305a89;
            border: 1px solid #305a89;
            display: inline-block;
        }

        .view-item-stl {
            border: 1px solid orange;
            width: 500px;
            height: 500px;
            cursor: pointer;

            &.show3d {
                width: calc(min(100vw, 100vh) - 200px);
                height: calc(min(100vw, 100vh) - 200px);
                cursor: default;
            }

            &.error {
                border: 1px solid red;
            }
        }
    }
}
</style>
