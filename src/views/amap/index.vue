<template>
    <div>
        <div class="wrapper">
            <ElButton type="success" @click="resetMap">重置地图配置</ElButton>
            <span> | </span>
            <span>是否跟随货车移动：</span
            ><ElSwitch v-model="followCarCenter" />
        </div>
        <div id="root">
            <div id="container" />
        </div>
    </div>
</template>

<script setup lang="ts">
import {
    ref,
    onMounted,
    onUnmounted,
    onBeforeMount,
    watch,
    defineProps
} from "vue"
import AMapLoader from "@amap/amap-jsapi-loader"
import generateRandomColor from "@/utils/generateRandomColor"
import movingCarIcon from "@/assets/moving-car.png"
import { useDark } from "@pureadmin/utils"
import { http } from "@/utils/http"
const { isDark } = useDark()
const props = defineProps({
    path: {
        type: Array,
        default: () => [
            [116.5, 40],
            [113.2, 24.1]
        ]
    },
    carPosition: {
        type: Array,
        default: () => [116.5, 40]
    }
})
let map: any = null
let car: any = null
let moveInterval: any = null
const movingFactor = ref(0.01)
const path = ref<any>(props.path)
const followCarCenter = ref(false)
// 中国地图中心：
const initialPosition = [116.397428, 39.90923]
const initialPath = props.path
function toggleFollowCarCenter() {
    followCarCenter.value = !followCarCenter.value
}

function resetMap() {
    map.setZoomAndCenter(4.8, initialPosition)
    followCarCenter.value = false
}

function generatePosition(
    coord1: [number, number],
    coord2: [number, number],
    factor: number
) {
    const [lon1, lat1] = coord1
    const [lon2, lat2] = coord2
    const randomLon = lon1 + (lon2 - lon1) * factor
    const randomLat = lat1 + (lat2 - lat1) * factor
    return [randomLon, randomLat]
}

function initMap(AMap: any) {
    map = new AMap.Map("container", {
        mapStyle: "amap://styles/normal",
        zoom: 4.8,
        center: initialPosition,
        scrollWheel: false,
        doubleClickZoom: false,
        dragEnable: false,
        keyboardEnable: false
    })

    const startMarker = addMapMarker(AMap, path.value[0], true) // 设置为起点图标
    const endMarker = addMapMarker(
        AMap,
        path.value[path.value.length - 1],
        false
    ) // 设置为终点图标

    // 添加路线
    const polyline = new AMap.Polyline({
        path: [path.value],
        strokeColor: generateRandomColor(),
        strokeWeight: 4
    })
    map.add(polyline)

    // 添加汽车图标
    const viaIcon = new AMap.Icon({
        size: new AMap.Size(40, 40),
        image: movingCarIcon
    })

    car = new AMap.Marker({
        map: map,
        position: path.value[0],
        icon: viaIcon,
        offset: new AMap.Pixel(-15, -30)
    })
    // 给汽车添加点击事件
    car.on("click", (event: any) => {
        handleCarClick(event, map) // 传递事件对象和汽车索引
    })

    // 模拟汽车移动, 没两秒获取一次请求，然后更新汽车位置
    let moveFactor = 0 // 移动因子，初始化为0
    moveInterval = setInterval(() => {
        // axios.....
        const pacing = 0.01
        moveFactor += pacing
        if (moveFactor > 1) moveFactor = 0
        const newPosition = generatePosition(
            path.value[0],
            path.value[1],
            moveFactor
        )
        car.setPosition(newPosition)
        if (followCarCenter.value === true) {
            map.setCenter(newPosition)
        }
        // handleCarMove(car, map)
    }, 1000) // 每秒更新一次位置

    watch(
        () => isDark.value,
        () => {
            console.log("watch", isDark.value)
            isDark.value
                ? map?.setMapStyle("amap://styles/darkblue")
                : map.setMapStyle("amap://styles/normal")
        },
        { immediate: true, deep: true }
    )
    //点击小车事件
    function handleCarClick(event, map) {
        map.setZoom(7)
        // map.setCenter()
    }
}

function addMapMarker(AMap: any, position: [number, number], isStart: boolean) {
    const icon = isStart
        ? new AMap.Icon({
              size: new AMap.Size(40, 40),
              image: "//a.amap.com/jsapi_demos/static/demo-center/icons/dir-marker.png",
              imageSize: new AMap.Size(135, 40)
          })
        : new AMap.Icon({
              size: new AMap.Size(25, 34),
              image: "//a.amap.com/jsapi_demos/static/demo-center/icons/dir-marker.png",
              imageSize: new AMap.Size(135, 40),
              imageOffset: new AMap.Pixel(-95, -3)
          })
    return new AMap.Marker({
        map: map,
        position: position,
        icon: icon,
        offset: new AMap.Pixel(-15, -30)
    })
}

onMounted(() => {
    AMapLoader.load({
        key: "5aee4f20d70cfb76d4a45dbea9f9104d", // 申请好的Web端开发者Key
        version: "2.0", // 指定要加载的 JSAPI 的版本
        plugins: [] // 需要使用的的插件列表
    })
        .then(AMap => {
            initMap(AMap)
        })
        .catch(e => {
            console.log(e)
        })
})
onBeforeMount(() => {
    // 请求接口：1. 或者props？ 2. 本地mock数据
    // lng lat from to
    // 1. 从接口获取数据
    // 然后setInterval进行数据更新
    // 请求接口放在
    // paths.value = [
    //     // 第一组路径
    //     [
    //         [116.5, 40],
    //         [113.2, 24.1]
    //     ]
    // ]
})
onUnmounted(() => {
    clearInterval(moveInterval)
    map?.destroy()
})
</script>

<style scoped>
#container {
    width: 100%;
    height: 90vh;
}
</style>
