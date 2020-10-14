<template>
    <view class="location">
        <view class="search">
            <input type="text" v-model="searchContent" @input="searchChange" placeholder="可输入检索位置">
            <view class="clear" v-show="searchContent" @click="clearSearchChange">
                <image mode="widthFix" src="@/static/close-icon.png">
            </view>
        </view>
        <view class="content">
            <view class="null" v-if="pointList.length === 0">暂无检索结果</view>
            <view class="li" v-for="(item, index) in pointList" :key="index" @click="pointSearchClick(index, item)">
                <view class="point">
                    <view class="name">{{ item.name }}</view>
                    <view class="address">{{ item.address }}</view>
                </view>
                <view class="tag" v-show="pointIndex === index"></view>
            </view>
        </view>
    </view>
</template>

<script>
export default {
    data () {
        return {
            systemInfo: uni.getSystemInfoSync(), // 设备信息
            isPointClick: false, // 地图状态变化判断
            StatusBar: null, // 状态栏
            Map: null,  // 地图
            Marker: null, // 覆盖物
            MapClose: null, // 关闭
            MapConfirm: null, // 确定
            MapCenter: null, // 当前位置
            searchContent: '', // 检索内容
            pointIndex: null, // 检索index
            pointList: [], // 检索列表
            location: null,  // 当前位置
        }
    },
    onLoad () {
        let that = this;
        // 获取当前位置
        uni.getLocation({
            type: 'gcj02',
            success (point) {
                // 获取当前窗口
                const currentWebview = that.$mp.page.$getAppWebview();
                // 创建地图组件
                that.Map = plus.maps.create('map', {
                    width: '100%',
                    height: '50%',
                    center: new plus.maps.Point( point.longitude, point.latitude ),
                    zoom: 14
                });
                // 向窗口添加地图组件
                currentWebview.append(that.Map);
                // 地图点击事件
                that.Map.onclick = (point) => {
                    that.centerFunc(point);
                }
                // 地图状态变化 (设置中心点便能触发状态，以此来绘点、获取数据，避免重复调用产生屏幕闪烁)
                that.Map.onstatuschanged = (evt) => {
                    let _point = evt.center;
                    that.markerFunc(_point);
                    // 点击检索结果时不触发反向地理编码及检索
                    if (!that.isPointClick) {
                        that.pointIndex = 0;
                        // 反向地理编码 将坐标点转换为地理位置信息
                        plus.maps.Map.reverseGeocode(_point, {
                            coordType: 'gcj02'
                        }, (res) => {
                            that.location = {
                                point: res.coord,
                                address: res.address
                            };
                        }, (err) => {
                            plus.nativeUI.toast(JSON.stringify(err));
                        });
                        that.pointSearchChange(_point);
                    }
                    that.isPointClick = false;
                };
            },
            fail (err) {
                plus.nativeUI.toast('读取当前地理位置失败');
            }
        });
        // view原生控件绘制按钮
        this.drawRectStatusBar();
        this.drawRectClose();
        this.drawRectConfirm();
        this.drawRectCenter();
    },
    methods: {
        centerFunc (point) {
            // 设置地图中心点
            this.Map.setCenter(new plus.maps.Point( point.longitude, point.latitude ));
        },
        markerFunc (point) {
            // 地图中心点覆盖物
            this.Map.removeOverlay(this.Marker);
            this.Marker = new plus.maps.Marker(new plus.maps.Point( point.longitude, point.latitude ));
            this.Marker.setIcon('static/marker-icon.png');
            this.Map.addOverlay(this.Marker);
        },
        searchChange () {
            // 检索内容 防抖/节流
            if (this.timer) clearTimeout(this.timer);
            this.timer = setTimeout(() => {
                if (this.searchContent) {
                    this.pointIndex = 0;
                    this.pointSearchChange(this.searchContent, 0);
                }
            }, 500);
        },
        clearSearchChange () {
            // 检索内容清空
            this.searchContent = '';
            this.centerFunc(this.location.point);
        },
        pointSearchChange (evt, type) {
            // 创建地图检索 当type=0时，evt为检索关键字，反之evt为经纬度
            let search = new plus.maps.Search( this.Map );
            if (type === 0) {
                // 全国检索 (检索的关键字、检索范围的左下角坐标点、检索范围的右上角坐标点)
                search.poiSearchInbounds( evt, new plus.maps.Point( 73.40, 3.52 ), new plus.maps.Point( 135.2, 53.33 ));
            } else {
                // 范围检索 (检索的关键字、检索的中心点坐标、检索的半径，单位为米)
                search.poiSearchNearBy( '', new plus.maps.Point( evt.longitude, evt.latitude ), 1000);
            }
            search.onPoiSearchComplete = ( status, result ) => {
                if ( status === 0 ) {
                    this.pointList = [];
                    this.pointList.push({
                        point: this.location.point,
                        address: this.location.address,
                        name: '地图当前位置'
                    });
                    if ( result.currentNumber > 0 ) {
                        // 循环地理编码 将地理位置信息转换为坐标点
                        result.poiList.map(item => {
                            plus.maps.Map.geocode(item.address + item.name, {
                                city: item.city
                            }, (res) => {
                                let _list = res.address.match(/.+?(省|市|区|自治区|自治州|县|乡|镇)/g);
                                let address = '';
                                _list.map(el => {
                                    address += el;
                                });
                                this.pointList.push({
                                    point: item.point,
                                    address: address + item.address,
                                    city: item.city,
                                    name: item.name
                                });
                            });
                        });
                    } else {
                        plus.nativeUI.toast('没有检索到结果');
                    }
                } else {
                    plus.nativeUI.toast('检索失败');
                }
            }
        },
        pointSearchClick (index, item) {
            // 检索结果事件
            this.pointIndex = index;
            // 防抖/节流
            if (this.timer) clearTimeout(this.timer);
            this.timer = setTimeout(() => {
                this.isPointClick = true;
                this.location = {
                    point: item.point,
                    address: item.address,
                    name: item.name
                };
                this.pointList.splice(0, 1, {
                    point: item.point,
                    address: item.address,
                    name: '地图当前位置'
                });
                this.centerFunc(item.point);
            }, 500);
        },
        drawRectStatusBar () {
            // 绘制状态栏
            let statusbar = new plus.nativeObj.View('statusbar', {
                height: `${this.systemInfo.statusBarHeight}px`,
                width: '100%'
            });
            statusbar.drawRect({
                color:'rgba(0, 0, 0, .3)'
            });
            statusbar.show();
            this.StatusBar = statusbar;
        },
        drawRectClose () {
            // 绘制关闭按钮
            let close = new plus.nativeObj.View('close', {
                top: `${this.systemInfo.statusBarHeight + 10}px`,
                left: '10px',
                height: '30px',
                width: '30px'
            });
            close.drawRect({
                color:'#999',
                radius:'8px'
            }, {
                width:'100%',
                height:'100%'
            });
            close.drawBitmap('static/close-icon.png', {}, {
                top: '7.5px',
                left: '7.5px',
                width: '15px',
                height: '15px'
            });
            close.show();
            this.MapClose = close;
            // 关闭按钮事件
            close.addEventListener('click', () => {
                uni.navigateBack();
            }, false);
        },
        drawRectConfirm () {
            // 绘制确定按钮
            let confirm = new plus.nativeObj.View('confirm', {
                top: `${this.systemInfo.statusBarHeight + 10}px`,
                left: `${this.systemInfo.screenWidth - 60}px`,
                height: '30px',
                width: '50px'
            });
            confirm.drawRect({
                color:'#409eff',
                radius:'8px'
            }, {
                width:'100%',
                height:'100%'
            });
            confirm.drawText('确定', {}, {
                color: '#fff',
                size: '14px'
            });
            confirm.show();
            this.MapConfirm = confirm;
            // 确定按钮事件
            confirm.addEventListener('click', () => {
                if (this.location) {
                    uni.$emit('location', this.location);
                    uni.navigateBack();
                } else {
                    plus.nativeUI.toast('请先选择地理位置');
                }
            }, false);
        },
        drawRectCenter () {
            // 绘制当前位置按钮
            let center = new plus.nativeObj.View('center', {
                top: '43%',
                left: `${this.systemInfo.screenWidth - 40}px`,
                height: '30px',
                width: '30px'
            });
            center.drawRect({
                color:'#999',
                radius:'30px'
            }, {
                width:'100%',
                height:'100%'
            });
            center.drawBitmap('static/center-icon.png', {}, {
                top: '5px',
                left: '5px',
                width: '20px',
                height: '20px'
            });
            center.show();
            this.MapCenter = center;
            let that = this;
            // 当前位置按钮事件
            center.addEventListener('click', () => {
                uni.getLocation({
                    type: 'gcj02',
                    success (point) {
                        that.centerFunc(point);
                    },
                    fail (err) {
                        plus.nativeUI.toast('读取当前地理位置失败');
                    }
                });
            }, false);
        }
    },
    onUnload () {
        // 卸载页面释放原生控件资源
        this.StatusBar.close();
        this.MapClose.close();
        this.MapConfirm.close();
        this.MapCenter.close();
        this.Map.close();
    }
}
</script>

<style lang="scss" scoped>
    .location {
        position: fixed;
        bottom: 0;
        left: 0;
        display: flex;
        flex-direction: column;
        width: 100%;
        height: 50vh;
        .search {
            position: relative;
            flex-shrink: 0;
            padding: 20rpx;
            input {
                padding: 15rpx 70rpx 15rpx 20rpx;
                background: #e5e5e5;
                border: 1rpx solid #e5e5e5;
                font-size: 28rpx;
                border-radius: 10rpx;
            }
            .clear {
                position: absolute;
                top: 50%;
                right: 40rpx;
                width: 30rpx;
                height: 30rpx;
                border: 2rpx solid #fff;
                text-align: center;
                line-height: 18rpx;
                border-radius: 50%;
                transform: translateY(-50%);
                image {
                    width: 20rpx;
                    height: 20rpx !important;
                }
            }
        }
        .content {
            overflow-y: auto;
            flex: 1;
            .li {
                display: flex;
                align-items: center;
                padding: 20rpx;
                border-top: 1rpx solid #e5e5e5;
                line-height: 1.3;
                .point {
                    flex: 1;
                }
                .name {
                    margin-bottom: 10rpx;
                    font-size: 28rpx;
                }
                .address {
                    color: #999;
                    font-size: 24rpx;
                }
                .tag {
                    flex-shrink: 0;
                    width: 30rpx;
                    height: 30rpx;
                    margin-left: 30rpx;
                    border: 10rpx solid #409eff;
                    background: #fff;
                    border-radius: 50%;
                    box-sizing: border-box;
                }
            }
        }
    }
    .null {
        padding: 30rpx 0;
        color: #999;
        font-size: 28rpx;
        text-align: center;
    }
</style>