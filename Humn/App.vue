<script>
	export default {
		onLaunch: function() {
			console.log('Humn App 启动')
		},
		onShow: function() {
			console.log('Humn App 显示')
		},
		onHide: function() {
			// App 进入后台，音频通过 BackgroundAudioManager 继续播放（App端）
			// H5 端 InnerAudioContext / Web Audio API 会自然继续
			console.log('Humn App 隐藏（后台运行中）')
		}
	}
</script>

<style>
	/* 全局样式 */
	page {
		background-color: #0a0a1a;
	}

	/* 禁止文字选中（提升 App 体验） */
	text {
		-webkit-user-select: none;
		user-select: none;
	}
</style>
