!-- 人脸识别组件 -->
<template>
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
</template>

<script>
	import {
		defineComponent,
		reactive,
		toRefs,
		ref,
		onMounted
	} from "vue";
	import "../../assets/tracking-min.js";
	import "../../assets/face-min.js";
	import "../../assets/eye-min.js";
	import "../../assets/mouth-min.js";

	export default defineComponent({
		name: "seePerson",
		components: {},
		setup(props, ctx) {
			const data = reactive({
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
				vwidth: "266",
				vheight: "266",
				cwidth: "266",
				cheight: "266",
				userInfo: {},
				orderData: {},
			});

			//设置video canvas宽高
			const setVideoConfig = () => {
				const scale = data.screenSize.width / 375;
				data.vwidth = 266 * scale;
				data.vheight = 266 * scale;
				data.cwidth = 266 * scale;
				data.cheight = 266 * scale;
				playVideo();
			};
			// 访问用户媒体设备
			const getUserMedia = (constrains) => {
				if (navigator.mediaDevices.getUserMedia) {
					//最新标准API
					navigator.mediaDevices
						.getUserMedia(constrains)
						.then((res) => {
							handleSuccess(res);
						})
						.catch((err) => {
							handleError(err);
						});
				} else if (navigator.webkitGetUserMedia) {
					//webkit内核浏览器
					navigator
						.webkitGetUserMedia(constrains)
						.then((res) => {
							handleSuccess(res);
						})
						.catch((err) => {
							handleError(err);
						});
				} else if (navigator.mozGetUserMedia) {
					//Firefox浏览器
					navagator
						.mozGetUserMedia(constrains)
						.then((res) => {
							handleSuccess(res);
						})
						.catch((err) => {
							handleError(err);
						});
				} else if (navigator.getUserMedia) {
					//旧版API
					navigator
						.getUserMedia(constrains)
						.then((res) => {
							handleSuccess(res);
						})
						.catch((err) => {
							handleError(err);
						});
				} else {
					data.scanTip = "你的浏览器不支持访问用户媒体设备";
				}
			};
			const close = () => {
				data.flag = false;
				data.tipFlag = false;
				data.showContainer = false;
				data.context = null;
				data.scanTip = "人脸识别中...";
				clearTimeout(data.removePhotoID);
				if (data.streamIns) {
					data.streamIns.enabled = false;
					data.streamIns.getTracks()[0].stop();
					data.streamIns.getVideoTracks()[0].stop();
				}
				data.streamIns = null;
				data.trackertask.stop();
				data.tracker = null;
			}

			// 将base64转文件
			const dataURLtoFile = (dataurl, filename) => {
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
			};

			// 保存为png,base64格式图片
			const saveAsPNG = (c) => {
				return c.toDataURL("image/png", 1);
			};

			// 拍照
			const tackPhoto = () => {
				// 在画布上面绘制拍到的照片
				data.context.drawImage(
					document.getElementById("video"),
					0,
					0,
					data.vwidth,
					data.vwidth
				);
				// 保存为base64格式
				data.imgUrl = saveAsPNG(document.getElementById("refCanvas"));
				console.log("data.imgUrl", data.imgUrl);

				//判断图片大小
				imgSize();
				// todo 这里可以调用后端接口将图片上传比对，然后调用close结束进程
				const file = dataURLtoFile(data.imgUrl, "人脸.jpg");

				console.log("file", file);
				ctx.emit("handleGetPhotoFace", file);

				close();
			};
			const imgSize = () => {
				if (data.imgUrl) {
					// 获取base64图片byte大小
					const equalIndex = data.imgUrl.indexOf("="); // 获取=号下标
					let size;
					if (equalIndex > 0) {
						const str = data.imgUrl.substring(0, equalIndex); // 去除=号
						const strLength = str.length;
						const fileLength = strLength - (strLength / 8) * 2; // 真实的图片byte大小
						size = Math.floor(fileLength / 1024); // 向下取整
						console.log("size", size + "KB");
					} else {
						const strLength = data.imgUrl.length;
						const fileLength = strLength - (strLength / 8) * 2;
						size = Math.floor(fileLength / 1024); // 向下取整
						console.log("size", size + "KB");
					}
					if (size > 1024) {
						// 图片超过1M 按比例压缩
						data.imgUrl = document
							.getElementById("refCanvas")
							.toDataURL("image/png", 1024 / size);
					}
				}
			};

			// 人脸捕捉 设置各种参数 实例化人脸捕捉实例对象,注意canvas上面的动画效果。
			const initTracker = () => {
				data.context = document.getElementById("refCanvas").getContext("2d"); // 画布
				data.canvas = document.getElementById("refCanvas");
				data.tracker = new window.tracking.ObjectTracker("face"); // tracker实例
				data.tracker.setInitialScale(4);
				data.tracker.setStepSize(2); // 设置步长
				data.tracker.setEdgesDensity(0.1);
				try {
					data.trackertask = window.tracking.track("#video", data.tracker); // 开始追踪
				} catch (e) {
					data.scanTip = "访问用户媒体失败，请重试";
				}
				//开始捕捉方法 一直不停的检测人脸直到检测到人脸
				data.tracker.on("track", (e) => {
					//画布描绘之前清空画布
					data.context.clearRect(0, 0, data.canvas.width, data.canvas.height);
					if (e.data.length === 0) {
						data.scanTip = "未检测到人脸";
					} else {
						e.data.forEach((rect) => {
							//设置canvas 方框的颜色大小
							data.context.strokeStyle = "#42e365";
							data.context.lineWidth = 2;
							data.context.strokeRect(rect.x, rect.y, rect.width, rect.height);
						});
						if (!data.tipFlag) {
							data.scanTip = "检测成功，正在拍照，请保持不动2秒";
						}
						// 1.5秒后拍照，仅拍一次 给用户一个准备时间
						// falg 限制一直捕捉人脸，只要拍照之后就停止检测
						if (!data.flag) {
							data.scanTip = "拍照中...";
							data.flag = true;
							data.removePhotoID = setTimeout(() => {
								tackPhoto();
								document.getElementById("video").pause();
								data.tipFlag = true;
							}, 2000);
						}
					}
				});
			};
			// 成功回调
			const handleSuccess = (stream) => {
				data.streamIns = stream;
				const video = document.getElementById("video");
				console.log("video", video);
				// webkit内核浏览器
				data.URL = window.URL || window.webkitURL;
				if ("srcObject" in video) {
					video.srcObject = stream;
				} else {
					video.src = data.URL.createObjectURL(stream);
				}

				// 苹果手机的系统弹框会阻止js的线程的继续执行 手动0.1秒之后自动执行代码
				setTimeout(() => {
					video.play();
					initTracker(); // 人脸捕捉
				}, 100);
			};
			// 失败回调
			const handleError = (err) => {
				data.scanTip = "访问用户媒体失败";
			};
			const playVideo = () => {
				getUserMedia({
					//摄像头拍摄的区域
					video: {
						width: 500,
						height: 500,
						facingMode: "user",
					} /* 前置优先 */ ,
				});
			};
			onMounted(() => {
				setVideoConfig();
			});

			return {
				...toRefs(data),
			};
		},
	});
</script>

<style lang="scss" scoped>
	#seePerson {
		height: 100%;

		.face-capture {
			display: flex;
			flex-direction: column;
			align-items: center;
			justify-content: center;

			.img-cover {
				position: fixed;
				top: 63px;
				width: 375px;
				height: 375px;
				object-fit: cover;
				z-index: 3;
				background-repeat: no-repeat;
				background-size: 100% 100%;
			}

			.yuan {
				position: fixed;
				//   top: 63px;
				top: 100px;
				width: 275px;
				height: 275px;
				object-fit: cover;
				z-index: 3;
				//   background-repeat: no-repeat;
				//   background-size: 100% 100%;
				border: 1px solid #ccc;
				border-radius: 50%;
			}

			.contentp {
				position: fixed;
				top: 398px;
				font-size: 18px;
				font-weight: 500;
				color: #333333;
			}

			.rect {
				border: 2px solid #0aeb08;
				position: fixed;
				z-index: 4;
			}

			video,
			canvas {
				position: fixed;
				top: 101px;
				width: 275px;
				height: 275px;
				object-fit: cover;
				z-index: 1;
				background-repeat: no-repeat;
				background-size: 100% 100%;
				border-radius: 50%;
			}
		}

		.tip {
			position: fixed;
			top: 48px;
			z-index: 5;
			font-size: 18px;
			font-family: "PingFang SC Medium", PingFang SC;
			font-weight: 500;
			color: #333333;
			line-height: 25px;
		}

		.img-face {
			display: flex;
			flex-direction: column;
			align-items: center;
			justify-content: center;

			.imgurl {
				position: fixed;
				top: 117.5px;
				width: 266px;
				height: 266px;
				border-radius: 133px;
			}
		}
	}
</style>
