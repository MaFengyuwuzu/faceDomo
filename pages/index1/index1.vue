<template>
	<view class="aaa">
			<web-view :webview-styles="webviewStyles" src="/hybrid/html/index.html"></web-view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				URL:''
			}
		},
		onLoad() {
			this.URL =' ../../hybrid/face.html'


		},
		mounted() {
						window.addEventListener("message", this.handlePostMessage);

		},
		methods: {
			
		}
	}
</script>

<style>
  .aaa{
	  width: 100%;
	  height: 100%;
  } 
</style>
