<template>
	<view class="content">
		<image class="logo" src="/static/logo.png"></image>
		<view class="text-area">
			<button type="default" @click="cs">录像</button>
			<button type="default" @click="cs22">人脸识别拍照</button>
		</view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				title: "Hello"
			};
		},
		onLoad() {},
		methods: {
			cs() {
				uni.navigateTo({
					url: "/pages/index1/index1"
				});
			},
			cs22() {
				uni.navigateTo({
					url: "/pages/index3/index3"
				});
			}
		}
	};
</script>

<style>
	.content {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
	}

	.logo {
		height: 200rpx;
		width: 200rpx;
		margin-top: 200rpx;
		margin-left: auto;
		margin-right: auto;
		margin-bottom: 50rpx;
	}

	.text-area {
		display: flex;
		justify-content: center;
	}

	.title {
		font-size: 36rpx;
		color: #8f8f94;
	}
</style>
