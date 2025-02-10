<!DOCTYPE html>
<html lang="zh">
	<head>
		<meta charset="UTF-8">
		<meta http-equiv="X-UA-Compatible" content="IE=edge">
		<meta name="viewport" content="width=device-width, initial-scale=1.0">
		<title>测试</title>
		<style>
			.face-capture {
				display: flex;
				flex-direction: column;
				align-items: center;
				justify-content: center;
			}
			.tip {
				position: fixed;
				top: 48px;
				z-index: 5;
				font-size: 18px;
				font-family: PingFangSC-Medium, PingFang SC;
				font-weight: 500;
				color: #333333;
				line-height: 25px;
			}
			.face-capture video,
			.face-capture canvas {
				position: fixed;
				top: 117.5px;
				object-fit: cover;
				z-index: 2;
				background-repeat: no-repeat;
				background-size: 100% 100%;
			}
			.face-capture .img-cover {
				position: fixed;
				top: 63px;
				width: 375px;
				height: 375px;
				object-fit: cover;
				z-index: 3;
				background-repeat: no-repeat;
				background-size: 100% 100%;
		
			}
		
			.face-capture .contentp {
				position: fixed;
				top: 438px;
				font-size: 18px;
				font-weight: 500;
				color: #333333;
			}
		
			.face-capture .rect {
				border: 2px solid #0aeb08;
				position: fixed;
				z-index: 4;
		
			}
			.img-face {
				display: flex;
				flex-direction: column;
				align-items: center;
				justify-content: center;
			}
			.img-face .imgurl {
				position: fixed;
				top: 117.5px;
				width: 266px;
				height: 266px;
				border-radius: 133px;
			}
		</style>
	</head>
	<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
	<body>
		<div id="seePerson">
			<div v-show="showContainer" class="face-capture" id="face-capture">
				<p class="tip">请保持人像在取景框内</p>
				<video id="video" :width="vwidth" :height="vheight" playsinline webkit-playsinline></video>
				<canvas id="refCanvas" :width="cwidth" :height="cheight"></canvas>
				<!-- <img class="img-cover" :src="getImageUrl('yuan')" alt="" /> -->
				<div class="yuan"></div>
				<p class="contentp">{{ scanTip }}</p>
			</div>
			<div v-if="!showContainer" class="img-face">
				<img class="imgurl" :src="imgUrl" />
			</div>
		</div>
	</body>
	<script src="../js/tracking-min.js"></script>
	<script src="../js/face-min.js"></script>
	<script src="../js/eye-min.js.js"></script>
	<script src="../js/mouth-min.js"></script>
	<script>
		new Vue({
			el: '#seePerson', //el用于指定当前Vue实例为哪个容器服务，值通常为选择器字符串。
			data: { //data中用于存储数据，数据供el所指定的容器去使用。
				screenSize: {
					width: window.screen.width,
					height: window.screen.height,
				},
				URL: null,
				streamIns: null, // 视频流
				showContainer: true, // 显示
				tracker: null,
				tipFlag: false, // 提示用户已经检测到
				flag: false, // 判断是否已经拍照
				context: null, // canvas上下文
				profile: [], // 轮廓
				removePhotoID: null, // 停止转换图片
				scanTip: "人脸识别中...", // 提示文字
				imgUrl: "",
				canvas: null,
				trackertask: null,
				vwidth: "666",
				vheight: "466",
				cwidth: "666",
				cheight: "466",
				userInfo: {},
				orderData: {},
			},
			mounted() {
				this.setVideoConfig();
			},
			methods: {
				setVideoConfig() {
					const scale = this.screenSize.width / 375;
					this.vwidth = 266 * scale;
					this.vheight = 266 * scale;
					this.cwidth = 266 * scale;
					this.cheight = 266 * scale;
					this.playVideo();
				},
				getUserMedia(constrains) {
					if (navigator.mediaDevices.getUserMedia) {
						//最新标准API
						navigator.mediaDevices
							.getUserMedia(constrains)
							.then((res) => {
								this.handleSuccess(res);
							})
							.catch((err) => {
								this.handleError(err);
							});
					} else if (navigator.webkitGetUserMedia) {
						//webkit内核浏览器
						navigator
							.webkitGetUserMedia(constrains)
							.then((res) => {
								this.handleSuccess(res);
							})
							.catch((err) => {
								this.handleError(err);
							});
					} else if (navigator.mozGetUserMedia) {
						//Firefox浏览器
						navagator
							.mozGetUserMedia(constrains)
							.then((res) => {
								this.handleSuccess(res);
							})
							.catch((err) => {
								this.handleError(err);
							});
					} else if (navigator.getUserMedia) {
						//旧版API
						navigator
							.getUserMedia(constrains)
							.then((res) => {
								this.handleSuccess(res);
							})
							.catch((err) => {
								this.handleError(err);
							});
					} else {
						this.scanTip = "你的浏览器不支持访问用户媒体设备";
					}
				},

				close() {
					this.flag = false;
					this.tipFlag = false;
					this.showContainer = false;
					this.context = null;
					this.scanTip = "人脸识别中...";
					clearTimeout(this.removePhotoID);
					if (this.streamIns) {
						this.streamIns.enabled = false;
						this.streamIns.getTracks()[0].stop();
						this.streamIns.getVideoTracks()[0].stop();
					}
					this.streamIns = null;
					this.trackertask.stop();
					this.tracker = null;
					// this.showContainer = true;
					// this.setVideoConfig();
					
				},
				dataURLtoFile(dataurl, filename) {
					// 获取到base64编码
					const arr = dataurl.split(",");
					// 将base64编码转为字符串
					const bstr = window.atob(arr[1]);
					let n = bstr.length;
					const u8arr = new Uint8Array(n); // 创建初始化为0的，包含length个元素的无符号整型数组
					while (n--) {
						u8arr[n] = bstr.charCodeAt(n);
					}
					return new File([u8arr], filename, {
						type: "image/jpeg",
					});
				},
				saveAsPNG(c) {
					return c.toDataURL("image/png", 1);
				},
				tackPhoto() {
					// 在画布上面绘制拍到的照片
					this.context.drawImage(
						document.getElementById("video"),
						0,
						0,
						this.vwidth,
						this.vwidth
					);
					// 保存为base64格式
					this.imgUrl = this.saveAsPNG(document.getElementById("refCanvas"));
					// console.log("data.imgUrl", data.imgUrl);
					//判断图片大小
					this.imgSize();
					// todo 这里可以调用后端接口将图片上传比对，然后调用close结束进程
					const file = this.dataURLtoFile(this.imgUrl, "人脸.jpg");
					console.log("file", file);
					 // this.faceToTengXun(); // 人脸比对
					this.close();
				},

				imgSize() {
					if (this.imgUrl) {
						// 获取base64图片byte大小
						const equalIndex = this.imgUrl.indexOf("="); // 获取=号下标
						let size;
						if (equalIndex > 0) {
							const str = this.imgUrl.substring(0, equalIndex); // 去除=号
							const strLength = str.length;
							const fileLength = strLength - (strLength / 8) * 2; // 真实的图片byte大小
							size = Math.floor(fileLength / 1024); // 向下取整
							console.log("size", size + "KB");
						} else {
							const strLength = this.imgUrl.length;
							const fileLength = strLength - (strLength / 8) * 2;
							size = Math.floor(fileLength / 1024); // 向下取整
							console.log("size", size + "KB");
						}
						if (size > 1024) {
							// 图片超过1M 按比例压缩
							this.imgUrl = document
								.getElementById("refCanvas")
								.toDataURL("image/png", 1024 / size);
						}
					}
				},

				initTracker() {
					this.context = document.getElementById("refCanvas").getContext("2d"); // 画布
					this.canvas = document.getElementById("refCanvas");
					this.tracker = new window.tracking.ObjectTracker("face"); // tracker实例
					this.tracker.setInitialScale(4);
					this.tracker.setStepSize(2); // 设置步长
					this.tracker.setEdgesDensity(0.1);
					try {
						this.trackertask = window.tracking.track("#video", this.tracker); // 开始追踪
					} catch (e) {
						this.scanTip = "访问用户媒体失败，请重试";
					}
					//开始捕捉方法 一直不停的检测人脸直到检测到人脸
					this.tracker.on("track", (e) => {
						console.log(e)
						//画布描绘之前清空画布
						this.context.clearRect(0, 0, this.canvas.width, this.canvas.height);
						if (e.data.length === 0) {
							this.scanTip = "未检测到人脸";
						} else {
							e.data.forEach((rect) => {
								//设置canvas 方框的颜色大小
								this.context.strokeStyle = "#42e365";
								this.context.lineWidth = 2;
								this.context.strokeRect(rect.x, rect.y, rect.width, rect.height);
							});
							if (!this.tipFlag) {
								this.scanTip = "检测成功，正在拍照，请保持不动8秒";
							}
							// 1.5秒后拍照，仅拍一次 给用户一个准备时间
							// falg 限制一直捕捉人脸，只要拍照之后就停止检测
							if (!this.flag) {
								console.log('执行了');
								this.scanTip = "拍照中...";
								this.flag = true;
								this.removePhotoID = setTimeout(() => {
									this.tackPhoto();
									document.getElementById("video").pause();
									this.tipFlag = true;
								}, 1500);
							}
						}
					});

				},

				handleSuccess(stream) {
					this.streamIns = stream;
					const video = document.getElementById("video");
					console.log("video", video);
					// webkit内核浏览器
					this.URL = window.URL || window.webkitURL;
					if ("srcObject" in video) {
						video.srcObject = stream;
					} else {
						video.src = this.URL.createObjectURL(stream);
					}

					// 苹果手机的系统弹框会阻止js的线程的继续执行 手动0.1秒之后自动执行代码
					setTimeout(() => {
						video.play();
						this.initTracker(); // 人脸捕捉
					}, 2000);
				},
				handleError(err) {
					this.scanTip = "访问用户媒体失败";
				},
				playVideo() {
					this.getUserMedia({
						//摄像头拍摄的区域
						video: {
							width: 800,
							height: 600,
							facingMode: "user",
						} /* 前置优先 */ ,
					});
				}
			}
		})
	</script>
</html>
