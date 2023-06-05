<script>
import { ref, watch, onBeforeUpdate, reactive, onMounted, nextTick } from 'vue'
import { forEach, toLower, sortBy, padStart } from 'lodash'
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

        const picSize = ref(100)
        const isSettingsShow = ref(true)
        const sort = ref('path')

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

        const getNewView = () => {
            return {
                uuid: uuidv4(),
                file: null,
                path: null,
                name: null,
                ext: null,
                isStl: false,
                isShow3d: false,
                img: null,
                imgList: [],
                rotateX: -45,
                rotateY: null,
                rotateZ: 45,

                sizeX: null,
                sizeY: null,
                sizeZ: null,

                error: null
            }
        }

        const handleFileUpload = (event) => {
            if (!event.target.files.length) return

            const viewListTemp = []

            forEach(event.target.files, (file) => {
                const ext = file.name.split('.').pop()
                const isStl = (toLower(ext) === 'stl')

                if (!isStl) return

                viewListTemp.push({
                    ...getNewView(),
                    file: file,
                    path: file.webkitRelativePath,
                    name: file.name,
                    ext: ext,
                    isStl
                })
            })

            viewList.value = viewListTemp

            startScan()
        }

        const sortView = () => {
            viewList.value = sortBy(viewList.value, [function(el) {
                if (sort.value === 'height') {
                    return -el.sizeZ
                } else {
                    return el.path
                }
            }])
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

        const readyScan = ({ uuid, sizeX, sizeY, sizeZ }) => {
            changeView(uuid, {
                img: viewScanRef.value.getImg(),
                sizeX,
                sizeY,
                sizeZ
            })

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

        // const download = (view) => {
        //     const anchor = document.createElement('a')
        //     if (typeof view.file === 'string') {
        //         anchor.href = view.file
        //     } else if (typeof view.file === 'object') {
        //         anchor.href = URL.createObjectURL(view.file)
        //     }
        //     anchor.download = padStart(view.sizeZ, 4, '00')
        //         + '-' + view.name
        //
        //     document.body.appendChild(anchor)
        //     anchor.click()
        //     document.body.removeChild(anchor)
        // }

        const download = (view, key) => {
            return new Promise(function(resolve, reject) {
                const xhr = new XMLHttpRequest()
                xhr.responseType = 'blob'
                xhr.onload = function() {
                    resolve(xhr)
                }
                xhr.onerror = reject
                if (typeof view.file === 'string') {
                    xhr.open('GET', view.file)
                } else if (typeof view.file === 'object') {
                    xhr.open('GET', window.URL.createObjectURL(view.file))
                }
                setTimeout(() => xhr.send(), 500)
            }).then(function(xhr) {
                // const filename = url.substring(url.lastIndexOf("/") + 1).split("?")[0]
                const filename = padStart(view.sizeZ, 4, '00')
                    + '-' + view.name

                console.log('download', key, filename)

                const a = document.createElement('a')
                a.href = window.URL.createObjectURL(xhr.response)
                a.download = filename
                a.style.display = 'none'
                document.body.appendChild(a)
                a.click()
                document.body.removeChild(a)

                return xhr
            });
        }

        const downloadAll = () => {
            let cur = Promise.resolve()
            forEach(viewList.value, (viewItem, key) => {
                cur = cur.then(function() {
                    return download(viewItem, key)
                })
            })
            return cur
        }

        watch(sort, (val) => {
            sortView()
        })

        if (import.meta.env.DEV) {
            (() => {
                viewList.value.push({
                    ...getNewView(),
                    file: './1.stl',
                    path: './1.stl',
                    name: '1.stl',
                    ext: 'stl',
                    isStl: true
                })
                viewList.value.push({
                    ...getNewView(),
                    file: './2.stl',
                    path: './2.stl',
                    name: '2.stl',
                    ext: 'stl',
                    isStl: true
                })
                viewList.value.push({
                    ...getNewView(),
                    file: './3.stl',
                    path: './3.stl',
                    name: '3.stl',
                    ext: 'stl',
                    isStl: true
                })
                startScan()
            })()
        }

        return {
            viewList,
            viewScan,
            viewScanRef,
            viewRef,

            picSize,
            isSettingsShow,
            sort,

            handleFileUpload,
            hide3d,
            show3d,
            setErrorScan,
            loadingScan,
            reloadScan,
            readyScan,
            rotateScan,

            download,
            downloadAll
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

    <div class="head">
        <div>
            <elIcon v-if="isSettingsShow"
                    @click="isSettingsShow = !isSettingsShow"
            ><ArrowDownBold /></elIcon>

            <elIcon v-else
                    @click="isSettingsShow = !isSettingsShow"
            ><ArrowUpBold /></elIcon>
        </div>

        <div v-show="isSettingsShow"
             class="content">

            <input type="file"
                   multiple="multiple"
                   webkitdirectory mozdirectory msdirectory odirectory directory
                   @change="handleFileUpload">

            <div>
                <span>Размер картинки</span>
                <elInputNumber v-model="picSize"
                               :min="100"
                               :max="1000"
                               :step="100"/>
            </div>

            <div>
                <span>Сортировка по</span>
                <elRadioGroup v-model="sort">
                    <elRadio label="path" size="large">названию</elRadio>
                    <elRadio label="height" size="large">высоте</elRadio>
                </elRadioGroup>
            </div>

            <div>
                <elButton icon="Download"
                          round
                          @click="downloadAll()"/>
            </div>
        </div>
    </div>

    <div class="view-list">
        <div v-for="(view, viewKey) in viewList"
             :key="view.uuid"
             class="view-item">

            <div>{{ viewKey }}. {{view.path }}</div>

            <div class="level-2">
                <div v-if="!view.error">
                    <div>высота: {{ view.sizeZ }} мм</div>
                    <div>ширина: {{ view.sizeY }} мм</div>
                    <div>глубина: {{ view.sizeX }} мм</div>
                </div>

                <div>
                    <template v-if="view.error">
                        <elButton icon="Refresh"
                                  round
                                  class="error"
                                  @click="reloadScan(view)"/>
                    </template>

                    <template v-else>
                        <elButton v-if="view.isShow3d"
                                  icon="Picture"
                                  round
                                  @click="hide3d(view)"/>

                        <elButton v-else
                                  icon="Monitor"
                                  round
                                  @click="show3d(view)"/>
                    </template>

                    <elButton icon="Download"
                              round
                              @click="download(view)"/>
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
                     class="view-item-stl"
                     :style="{
                width: picSize+'px',
                height: picSize+'px'
             }"/>

                <div v-else
                     class="view-item-stl"
                     :class="{error: view.error}"></div>
            </div>
        </div>
    </div>
</template>

<style lang="scss">
.head {
    display: flex;
    flex-direction: row;
    gap: 10px;

    position: sticky;
    top: 0;

    background-color: var(--background-color-main);

    .content {
        display: flex;
        flex-direction: column;
        gap: var(--size-main);

        div {
            display: flex;
            align-items: center;
            gap: var(--size-main);
        }
    }
}

.view-scan {
    border: 1px solid var(--color-main);
    width: 500px;
    height: 500px;
}

.view-list {
    display : flex;
    flex-direction: column;
    gap: var(--size-main);

    .view-item {
        padding-top: 20px;
        display: flex;
        flex-direction: column;
        gap: var(--size-main);

        .level-2 {
            display        : flex;
            gap            : 10px;
            flex-direction : column;
            margin-left    : 40px
        }

        .view-item-stl {
            border: 1px solid var(--color-main);
            width: 100px;
            height: 100px;
            cursor: pointer;

            &.show3d {
                width: calc(min(100vw, 100vh) - 200px);
                height: calc(min(100vw, 100vh) - 200px);
                cursor: default;
            }

            &.error {
                border: 1px solid var(--color-error);
            }
        }
    }
}
</style>
