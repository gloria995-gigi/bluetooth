<template>
	<view class="container">
		<web-view :src="webviewUrl"></web-view>
		<text class="status">蓝牙连接状态: {{ isConnected ? '已连接' : '未连接' }}</text>
		<text class="data">蓝牙数据!!!: {{ bluetoothData }}</text>
		<!-- <CustomModal :show="showModal" :devices="devices" @select="handleSelect"></CustomModal> -->
	</view>
</template>

<script>
export default {

	data() {
		return {
			isInitSuccess: false, // 蓝牙连接状态
			isConnectSuccess: false,
			hasConnectSuccess: false,
			bluetoothData: '',  // 蓝牙数据
			deviceId: '',       // 目标蓝牙设备的deviceId
			serviceId: '',
			characteristicId: '0000FFE1-0000-1000-8000-00805F9B34FB',
			lastUploadTime: null,
			lastLatitude: null,
			lastLongitude: null,
			userid: 137,
			changci: 16,
			qiuchang: 1024,
			qiudong: 5,
			bluetoothDevices: [], // 存储蓝牙设备列表
			selectedDevice: null, // 选中的设备
			actionSheetVisible: false, // 标记弹窗是否可见
			retryCount: 0,
			isFisrstConection: true,
			reconnectTimeout: null,
			reconnectAttempts: 0,
			reconnectTimer: null,
			isConnected: null,
			showedDisconnectedModal: null,
			askedBefore: false,
			connectionInstance: null
		};
	},

	computed: {
		webviewUrl() {
			return `https://earthg.cn/golf/index.html?userid=${this.userid}&changci=${this.changci}&qiudong=${this.qiudong}&qiuchang=${this.qiuchang}`;
		}
	},

	onLoad(option) {
		let that = this
		this.userid = option.userid
		this.changci = option.changci
		this.qiuchang = option.qiuchang
		this.qiudong = option.qiudong
		if (!this.userid)
			this.userid = 236;
		if (!this.changci)
			this.changci = 16;
		if (!this.qiuchang)
			this.qiuchang = 16;
		if (!this.qiudong)
			this.qiudong = 3;


		uni.onBLEConnectionStateChange((res) => {
			console.log(`device ${res.deviceId} state has changed, connected: ${res.connected}`);
			if (!res.connected) {
				// 设置一个5秒的定时器
				this.isConnected = false
				this.reconnectTimeout = setTimeout(() => {
					that.reconnectAttempts = 0;  // 初始化重新连接尝试次数
					that.attemptReconnect()
					// uni.showModal({
					// 	title: '提示',
					// 	content: '连接已断开，正在尝试重新连接',
					// 	showCancel: false, // 只有一个确定按钮
					// 	success: function () {
					// 		that.reconnectAttempts = 0;  // 初始化重新连接尝试次数
					// 		that.attemptReconnect()
					// 	}
					// });
				}, 5000);

			} else {
				if (this.reconnectTimeout) {
					clearTimeout(this.reconnectTimeout);
					this.reconnectTimeout = null;
				}

				that.isConnected = true;

				if (this.reconnectTimer) {
					this.reconnectTimer = null
					clearTimeout(this.reconnectTimer)
				}

				// if (!this.showedDisconnectedModal) {
				// 	this.showedDisconnectedModal = true;

				// 	// 显示已连接提示
				// 	uni.showModal({
				// 		title: '提示',
				// 		content: '蓝牙设备已成功连接。',
				// 		showCancel: false // 只有一个确定按钮
				// 	});
				// }

				// // 显示已连接提示
				// uni.showModal({
				// 	title: '提示',
				// 	content: '蓝牙设备已成功连接。',
				// 	showCancel: false // 只有一个确定按钮
				// });
			}
		});

		//初始化蓝牙连接
		this.openBluetoothAdapter()
	},

	methods: {
		openBluetoothAdapter() {
			uni.openBluetoothAdapter({
				success: (res) => {
					console.log(res)
					this.isInitSuccess = true;
					this.startBluetoothDiscovery()
				},
				fail: (res) => {
					console.log(res)
					uni.showModal({
						title: '提示',
						content: '初始化失败,请打开蓝牙退出重新进入'
					})
					this.isInitSuccess = false;
				}
			});
		},

		attemptReconnect() {
			if (this.reconnectAttempts < 3) {
				console.log(`Attempt ${this.reconnectAttempts + 1} to reconnect...`);
				this.reconnectAttempts++;
				this.connectToDevice();

				// 设置一个5秒的定时器检查是否需要进行下一次尝试
				this.reconnectTimer = setTimeout(() => {
					if (!this.isConnected) {
						this.attemptReconnect();
					}
				}, 8000);  // 这里我设定了5秒，您可以根据实际情况调整
			} else {
				uni.closeBluetoothAdapter({
					deviceId: this.deviceId,
					success: (res) => {
						console.log(res)
					}
				})
				// 如果3次尝试都失败了，开始搜索蓝牙设备
				uni.openBluetoothAdapter({
					success: (res) => {
						console.log(res)
						this.isInitSuccess = true;
						this.startBluetoothDiscovery()
						// uni.showModal({
						// 	title: '提示',
						// 	content: '初始化成功',
						// 	success: function () {
						// 		that.startBluetoothDiscovery()
						// 	}
						// })
					},
					fail: (res) => {
						console.log(res)
						uni.showModal({
							title: '提示',
							content: '初始化失败,请打开蓝牙退出重新进入'
						})
						this.isInitSuccess = false;
					}
				});
			}
		},

		// 开始搜索蓝牙设备
		startBluetoothDiscovery() {
			let that = this
			uni.startBluetoothDevicesDiscovery({
				success: (res) => {
					console.log(res)
					uni.showModal({
						title: '提示',
						content: '开始搜索',
						success: function () {
							that.getBluetoothDevices()

							//15秒后停止搜索释放系统资源
							setTimeout(() => {
								if (!this.bluetoothDevices || this.bluetoothDevices.length == 0)
									uni.stopBluetoothDevicesDiscovery();
							}, 5000);
						}
					})
				},
				fail: (res) => {
					uni.showModal({
						title: '提示',
						content: '搜索失败' + res
					})
					console.error('搜索蓝牙设备失败', res);
				}
			});
		},

		// 监听发现新设备事件
		onBluetoothDeviceFound() {
			uni.onBluetoothDeviceFound(devices => {
				console.log('开始监听寻找到新设备的事件');
				// this.$set(this.disabled, 3, false);
				this.getBluetoothDevices();
			});
		},

		/**
		 * 获取在蓝牙模块生效期间所有已发现的蓝牙设备。包括已经和本机处于连接状态的设备。
		 */
		// 获取蓝牙设备列表
		getBluetoothDevices() {
			let that = this
			uni.getBluetoothDevices({
				success: res => {
					console.log(res)
					this.bluetoothDevices = res.devices; // 更新设备列表
					this.showBluetoothActionSheet(); // 显示最新的弹窗
				}, fail: (res) => {
					console.log(res)
					uni.showModal({
						title: '提示',
						content: '无法连接，请检查蓝牙设备是否开启，请确定开启后点击确认进行后续步骤',
						success: function () {
							that.getBluetoothDevices()
						}
					})
				}
			});
		},

		//显示蓝牙设备选择弹窗
		showBluetoothActionSheet() {
			let targetDevices = this.bluetoothDevices.filter(device => device.name.startsWith('cs'));


			//设备去重
			let uniqueDevice = [];
			const deviceIds = new Set();

			for (const device of targetDevices) {
				if (!deviceIds.has(device.deviceId)) {
					deviceIds.add(device.deviceId)
					uniqueDevice.push(device)
				}
			}

			if (uniqueDevice.length > 0) {
				let itemList = uniqueDevice.map(device => device.name);
				uni.showActionSheet({
					itemList: itemList,
					success: (res) => {
						if (!res.cancel) {
							let selectedDevice = uniqueDevice[res.tapIndex];
							this.selectedDevice = selectedDevice;
							this.deviceId = selectedDevice.deviceId
							// 进行后续操作
							this.connectToDevice();
						}
					},
				});
			} else {
				uni.showModal({
					title: '提示',
					content: '没有找到相关设备，请检查设备是否在周围，是否开机,并退出重试'
				})
			}
		},

		// 连接目标设备
		connectToDevice() {
			let that = this
			uni.createBLEConnection({
				deviceId: that.deviceId,
				success: (res) => {
					if (!that.hasConnectSuccess) {
						// uni.showModal({
						// 	title: '提示',
						// 	content: '连接成功',
						// })
					}
					that.isConnectSuccess = true;
					that.hasConnectSuccess = true;
					//获取服务
					that.getBLEDeviceServices()
				},
				fail: (res) => {
					if (!that.isConnectSuccess) {
						console.log(this.retryCount)
						this.retryCount++
						console.error('连接蓝牙设备失败', res);
						if (this.retryCount < 3 && !that.askedBefore) {
							uni.showModal({
								title: '提示',
								content: '连接失败,重连？',
								success(res) {
									if (res.confirm) {
										that.connectToDevice()
									} else if (res.cancel) {
										console.log('用户点击取消');
										that.askedBefore = true;
									}
								}
							})
						} else {
							this.retryCount = 0
							this.getBluetoothDevices()
						}
					}
				}
			});
		},

		closeBLEConnection() {
			uni.closeBLEConnection({
				deviceId: this.deviceId,
				success: res => {
					uni.showModal({
						title: '提示',
						content: '断开成功,重连？',
						success(res) {
							if (res.confirm) {
								that.getBluetoothDevices()
							} else if (res.cancel) {
								console.log('用户点击取消')
							}
						}
					})
				}
			})
		},

		//获取服务
		getBLEDeviceServices() {
			let that = this
			uni.getBLEDeviceServices({
				deviceId: that.deviceId,
				success(res) {
					console.log(res)
					for (let i = 0; i < res.services.length; i++) {
						if (res.services[i].uuid == '0000FFE0-0000-1000-8000-00805F9B34FB') {
							that.serviceId = res.services[i].uuid
							console.log(that.serviceId)
							that.getBLEDeviceCharacteristics()
						}
					}
				}, fail(err) {
					console.log(err)
					uni.showModal({
						title: '提示',
						content: '未找到服务,重连？',
						success(res) {
							if (res.confirm) {
								that.connectToDevice()
							} else if (res.cancel) {
								console.log('用户点击取消')
							}
						}
					})
				}
			})
		},

		//获取特征值
		getBLEDeviceCharacteristics() {
			let that = this
			uni.getBLEDeviceCharacteristics({
				deviceId: that.deviceId,
				serviceId: that.serviceId,
				success(res) {
					console.log(res)
					for (let i = 0; i < res.characteristics.length; i++) {
						if (res.characteristics[i].uuid == '0000FFE1-0000-1000-8000-00805F9B34FB') {
							that.characteristicId == res.characteristics[i].uuid
							console.log(res.characteristics[i].uuid)
							console.log(that.characteristicId)
							console.log(that.deviceId)
							that.notify()
						}
					}
				}, fail(err) {
					console.log(err)
					uni.showModal({
						title: '提示',
						content: '未找到服务,重连？',
						success(res) {
							if (res.confirm) {
								that.connectToDevice()
							} else if (res.cancel) {
								console.log('用户点击取消')
							}
						}
					})
				}
			})
		},

		notify() {
			let that = this
			uni.notifyBLECharacteristicValueChange({
				state: true,
				deviceId: that.deviceId,
				serviceId: that.serviceId,
				characteristicId: that.characteristicId,
				success(res) {
					console.log(res)
					that.listenValueChange()
				},
				fail(err) {
					console.log(err)
				}
			})
		},

		listenValueChange() {
			let that = this
			uni.onBLECharacteristicValueChange(res => {

				let resHex = that.ab2hex(res.value)

				let result = that.hexCharCodeToStr(resHex)

				let finalresult = that.parseGPSData(result)
				console.log(finalresult)

				let isLoad = that.shouldUpload(finalresult);

				if (isLoad) {
					uni.request({
						url: 'https://golf.earthg.cn/Qiuchang/PostPosition',
						method: 'POST',
						header: { 'content-type': "application/x-www-form-urlencoded" },
						data: {
							userid: this.userid,
							changci: this.changci,
							qiuchang: this.qiuchang,
							qiudong: this.qiudong,
							l: finalresult.longitudeInDegrees,
							b: finalresult.latitudeInDegrees,
							h: 1
						},
						success: (res) => {
							console.log('数据发送成功', res)
						},
						fail: (err) => {
							console.log('失败', err)
						}
					})
				}


			})
		},

		shouldUpload(finalresult) {
			if (finalresult !== null) {
				const currentTime = Date.now();
				var timecha = (currentTime - this.lastUploadTime);

				if (this.lastUploadTime == null || timecha >= 10000) {
					const latitudeDifference = Math.abs(finalresult.latitudeInDegrees - this.lastLatitude);
					const longitudeDifference = Math.abs(finalresult.longitudeInDegrees - this.lastLongitude);
					if (latitudeDifference < 0.000001 && longitudeDifference < 0.000001) {
						return false;
					}
					this.lastLatitude = finalresult.latitudeInDegrees;
					this.lastLongitude = finalresult.longitudeInDegrees;
					this.lastUploadTime = currentTime;
					return true;
				}
				else if (timecha < 2000)
					return false;
				else
					return true;
			}
			return false;
		},

		ab2hex(buffer) {
			const hexArr = Array.prototype.map.call(
				new Uint8Array(buffer),
				function (bit) {
					return ('00' + bit.toString(16).slice(-2))
				}
			)
			return hexArr.join('')
		},

		hexCharCodeToStr(hexCharCodeStr) {
			var trimedStr = hexCharCodeStr.trim();
			var rawStr = trimedStr.substr(0, 2).toLowerCase() === "0x" ? trimedStr.substr(2) : trimedStr;
			var len = rawStr.length;
			if (len % 2 !== 0) {
				alert("存在非法字符!");
				return "";
			}
			var curCharCode;
			var resultStr = [];
			for (var i = 0; i < len; i = i + 2) {
				curCharCode = parseInt(rawStr.substr(i, 2), 16);
				// 过滤掉不可见字符
				if (curCharCode >= 32 && curCharCode <= 126) {
					resultStr.push(String.fromCharCode(curCharCode));
				}
			}
			return resultStr.join("");
		},

		parseGPSData(gpsdata) {
			const parts = gpsdata.split(',');
			console.log(parts)

			if (gpsdata.startsWith("$GNGGA")) {
				if (parts.length >= 10) {
					const latitude = parseFloat(parts[2]);
					const longitude = parseFloat(parts[4]);

					if (!isNaN(latitude) && !isNaN(longitude)) {
						const latitudeInDegrees = Math.floor(latitude / 100) + (latitude % 100) / 60;
						const longitudeInDegrees = Math.floor(longitude / 100) + (longitude % 100) / 60;
						return { latitudeInDegrees, longitudeInDegrees };
					}
				}
			} else if (gpsdata.startsWith("$GNRMC")) {
				if (parts.length >= 10) {
					const latitude = parseFloat(parts[3]);
					const longitude = parseFloat(parts[5]);

					if (!isNaN(latitude) && !isNaN(longitude)) {
						const latitudeInDegrees = Math.floor(latitude / 100) + (latitude % 100) / 60;
						const longitudeInDegrees = Math.floor(longitude / 100) + (longitude % 100) / 60;
						return { latitudeInDegrees, longitudeInDegrees };
					}
				}
			}

			return null;
		}
	},

	onUnload: function () {
		uni.closeBLEConnection({
			deviceId: this.deviceId,
			success(res) {
				console.log(res)
			}
		})
		uni.closeBluetoothAdapter({
			success(res) {
				console.log(res)
			}
		})
	}
};
</script>

<style>
.container {
	padding: 20rpx;
}

.status,
.data {
	font-size: 28rpx;
	margin-bottom: 20rpx;
}
</style>
