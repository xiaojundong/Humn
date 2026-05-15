<template>
	<view
		class="container"
		:class="'theme-' + actualTheme"
		:style="{ paddingTop: statusBarHeight + 'px' }"
	>
		<!-- ========== 顶部栏 ========== -->
		<view class="top-bar">
			<view class="app-brand">
				<text class="app-name">Humn</text>
				<text class="app-tagline">白噪音 · 助眠 · 专注</text>
			</view>
			<view class="menu-btn" @tap="toggleMenu">
				<view class="menu-line"></view>
				<view class="menu-line"></view>
				<view class="menu-line"></view>
			</view>
		</view>
 
		<!-- 当前播放提示 -->
		<view class="now-playing" :class="{ in: nowPlayingIn }">
			<view class="playing-wave">
				<view v-for="i in 4" :key="i" class="wave-bar" :class="{ active: isPlaying }"></view>
			</view>
			<text class="playing-text">
				<template v-if="activeSound">
					{{ isPlaying ? '正在播放' : '已暂停' }} · {{ activeSound.name }}
				</template>
			</text>
		</view>

		<!-- ========== 声音列表 ========== -->
		<scroll-view class="sound-list" scroll-y :show-scrollbar="false">
			<view
				v-for="sound in allSounds"
				:key="sound.id"
				class="sound-row"
				:class="{
					'row-playing': activeSound && activeSound.id === sound.id && isPlaying,
					'row-paused': activeSound && activeSound.id === sound.id && !isPlaying
				}"
				@tap="toggleSound(sound)"
				@longpress="onLongPress(sound)"
			>
				<view class="sound-circle" :class="{ active: activeSound && activeSound.id === sound.id }">
					<view class="pulse-ring" v-if="activeSound && activeSound.id === sound.id && isPlaying"></view>
					<view class="pulse-ring delay" v-if="activeSound && activeSound.id === sound.id && isPlaying"></view>
					<text class="sound-emoji">{{ sound.icon }}</text>
					<view class="custom-dot" v-if="sound.type === 'custom' && !sound.played"></view>
				</view>

				<view class="sound-rect">
					<text class="sound-title">{{ sound.name }}</text>
					<text class="sound-sub" v-if="activeSound && activeSound.id === sound.id">
						{{ isPlaying ? '播放中' : '已暂停' }}
					</text>
					<text class="sound-sub" v-else-if="sound.type === 'custom'">自定义</text>
					<text class="sound-sub" v-else-if="sound.type === 'noise'">程序合成</text>
					<text class="sound-sub" v-else>自然之声</text>
				</view>

				<view class="sound-status">
					<view class="status-dot playing-dot" v-if="activeSound && activeSound.id === sound.id && isPlaying"></view>
					<view class="status-dot paused-dot" v-else-if="activeSound && activeSound.id === sound.id && !isPlaying"></view>
					<view class="status-dot idle-dot" v-else></view>
				</view>
			</view>
			<view style="height: 100rpx;"></view>
		</scroll-view>

		<!-- 底部提示 -->
		<view class="file-tip" :class="{ in: fileTipVisible }" @tap="hideFileTip">
			<text>部分声音需要放入音频文件（static/audio/）\n或通过右上角菜单导入自定义音频</text>
		</view>

		<!-- ==================== 菜单面板 ==================== -->
		<view class="menu-overlay" :class="{ in: menuAnimIn }" v-if="showMenu" @tap="closeMenu">
			<view
				class="menu-panel"
				:class="{ in: menuAnimIn }"
				:style="{ marginTop: (statusBarHeight + 90) + 'px' }"
				@tap.stop
			>
				<text class="menu-title">选项</text>

				<text class="menu-section-label">主题</text>
				<view
					v-for="opt in themeOptions"
					:key="opt.value"
					class="menu-item theme-item"
					@tap="setTheme(opt.value)"
				>
					<text class="menu-item-icon">{{ themeValue === opt.value ? '●' : '○' }}</text>
					<text class="menu-item-text">{{ opt.label }}</text>
				</view>

				<view class="menu-divider"></view>

				<view class="menu-item" @tap="openAddModal">
					<text class="menu-item-icon">＋</text>
					<text class="menu-item-text">导入自定义声音</text>
				</view>

				<view class="menu-divider" v-if="customSounds.length > 0"></view>

				<text class="menu-section-label" v-if="customSounds.length > 0">已导入</text>
				<view
					v-for="sound in customSounds"
					:key="sound.id"
					class="menu-item custom-item"
				>
					<text class="menu-item-icon">{{ sound.icon }}</text>
					<text class="menu-item-text">{{ sound.name }}</text>
					<view class="menu-delete" @tap.stop="confirmDeleteCustomSound(sound)">
						<text>🗑</text>
					</view>
				</view>

				<view class="menu-empty" v-if="customSounds.length === 0">
					<text>暂无导入的声音</text>
				</view>

				<view class="menu-divider"></view>

				<view class="menu-item close-item" @tap="closeMenu">
					<text class="menu-item-text">关闭菜单</text>
				</view>
			</view>
		</view>

		<!-- ==================== 确认弹窗 ==================== -->
		<view class="modal-overlay" :class="{ in: confirmAnimIn }" v-if="confirmDialog.show" @tap.self="closeConfirm" @touchmove.stop.prevent>
			<view class="modal-card confirm-card" :class="{ in: confirmAnimIn }">
				<text class="modal-title">{{ confirmDialog.title }}</text>
				<text class="confirm-message">{{ confirmDialog.message }}</text>
				<view class="modal-actions">
					<view class="btn btn-cancel" @tap="closeConfirm">
						<text>{{ confirmDialog.cancelText }}</text>
					</view>
					<view
						class="btn"
						:class="confirmDialog.danger ? 'btn-danger' : 'btn-save'"
						@tap="handleConfirm"
					>
						<text>{{ confirmDialog.confirmText }}</text>
					</view>
				</view>
			</view>
		</view>

		<!-- ==================== 导入弹窗 ==================== -->
		<view class="modal-overlay" :class="{ in: importModalAnimIn }" v-if="showAddModal" @tap.self="cancelAdd" @touchmove.stop.prevent>
			<view class="modal-card" :class="{ in: importModalAnimIn }">
				<text class="modal-title">导入自定义声音</text>

				<view class="form-group">
					<text class="form-label">名称</text>
					<input
						class="form-input"
						v-model="formName"
						placeholder="输入声音名称"
						placeholder-style="color:#999"
						maxlength="10"
					/>
				</view>

				<view class="form-group">
					<text class="form-label">图标</text>
					<view class="emoji-grid">
						<view
							v-for="emoji in emojiPalette"
							:key="emoji"
							class="emoji-item"
							:class="{ selected: formIcon === emoji }"
							@tap="formIcon = emoji"
						>
							<text class="emoji-char">{{ emoji }}</text>
						</view>
					</view>
				</view>

				<view class="form-group">
					<text class="form-label">音频文件</text>
					<view class="file-picker" @tap="pickFile">
						<text class="file-picker-icon">📁</text>
						<text class="file-picker-text" v-if="!formFile">点击选择音频文件</text>
						<text class="file-picker-text file-selected" v-else>已选择：{{ formFileName }}</text>
					</view>
					<text class="form-hint">支持 MP3 / WAV / OGG / M4A / AAC</text>
				</view>

				<view class="modal-actions">
					<view class="btn btn-cancel" @tap="cancelAdd">
						<text>取消</text>
					</view>
					<view class="btn btn-save" :class="{ disabled: !canSave }" @tap="saveCustomSound">
						<text>保存</text>
					</view>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
// ==================== 噪声引擎 ====================
class NoiseEngine {
	constructor() {
		this.ctx = null
		this.source = null
		this.gainNode = null
		this._running = false
	}

	_init() {
		if (this.ctx) return
		const Ctor = window.AudioContext || window.webkitAudioContext
		if (!Ctor) throw new Error('当前环境不支持 Web Audio API')
		this.ctx = new Ctor()
		this.gainNode = this.ctx.createGain()
		this.gainNode.gain.value = 0.5
		this.gainNode.connect(this.ctx.destination)
	}

	start(type) {
		this._init()
		const sr = this.ctx.sampleRate
		const len = 2 * sr
		const buf = this.ctx.createBuffer(1, len, sr)
		const d = buf.getChannelData(0)

		if (type === 'white') {
			for (let i = 0; i < len; i++) d[i] = Math.random() * 2 - 1
		} else if (type === 'pink') {
			let b0 = 0, b1 = 0, b2 = 0, b3 = 0, b4 = 0, b5 = 0, b6 = 0
			for (let i = 0; i < len; i++) {
				const w = Math.random() * 2 - 1
				b0 = 0.99886 * b0 + w * 0.0555179
				b1 = 0.99332 * b1 + w * 0.0750759
				b2 = 0.96900 * b2 + w * 0.1538520
				b3 = 0.86650 * b3 + w * 0.3104856
				b4 = 0.55000 * b4 + w * 0.5329522
				b5 = -0.7616 * b5 - w * 0.0168980
				d[i] = (b0 + b1 + b2 + b3 + b4 + b5 + b6 + w * 0.5362) * 0.11
				b6 = w * 0.115926
			}
		} else if (type === 'brown') {
			let last = 0
			for (let i = 0; i < len; i++) {
				const w = Math.random() * 2 - 1
				last = (last + 0.02 * w) / 1.02
				d[i] = last * 3.5
			}
		}

		this.source = this.ctx.createBufferSource()
		this.source.buffer = buf
		this.source.loop = true
		this.source.connect(this.gainNode)
		this.source.start(0)
		this._running = true
	}

	stop() {
		if (this.source) {
			try { this.source.stop() } catch (_) {}
			this.source.disconnect()
			this.source = null
		}
		this._running = false
	}

	destroy() {
		this.stop()
		if (this.ctx) { this.ctx.close(); this.ctx = null; this.gainNode = null }
	}
}

// ==================== 预设声音 ====================
const PRESET_SOUNDS = [
	{ id: 'rain',       name: '雨声',       icon: '🌧️',  type: 'sample', file: '/static/audio/rain.mp3' },
	{ id: 'thunder',    name: '雷雨',       icon: '⛈️',  type: 'sample', file: '/static/audio/thunder.mp3' },
	{ id: 'ocean',      name: '海浪',       icon: '🌊',  type: 'sample', file: '/static/audio/ocean.mp3' },
	{ id: 'stream',     name: '溪流',       icon: '💧',  type: 'sample', file: '/static/audio/stream.mp3' },
	{ id: 'campfire',   name: '篝火',       icon: '🔥',  type: 'sample', file: '/static/audio/campfire.mp3' },
	{ id: 'wind',       name: '风声',       icon: '🍃',  type: 'sample', file: '/static/audio/wind.mp3' },
	{ id: 'forest',     name: '森林鸟鸣',   icon: '🌲',  type: 'sample', file: '/static/audio/forest.mp3' },
	{ id: 'cicada',     name: '夏夜蝉鸣',   icon: '🦗',  type: 'sample', file: '/static/audio/cicada.mp3' },
	{ id: 'fan',        name: '风扇',       icon: '🌀',  type: 'sample', file: '/static/audio/fan.mp3' },
	{ id: 'whitenoise', name: '白噪音',     icon: '📡',  type: 'noise',  noiseType: 'white' },
	{ id: 'pinknoise',  name: '粉红噪音',   icon: '🎀',  type: 'noise',  noiseType: 'pink' },
	{ id: 'brownnoise', name: '棕色噪音',   icon: '🔊',  type: 'noise',  noiseType: 'brown' },
	{ id: 'squeeze',    name: '捏捏乐',     icon: '🤏',  type: 'sample', file: '/static/audio/squeeze.mp3' },
	{ id: 'keyboard',   name: '键盘敲击',   icon: '⌨️',  type: 'sample', file: '/static/audio/keyboard.mp3' },
	{ id: 'catpurr',    name: '猫咪呼噜',   icon: '🐱',  type: 'sample', file: '/static/audio/catpurr.mp3' },
	{ id: 'heartbeat',  name: '心跳',       icon: '💓',  type: 'sample', file: '/static/audio/heartbeat.mp3' },
	{ id: 'coffee',     name: '咖啡厅',     icon: '☕',  type: 'sample', file: '/static/audio/coffee.mp3' },
	{ id: 'bell',       name: '风铃',       icon: '🔔',  type: 'sample', file: '/static/audio/bell.mp3' }
]

const EMOJI_PALETTE = [
	'🎵','🎶','🎼','🎧','🎤','🎹','🥁','🎸',
	'🌧️','⛈️','🌊','💧','🔥','🍃','🌲','⭐',
	'🌙','✨','💫','🌈','🐱','🐶','🐦','🦉',
	'🦋','🐝','🐢','🐠','🐋','🦜','🌸','🌺',
	'❤️','💙','💚','💛','💜','🧡','🤍','💗',
	'☕','🍵','🕯️','📖','🧘','💤','🌿','🪷'
]

const STORAGE_KEY = 'humn_custom_sounds'
const THEME_KEY = 'humn_theme'

// 动画时长常量（与 CSS 保持同步）
const ANIM = {
	MENU: 350,
	MODAL: 380,
	CONFIRM: 380,
	TIP: 350,
	NOW_PLAYING: 400
}

export default {
	data() {
		return {
			presetSounds: PRESET_SOUNDS,
			customSounds: [],
			activeSound: null,
			isPlaying: false,
			statusBarHeight: 0,
			showFileTip: false,
			noise: null,
			innerAudio: null,
			bgAudio: null,

			// 菜单
			showMenu: false,
			menuAnimIn: false,

			// 主题
			themeValue: 'light',
			actualTheme: 'light',
			themeOptions: [
				{ value: 'light', label: '浅色主题' },
				{ value: 'dark',  label: '深色主题' }
			],

			// 导入弹窗
			showAddModal: false,
			importModalAnimIn: false,
			formName: '',
			formIcon: '',
			formFile: '',
			formFileName: '',
			emojiPalette: EMOJI_PALETTE,
			_themeMediaQuery: null,

			// 确认弹窗
			confirmDialog: {
				show: false,
				title: '',
				message: '',
				confirmText: '确认',
				cancelText: '取消',
				danger: false,
				onConfirm: null
			},
			confirmAnimIn: false,

			// 动画状态
			fileTipVisible: false,
			nowPlayingIn: false,
			_fileTipTimer: null,
			_nowPlayingTimer: null,
			_lastToggle: 0,
			_switchTimestamp: 0
		}
	},
	computed: {
		allSounds() {
			return [...this.presetSounds, ...this.customSounds]
		},
		canSave() {
			return this.formName.trim() && this.formFile
		}
	},
	watch: {
		// 当 activeSound 变化时，控制 now-playing 条的进出动画
		activeSound(val) {
			if (val) {
				this.enterNowPlaying()
			} else {
				this.leaveNowPlaying()
			}
		},
		showFileTip(val) {
			if (val) {
				this.$nextTick(() => {
				setTimeout(() => { this.fileTipVisible = true }, 30)
			})
			}
		}
	},
	onLoad() {
		const systemInfo = uni.getSystemInfoSync()
		this.statusBarHeight = systemInfo.statusBarHeight || 20

		this.loadCustomSounds()
		this.loadTheme()
		this.watchSystemTheme()

		this.noise = new NoiseEngine()

		// #ifndef APP-PLUS
		// 仅 H5 / 小程序用 InnerAudioContext；App 端由 BackgroundAudioManager 接管
		this.innerAudio = uni.createInnerAudioContext()
		this.innerAudio.loop = true
		this.innerAudio.onCanplay(() => {
			if (this.activeSound && this.activeSound.type !== 'noise') {
				this.isPlaying = true
			}
		})
		this.innerAudio.onError((err) => {
			console.error('音频错误:', err)
			this.showFileTipMsg()
			if (this.activeSound && this.activeSound.type !== 'noise') {
				this.activeSound = null
				this.isPlaying = false
			}
		})
		// #endif

		// #ifdef APP-PLUS
		this.bgAudio = uni.getBackgroundAudioManager()
		this.bgAudio.onEnded(() => {
			if (Date.now() - this._switchTimestamp < 1000) return
			if (this.activeSound && this.activeSound.type !== 'noise') {
				this.bgAudio.play()
			}
		})
		this.bgAudio.onPlay(() => { if (Date.now() - this._switchTimestamp < 1000) return; if (this.activeSound && this.activeSound.type !== 'noise') this.isPlaying = true })
		this.bgAudio.onPause(() => { if (Date.now() - this._switchTimestamp < 1000) return; if (this.activeSound && this.activeSound.type !== 'noise') this.isPlaying = false })
		this.bgAudio.onStop(() => {
				if (Date.now() - this._switchTimestamp < 1000) return
				if (this.activeSound && this.activeSound.type !== 'noise') {
					this.activeSound = null
					this.isPlaying = false
				}
			})
		this.bgAudio.onError((err) => {
				console.error('后台音频错误:', err)
				// 清掉无效 src 防止平台自动重试（某些 Android ROM 会每 3s 重试一次）
				try { this.bgAudio.src = '' } catch (_) {}
				if (this.activeSound && this.activeSound.type !== 'noise') {
					this.showFileTipMsg()
					this.activeSound = null
					this.isPlaying = false
				}
			})
		// #endif
	},
	onShow() {
		if (this.activeSound) {
			if (this.activeSound.type === 'noise') {
				if (this.noise && !this.noise._running) {
					this.noise.start(this.activeSound.noiseType)
					this.isPlaying = true
				}
			}
			// #ifdef APP-PLUS
			if (this.activeSound.type !== 'noise' && this.bgAudio && !this.bgAudio.paused) {
				this.isPlaying = true
			}
			// #endif
		}
	},
	onUnload() {
		this.stopAll()
		this.unwatchSystemTheme()
		if (this.noise) { this.noise.destroy(); this.noise = null }
		if (this.innerAudio) { this.innerAudio.destroy(); this.innerAudio = null }
	},
	methods: {
		// ==================== 播放 ====================
		toggleSound(sound) {
			// 防连点：300ms 内忽略重复点击，避免状态竞争
			const now = Date.now()
			if (now - this._lastToggle < 300) return
			this._lastToggle = now
			if (this.activeSound && this.activeSound.id === sound.id) {
				this.isPlaying ? this.stopSound() : this.resumeSound()
			} else {
				this.playSound(sound)
			}
		},

		playSound(sound) {
			if (this.noise) this.noise.stop()
			if (sound.type === 'custom' && !sound.played) {
				sound.played = true
				this.persistCustomSounds()
			}
			this.activeSound = sound
			this.isPlaying = true
			this._switchTimestamp = Date.now()
			if (sound.type === 'noise') {
				this.noise.start(sound.noiseType)
			} else {
				// #ifdef APP-PLUS
				this.bgAudio.title = sound.name
				this.bgAudio.coverImgUrl = '/static/logo.png'
				this.bgAudio.src = sound.file
				this.$nextTick(() => { this.bgAudio.play() })
				// #endif
				// #ifndef APP-PLUS
				this.innerAudio.src = sound.file
				this.innerAudio.play()
				// #endif
			}
		},

		resumeSound() {
			const s = this.activeSound
			if (!s) return
			this.isPlaying = true
			if (s.type === 'noise') {
				this.noise.start(s.noiseType)
			} else {
				// #ifdef APP-PLUS
				this.bgAudio.play()
				// #endif
				// #ifndef APP-PLUS
				this.innerAudio.play()
				// #endif
			}
		},

		stopSound() {
			this.stopAll()
			this.activeSound = null
			this.isPlaying = false
		},

		stopAll() {
			if (this.noise) this.noise.stop()
			// #ifdef APP-PLUS
			if (this.bgAudio && this.activeSound && this.activeSound.type !== 'noise') { try { this.bgAudio.stop() } catch (_) {} }
			// #endif
			// #ifndef APP-PLUS
			if (this.innerAudio) { this.innerAudio.stop() }
			// #endif
		},

		// ==================== Now-Playing 动画 ====================
		enterNowPlaying() {
			clearTimeout(this._nowPlayingTimer)
			this.$nextTick(() => {
				setTimeout(() => { this.nowPlayingIn = true }, 30)
			})
		},
		leaveNowPlaying() {
			this.nowPlayingIn = false
			// 不需要从 DOM 移除，只是折叠起来
		},

		// ==================== File Tip 动画 ====================
		showFileTipMsg() {
			clearTimeout(this._fileTipTimer)
			this.showFileTip = true
			this.$nextTick(() => {
				setTimeout(() => { this.fileTipVisible = true }, 30)
			})
			this._fileTipTimer = setTimeout(() => { this.hideFileTip() }, 4000)
		},
		hideFileTip() {
			this.fileTipVisible = false
			this._fileTipTimer = setTimeout(() => {
				this.showFileTip = false
			}, ANIM.TIP)
		},

		// ==================== 菜单动画 ====================
		toggleMenu() {
			if (this.showMenu) {
				this.closeMenu()
			} else {
				this.openMenu()
			}
		},
		openMenu() {
			this.showMenu = true
			this.$nextTick(() => {
				setTimeout(() => { this.menuAnimIn = true }, 30)
			})
		},
		closeMenu() {
			this.menuAnimIn = false
			setTimeout(() => {
				this.showMenu = false
			}, ANIM.MENU)
		},

		// ==================== 确认弹窗动画 ====================
		showConfirm(title, message, onConfirm, opts = {}) {
			this.confirmDialog.show = true
			this.confirmDialog.title = title
			this.confirmDialog.message = message
			this.confirmDialog.confirmText = opts.confirmText || '确认'
			this.confirmDialog.cancelText = opts.cancelText || '取消'
			this.confirmDialog.danger = !!opts.danger
			this.confirmDialog.onConfirm = onConfirm
			this.$nextTick(() => {
				setTimeout(() => { this.confirmAnimIn = true }, 30)
			})
		},

		closeConfirm() {
			this.confirmAnimIn = false
			setTimeout(() => {
				this.confirmDialog.show = false
				this.confirmDialog.onConfirm = null
			}, ANIM.CONFIRM)
		},

		handleConfirm() {
			if (this.confirmDialog.onConfirm) {
				this.confirmDialog.onConfirm()
			}
			this.closeConfirm()
		},

		// ==================== 导入弹窗动画 ====================
		openAddModal() {
			this.closeMenu()
			setTimeout(() => {
				this.showAddModal = true
				this.$nextTick(() => {
					setTimeout(() => { this.importModalAnimIn = true }, 30)
				})
			}, ANIM.MENU + 50)
		},

		cancelAdd() {
			this.importModalAnimIn = false
			setTimeout(() => {
				this.showAddModal = false
				this.resetFormFields()
			}, ANIM.MODAL)
		},

		resetForm() {
			this.importModalAnimIn = false
			setTimeout(() => {
				this.showAddModal = false
				this.resetFormFields()
			}, ANIM.MODAL)
		},

		resetFormFields() {
			this.formName = ''
			this.formIcon = ''
			this.formFile = ''
			this.formFileName = ''
		},

		// ==================== 删除 ====================
		onLongPress(sound) {
			if (sound.type !== 'custom') return
			this.showConfirm(
				'删除自定义声音',
				`确定要删除「${sound.name}」吗？删除后不可恢复。`,
				() => { this.deleteCustomSound(sound) },
				{ confirmText: '删除', cancelText: '保留', danger: true }
			)
		},

		confirmDeleteCustomSound(sound) {
			this.showConfirm(
				'删除自定义声音',
				`确定要删除「${sound.name}」吗？删除后不可恢复。`,
				() => { this.deleteCustomSound(sound) },
				{ confirmText: '删除', cancelText: '保留', danger: true }
			)
		},

		deleteCustomSound(sound) {
			if (this.activeSound && this.activeSound.id === sound.id) this.stopSound()
			const idx = this.customSounds.findIndex(s => s.id === sound.id)
			if (idx !== -1) {
				this.customSounds.splice(idx, 1)
				this.persistCustomSounds()
			}
		},

		// ==================== 导入 ====================
		pickFile() {
			// #ifdef MP-WEIXIN
			uni.chooseMessageFile({
				count: 1,
				type: 'file',
				extension: ['mp3', 'wav', 'ogg', 'm4a', 'aac', 'flac'],
				success: (res) => {
					this.formFile = res.tempFiles[0].path
					this.formFileName = res.tempFiles[0].name
				}
			})
			// #endif
			// #ifdef APP-PLUS
			this._pickFileAndroid()
			// #endif
			// #ifdef H5
			const input = document.createElement('input')
			input.type = 'file'
			input.accept = 'audio/*'
			input.onchange = (e) => {
				const file = e.target.files[0]
				if (!file) return
				this.formFile = URL.createObjectURL(file)
				this.formFileName = file.name
			}
			input.click()
			// #endif
		},

		// #ifdef APP-PLUS
		// Android 原生文件选择器（带 [Humn] 日志标签）
		_pickFileAndroid() {
			console.log('[Humn] pickFile: 启动文件选择器')
			const invoke = plus.android.invoke
			const main = plus.android.runtimeMainActivity()
			console.log('[Humn] pickFile: main=' + (main ? 'ok' : 'null'))
			const Intent = plus.android.importClass('android.content.Intent')
			const intent = new Intent(Intent.ACTION_OPEN_DOCUMENT)
			invoke(intent, 'addCategory', Intent.CATEGORY_OPENABLE)
			invoke(intent, 'setType', 'audio/*')
			console.log('[Humn] pickFile: intent 创建完成')
			const self = this
			const _origHandler = main.onActivityResult
			console.log('[Humn] pickFile: _origHandler type=' + typeof _origHandler)
			main.onActivityResult = function(requestCode, resultCode, data) {
				console.log('[Humn] onActivityResult: code=' + requestCode + ' result=' + resultCode + ' data=' + (data ? 'ok' : 'null'))
				if (requestCode === 1001) {
					if (resultCode === -1 && data) {
						console.log('[Humn] onActivityResult: 用户选择了文件')
						try {
							self._handleFileResult(main, data)
						} catch(e) {
							console.error('[Humn] _handleFileResult 异常:', e)
						}
					} else {
						console.log('[Humn] onActivityResult: 用户取消选择')
					}
				}
				// 直接调用原始 handler（不用 .call，native bridge 可能不支持）
				if (_origHandler) {
					try {
						_origHandler(requestCode, resultCode, data)
					} catch(e) {
						console.error('[Humn] _origHandler 调用异常:', e)
					}
				}
			}
			console.log('[Humn] pickFile: 启动 startActivityForResult')
			invoke(main, 'startActivityForResult', intent, 1001)
		},

		_handleFileResult(main, data) {
			console.log('[Humn] handleFile: 开始')
			const invoke = plus.android.invoke
			const uri = invoke(data, 'getData')
			console.log('[Humn] handleFile: uri=' + (uri ? String(uri) : 'null'))
			if (!uri) { console.log('[Humn] handleFile: uri 为空，退出'); return }

			// 1. 查文件名和可能的直接路径
			let directPath = null
			let fileName = null
			try {
				const resolver = invoke(main, 'getContentResolver')
				const proj = plus.android.newObject('[Ljava.lang.String;', 2)
				proj[0] = '_data'
				proj[1] = '_display_name'
				const cursor = invoke(resolver, 'query', uri, proj, null, null, null)
				if (cursor && invoke(cursor, 'moveToFirst')) {
					const dataIdx = invoke(cursor, 'getColumnIndex', '_data')
					const nameIdx = invoke(cursor, 'getColumnIndex', '_display_name')
					if (dataIdx >= 0) directPath = invoke(cursor, 'getString', dataIdx)
					if (nameIdx >= 0) fileName = invoke(cursor, 'getString', nameIdx)
					invoke(cursor, 'close')
				}
			} catch(e) { console.log('[Humn] handleFile: 查询异常:', e) }

			// 2. 设置文件路径（直接路径优先，否则存 content URI）
			if (directPath) {
				console.log('[Humn] handleFile: ★ 直接路径: ' + directPath)
				this.formFile = directPath
			} else {
				const uriStr = String(uri)
				console.log('[Humn] handleFile: ★ 存 content URI: ' + uriStr)
				this.formFile = uriStr
			}
			this.formFileName = fileName || ('audio_' + Date.now() + '.m4a')
			console.log('[Humn] handleFile: fileName=' + this.formFileName)
			console.log('[Humn] handleFile: 完成（无阻塞）')
		},
		// #endif

		saveCustomSound() {
			if (!this.canSave) return
			this.customSounds.push({
				id: 'custom_' + Date.now(),
				name: this.formName.trim(),
				icon: this.formIcon || '🎵',
				type: 'custom',
				file: this.formFile
			})
			this.persistCustomSounds()
			this.resetForm()
		},

		// ==================== 主题 ====================
		setTheme(value) {
			this.themeValue = value
			this.applyResolvedTheme()
			try { uni.setStorageSync(THEME_KEY, value) } catch (_) {}
		},

		applyResolvedTheme() {
			this.actualTheme = this.themeValue
			this.updateStatusBar()
		},

		getSystemTheme() {
			// #ifdef APP-PLUS
			// Android 原生 Configuration API，适配 OriginOS 等国产 ROM
			try {
				const config = plus.android.runtimeMainActivity().getResources().getConfiguration()
				if ((config.uiMode & 0x30) === 0x20) return 'dark'
				return 'light'
			} catch(e) {}
			// #endif
			// matchMedia 作为 H5 和 App 端的通用降级
			if (typeof window !== 'undefined' && window.matchMedia) {
				return window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light'
			}
			return 'dark'
		},

		watchSystemTheme() {
			if (typeof window === 'undefined' || !window.matchMedia) return
			const mq = window.matchMedia('(prefers-color-scheme: dark)')
			const handler = () => {
				if (this.themeValue === 'system') {
					this.actualTheme = mq.matches ? 'dark' : 'light'
					this.updateStatusBar()
				}
			}
			mq.addEventListener('change', handler)
			this._themeMediaQuery = { mq, handler }
		},

		unwatchSystemTheme() {
			if (this._themeMediaQuery) {
				this._themeMediaQuery.mq.removeEventListener('change', this._themeMediaQuery.handler)
				this._themeMediaQuery = null
			}
		},

		updateStatusBar() {
			// #ifdef APP-PLUS
			try {
				if (this.actualTheme === 'light') {
					plus.navigator.setStatusBarStyle('dark')
					plus.navigator.setStatusBarBackground('#f4f4f8')
				} else {
					plus.navigator.setStatusBarStyle('light')
					plus.navigator.setStatusBarBackground('#0a0a1a')
				}
			} catch (_) {}
			// #endif
		},

		loadTheme() {
			try {
				const saved = uni.getStorageSync(THEME_KEY)
				if (saved && ['dark', 'light', 'system'].includes(saved)) {
					this.themeValue = saved
				}
			} catch (_) {}
			this.applyResolvedTheme()
		},

		// ==================== 持久化 ====================
		persistCustomSounds() {
			try {
				const data = this.customSounds.map(s => ({
					id: s.id, name: s.name, icon: s.icon, file: s.file, played: s.played
				}))
				uni.setStorageSync(STORAGE_KEY, JSON.stringify(data))
			} catch (e) {
				console.error('保存失败:', e)
			}
		},

		loadCustomSounds() {
			try {
				const raw = uni.getStorageSync(STORAGE_KEY)
				if (raw) {
					this.customSounds = JSON.parse(raw).map(item => ({ ...item, type: 'custom', played: item.played || false }))
				}
			} catch (e) {
				console.error('加载失败:', e)
				this.customSounds = []
			}
		}
	}
}
</script>

<style>
/* ================================================================
   CSS 自定义属性 —— 深色主题
   ================================================================ */
.container.theme-dark {
	--bg-start: #0a0a1a;
	--bg-mid: #1a1a3e;
	--bg-end: #0f0f2a;
	--text-primary: #e8e8f0;
	--text-secondary: #6a6a9a;
	--text-tertiary: #8a8aae;
	--text-sub: #5a5a80;
	--surface: rgba(255, 255, 255, 0.04);
	--surface-paused: rgba(255, 255, 255, 0.06);
	--surface-active: rgba(124, 124, 240, 0.12);
	--accent: #7c7cf0;
	--accent-glow: rgba(124, 124, 240, 0.12);
	--circle-bg: rgba(255, 255, 255, 0.08);
	--circle-active: rgba(124, 124, 240, 0.22);
	--rect-bg: rgba(255, 255, 255, 0.03);
	--menu-bg: #1c1c3a;
	--menu-overlay: rgba(0, 0, 0, 0.55);
	--modal-bg: #1a1a38;
	--modal-overlay: rgba(0, 0, 0, 0.65);
	--input-bg: rgba(255, 255, 255, 0.06);
	--input-text: #e0e0f0;
	--divider: rgba(255, 255, 255, 0.06);
	--wave-idle: #2a2a50;
	--menu-btn-bg: rgba(255, 255, 255, 0.06);
	--menu-btn-line: #8a8ab0;
	--menu-text: #c0c0e0;
	--menu-title: #8a8ab0;
	--file-tip-bg: rgba(124, 124, 240, 0.15);
	--btn-cancel-bg: rgba(255, 255, 255, 0.06);
	--btn-cancel-text: #8a8aae;
	--emoji-bg: rgba(255, 255, 255, 0.05);
	--emoji-selected: rgba(124, 124, 240, 0.25);
	--emoji-border: rgba(124, 124, 240, 0.5);
	--dot-idle: rgba(255, 255, 255, 0.1);
	--dot-paused: #8a8aae;
	--file-picker-bg: rgba(255, 255, 255, 0.06);
	--file-picker-border: rgba(255, 255, 255, 0.12);
	--file-picker-text: #6a6a9a;
	--file-picker-selected: #7c7cf0;
	--menu-delete-bg: rgba(224, 85, 106, 0.12);
	--menu-close-text: #6a6a9a;
	--hint-text: #4a4a6a;
	--danger-btn: #e0556a;
	--danger-btn-bg: rgba(224, 85, 106, 0.15);
	--confirm-msg: #a0a0c0;
}

/* ================================================================
   CSS 自定义属性 —— 浅色主题
   ================================================================ */
.container.theme-light {
	--bg-start: #f4f4f8;
	--bg-mid: #eaeaef;
	--bg-end: #f9f9fc;
	--text-primary: #1a1a2e;
	--text-secondary: #7a7a96;
	--text-tertiary: #666680;
	--text-sub: #8e8ea6;
	--surface: rgba(0, 0, 0, 0.03);
	--surface-paused: rgba(0, 0, 0, 0.04);
	--surface-active: rgba(106, 106, 224, 0.08);
	--accent: #6a6ae0;
	--accent-glow: rgba(106, 106, 224, 0.08);
	--circle-bg: rgba(0, 0, 0, 0.05);
	--circle-active: rgba(106, 106, 224, 0.12);
	--rect-bg: rgba(0, 0, 0, 0.02);
	--menu-bg: #ffffff;
	--menu-overlay: rgba(0, 0, 0, 0.3);
	--modal-bg: #ffffff;
	--modal-overlay: rgba(0, 0, 0, 0.4);
	--input-bg: rgba(0, 0, 0, 0.04);
	--input-text: #1a1a2e;
	--divider: rgba(0, 0, 0, 0.06);
	--wave-idle: #d0d0e0;
	--menu-btn-bg: rgba(0, 0, 0, 0.05);
	--menu-btn-line: #666680;
	--menu-text: #333350;
	--menu-title: #7a7a96;
	--file-tip-bg: rgba(106, 106, 224, 0.1);
	--btn-cancel-bg: rgba(0, 0, 0, 0.05);
	--btn-cancel-text: #7a7a96;
	--emoji-bg: rgba(0, 0, 0, 0.04);
	--emoji-selected: rgba(106, 106, 224, 0.15);
	--emoji-border: rgba(106, 106, 224, 0.4);
	--dot-idle: rgba(0, 0, 0, 0.12);
	--dot-paused: #9999b0;
	--file-picker-bg: rgba(0, 0, 0, 0.03);
	--file-picker-border: rgba(0, 0, 0, 0.1);
	--file-picker-text: #8888a0;
	--file-picker-selected: #6a6ae0;
	--menu-delete-bg: rgba(224, 85, 106, 0.08);
	--menu-close-text: #8888a0;
	--hint-text: #aaaac0;
	--danger-btn: #d94550;
	--danger-btn-bg: rgba(217, 69, 80, 0.08);
	--confirm-msg: #666680;
}

/* ================================================================
   主题过渡动画
   ================================================================ */
.container,
.container .sound-row,
.container .sound-circle,
.container .sound-rect,
.container .menu-panel,
.container .modal-card,
.container .menu-btn,
.container .menu-line,
.container .app-name,
.container .app-tagline,
.container .playing-text,
.container .now-playing,
.container .sound-title,
.container .sound-sub,
.container .menu-item-text,
.container .menu-item-icon,
.container .menu-title,
.container .menu-section-label,
.container .menu-empty text,
.container .modal-title,
.container .confirm-message,
.container .form-label,
.container .form-input,
.container .wave-bar,
.container .status-dot,
.container .file-tip,
.container .file-picker,
.container .file-picker-text,
.container .emoji-item,
.container .btn-cancel,
.container .btn-cancel text,
.container .menu-delete,
.container .custom-dot,
.container .pulse-ring,
.container .menu-divider {
	transition:
		background-color 0.45s ease,
		color 0.3s ease,
		border-color 0.4s ease,
		box-shadow 0.4s ease,
		background 0.45s ease;
}

/* ================================================================
   全局
   ================================================================ */
.container {
	min-height: 100vh;
	background: linear-gradient(180deg, var(--bg-start) 0%, var(--bg-mid) 50%, var(--bg-end) 100%);
	padding: 0 32rpx;
	display: flex;
	flex-direction: column;
}

/* ================================================================
   顶部栏
   ================================================================ */
.top-bar {
	display: flex;
	align-items: flex-start;
	justify-content: space-between;
	padding-top: 36rpx;
	padding-bottom: 16rpx;
}

.app-brand { display: flex; flex-direction: column; }

.app-name {
	font-size: 56rpx;
	font-weight: 700;
	color: var(--text-primary);
	letter-spacing: 4rpx;
	line-height: 1.1;
}

.app-tagline {
	font-size: 24rpx;
	color: var(--text-secondary);
	margin-top: 6rpx;
	letter-spacing: 2rpx;
}

.menu-btn {
	width: 72rpx;
	height: 72rpx;
	border-radius: 20rpx;
	background: var(--menu-btn-bg);
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
	gap: 10rpx;
	margin-top: 4rpx;
}
.menu-btn:active { opacity: 0.7; transform: scale(0.95); }

.menu-line {
	width: 36rpx;
	height: 4rpx;
	background: var(--menu-btn-line);
	border-radius: 2rpx;
}

/* ================================================================
   播放条 — 固定高度，仅淡入淡出，避免下方列表上下跳动
   ================================================================ */
.now-playing {
	display: flex;
	align-items: center;
	height: 56rpx;
	opacity: 0;
	transition: opacity 0.35s ease;
}

.now-playing.in {
	opacity: 1;
}

.playing-wave {
	display: flex;
	align-items: flex-end;
	gap: 5rpx;
	margin-right: 16rpx;
}

.wave-bar {
	width: 5rpx;
	height: 10rpx;
	background: var(--wave-idle);
	border-radius: 3rpx;
}

.wave-bar.active {
	background: var(--accent);
	animation: waveAnim 0.6s ease-in-out infinite alternate;
}
.wave-bar.active:nth-child(1) { animation-delay: 0s; }
.wave-bar.active:nth-child(2) { animation-delay: 0.15s; }
.wave-bar.active:nth-child(3) { animation-delay: 0.3s; }
.wave-bar.active:nth-child(4) { animation-delay: 0.45s; }

@keyframes waveAnim {
	0%   { height: 8rpx; }
	100% { height: 28rpx; }
}

.playing-text {
	font-size: 24rpx;
	color: var(--text-tertiary);
}

/* ================================================================
   声音列表
   ================================================================ */
.sound-list {
	flex: 1;
	margin: 0 -8rpx;
	scrollbar-width: none;
	-ms-overflow-style: none;
}

.sound-list::-webkit-scrollbar {
	display: none;
	width: 0;
	height: 0;
}

.sound-row {
	display: flex;
	align-items: center;
	padding: 18rpx 20rpx;
	margin: 6rpx 8rpx;
	border-radius: 24rpx;
	background: var(--surface);
	transition: background 0.35s ease, box-shadow 0.35s ease, transform 0.2s ease;
}
.sound-row:active { transform: scale(0.98); }

.sound-row.row-playing {
	background: var(--surface-active);
	box-shadow: 0 0 24rpx var(--accent-glow);
}

.sound-row.row-paused {
	background: var(--surface-paused);
}

.sound-circle {
	position: relative;
	width: 96rpx;
	height: 96rpx;
	min-width: 96rpx;
	border-radius: 50%;
	background: var(--circle-bg);
	display: flex;
	align-items: center;
	justify-content: center;
	transition: background 0.35s ease, transform 0.2s ease;
}

.sound-circle.active {
	background: var(--circle-active);
}

.sound-emoji { font-size: 44rpx; position: relative; z-index: 2; }

.custom-dot {
	position: absolute;
	top: 4rpx; right: 8rpx;
	width: 12rpx; height: 12rpx;
	border-radius: 50%;
	background: var(--accent);
	z-index: 3;
}

.pulse-ring {
	position: absolute;
	inset: -6rpx;
	border-radius: 50%;
	border: 2rpx solid rgba(124, 124, 240, 0.4);
	animation: pulse 1.5s ease-out infinite;
	z-index: 1;
}
.pulse-ring.delay { animation-delay: 0.75s; }

@keyframes pulse {
	0%   { transform: scale(0.88); opacity: 0.7; }
	100% { transform: scale(1.18); opacity: 0; }
}

.sound-rect {
	flex: 1;
	margin-left: 22rpx;
	background: var(--rect-bg);
	border-radius: 16rpx;
	padding: 16rpx 20rpx;
	display: flex;
	flex-direction: column;
	gap: 4rpx;
	transition: background 0.35s ease;
}

.sound-title {
	font-size: 30rpx; font-weight: 500;
	color: var(--text-primary);
}

.sound-sub { font-size: 22rpx; color: var(--text-sub); }

.sound-row.row-playing .sound-sub { color: var(--text-tertiary); }

.sound-status {
	width: 48rpx;
	display: flex;
	align-items: center; justify-content: center;
	margin-left: 8rpx;
}

.status-dot { width: 14rpx; height: 14rpx; border-radius: 50%; }

.playing-dot {
	background: var(--accent);
	box-shadow: 0 0 10rpx rgba(124, 124, 240, 0.6);
	animation: dotPulse 1.2s ease-in-out infinite;
}

@keyframes dotPulse {
	0%, 100% { opacity: 1; transform: scale(1); }
	50%      { opacity: 0.5; transform: scale(0.7); }
}

.paused-dot { background: var(--dot-paused); }
.idle-dot   { background: var(--dot-idle); }

/* ================================================================
   底部提示 — 淡入上浮 / 淡出下落
   ================================================================ */
.file-tip {
	position: fixed;
	bottom: 40rpx; left: 40rpx; right: 40rpx;
	background: var(--file-tip-bg);
	border-radius: 16rpx;
	padding: 24rpx 32rpx;
	text-align: center;
	opacity: 0;
	transform: translateY(24rpx);
	pointer-events: none;
	transition: opacity 0.35s ease, transform 0.35s cubic-bezier(0.34, 1.56, 0.64, 1);
}

.file-tip.in {
	opacity: 1;
	transform: translateY(0);
	pointer-events: auto;
}

.file-tip text {
	font-size: 24rpx;
	color: var(--text-tertiary);
	line-height: 1.6;
}

/* ================================================================
   菜单面板 — 右侧滑入 / 滑出
   ================================================================ */
.menu-overlay {
	position: fixed; inset: 0;
	background: transparent;
	z-index: 1000;
	display: flex;
	justify-content: flex-end;
	padding: 20rpx;
	transition: background 0.35s ease;
	pointer-events: none;
}
.menu-overlay.in {
	background: var(--menu-overlay);
	pointer-events: auto;
}

.menu-panel {
	width: 520rpx;
	background: var(--menu-bg);
	border-radius: 20rpx;
	padding: 28rpx 0;
	align-self: flex-start;
	box-shadow: 0 12rpx 40rpx rgba(0, 0, 0, 0.4);
	transform: translateX(120%);
	transition: transform 0.35s cubic-bezier(0.4, 0, 0.2, 1);
}
.menu-panel.in {
	transform: translateX(0);
}

.menu-title {
	font-size: 28rpx; font-weight: 600;
	color: var(--menu-title);
	padding: 0 28rpx 20rpx;
	display: block;
}

.menu-section-label {
	font-size: 24rpx;
	color: var(--menu-title);
	padding: 12rpx 28rpx 6rpx;
	display: block;
}

.menu-divider {
	height: 1rpx;
	background: var(--divider);
	margin: 8rpx 28rpx;
}

.menu-item {
	display: flex; align-items: center;
	padding: 18rpx 28rpx;
	transition: background 0.15s ease;
}
.menu-item:active { background: var(--surface); }

.menu-item-icon {
	font-size: 30rpx;
	margin-right: 18rpx;
	width: 44rpx;
	text-align: center;
	color: var(--accent);
}

.menu-item-text {
	font-size: 28rpx;
	color: var(--menu-text);
	flex: 1;
}

.theme-item .menu-item-icon { color: var(--accent); font-size: 22rpx; }

.menu-delete {
	width: 56rpx; height: 56rpx;
	display: flex;
	align-items: center; justify-content: center;
	border-radius: 14rpx;
	background: var(--menu-delete-bg);
	transition: transform 0.15s ease, background 0.2s ease;
}
.menu-delete:active { transform: scale(0.9); }

.menu-delete text { font-size: 28rpx; }

.menu-empty { padding: 24rpx 28rpx; }
.menu-empty text { font-size: 24rpx; color: var(--hint-text); }

.close-item { justify-content: center; }
.close-item .menu-item-text {
	color: var(--menu-close-text);
	text-align: center;
	flex: none;
}

/* ================================================================
   弹窗 — 淡入 + 弹性缩放
   ================================================================ */
.modal-overlay {
	position: fixed; inset: 0;
	background: transparent;
	z-index: 1100;
	display: flex;
	align-items: center; justify-content: center;
	padding: 60rpx;
	transition: background 0.35s ease;
	pointer-events: none;
}
.modal-overlay.in {
	background: var(--modal-overlay);
	pointer-events: auto;
}

.modal-card {
	width: 100%;
	max-width: 600rpx;
	background: var(--modal-bg);
	border-radius: 24rpx;
	padding: 44rpx 36rpx 32rpx;
	max-height: 85vh;
	overflow-y: auto;
	opacity: 0;
	transform: scale(0.9) translateY(30rpx);
	transition:
		opacity 0.38s ease,
		transform 0.38s cubic-bezier(0.34, 1.56, 0.64, 1);
}
.modal-card.in {
	opacity: 1;
	transform: scale(1) translateY(0);
}

.modal-title {
	font-size: 36rpx; font-weight: 600;
	color: var(--text-primary);
	display: block;
	margin-bottom: 20rpx;
}

.confirm-message {
	font-size: 28rpx;
	color: var(--confirm-msg);
	line-height: 1.6;
	display: block;
	margin-bottom: 32rpx;
}

.modal-card .modal-title + .form-group { margin-top: 16rpx; }

.form-group { margin-bottom: 28rpx; }

.form-label {
	font-size: 26rpx;
	color: var(--text-secondary);
	display: block;
	margin-bottom: 12rpx;
}

.form-input {
	width: 100%; height: 80rpx;
	background: var(--input-bg);
	border-radius: 12rpx;
	padding: 0 24rpx;
	font-size: 28rpx;
	color: var(--input-text);
	box-sizing: border-box;
}

.form-hint {
	font-size: 22rpx; color: var(--hint-text);
	margin-top: 8rpx;
	display: block;
}

.emoji-grid { display: flex; flex-wrap: wrap; gap: 12rpx; }

.emoji-item {
	width: 64rpx; height: 64rpx;
	border-radius: 16rpx;
	background: var(--emoji-bg);
	display: flex;
	align-items: center; justify-content: center;
	transition: background 0.2s ease, box-shadow 0.2s ease, transform 0.15s ease;
}
.emoji-item:active { transform: scale(0.9); }

.emoji-item.selected {
	background: var(--emoji-selected);
	box-shadow: 0 0 0 2rpx var(--emoji-border);
}

.emoji-char { font-size: 32rpx; }

.file-picker {
	display: flex; align-items: center; gap: 16rpx;
	background: var(--file-picker-bg);
	border-radius: 12rpx;
	padding: 22rpx 24rpx;
	border: 1rpx dashed var(--file-picker-border);
	transition: background 0.2s ease, border-color 0.2s ease;
}
.file-picker:active { background: var(--surface); }

.file-picker-icon { font-size: 36rpx; }

.file-picker-text { font-size: 26rpx; color: var(--file-picker-text); }
.file-picker-text.file-selected { color: var(--file-picker-selected); }

.modal-actions { display: flex; gap: 20rpx; margin-top: 36rpx; }

.btn {
	flex: 1; height: 84rpx;
	border-radius: 16rpx;
	display: flex;
	align-items: center; justify-content: center;
	transition: transform 0.15s ease, background 0.25s ease, opacity 0.25s ease;
}
.btn:active { transform: scale(0.96); }

.btn text { font-size: 30rpx; font-weight: 500; }

.btn-cancel { background: var(--btn-cancel-bg); }
.btn-cancel text { color: var(--btn-cancel-text); }

.btn-save { background: var(--accent); }
.btn-save text { color: #fff; }

.btn-save.disabled {
	background: rgba(124, 124, 240, 0.25);
}

.btn-save.disabled text {
	color: rgba(255, 255, 255, 0.3);
}

.btn-danger { background: var(--danger-btn-bg); }
.btn-danger text { color: var(--danger-btn); font-weight: 600; }
</style>
