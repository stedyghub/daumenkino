<template lang="pug">
.daumenkino(:class="classes", :style="style")
	Content
	.progress-bar-rail
		.progress-bar
	template(v-if="speakerMode")
		.clock {{ elapsedTime }}
		portal-target.daumenkino-notes-portal-target(name="notes")
</template>
<script>
import { mapState, mapGetters } from 'vuex'
import isEqual from 'lodash/isEqual'

export default {
	data () {
		return {
			startTime: Date.now(),
			currentTime: Date.now()
		}
	},
	computed: {
		...mapState(['slides', 'activePath', 'overview', 'speakerMode']),
		...mapGetters(['activeSlide', 'nextPath', 'previousPath']),
		theme () {
			return this.$site.themeConfig?.theme || 'white'
		},
		totalSlides () {
			return this.slides.reduce((acc, slide) => {
				acc += slide.nestedSlides.length || 1
				return acc
			}, 0)
		},
		progress () {
			let pastActive = false
			const count = (acc, slide) => {
				if (pastActive) return acc
				if (slide === this.activeSlide) {
					pastActive = true
					return acc
				}
				return acc + (slide.nestedSlides.length > 0 ? slide.nestedSlides.reduce(count, 0) : 1)
			}
			return this.slides.reduce(count, 0)
		},
		classes () {
			const classes = [
				`daumenkino-theme-${this.theme}`,
				`slide-${(this.activePath[0] || 0) + (this.slides[this.activePath[0]]?.nestedSlides[this.activePath[1]] ? `-${this.activePath[1]}` : '')}`,
				`slide--${this.progress}`,
				{
					'speaker-mode': this.speakerMode,
					overview: this.overview,
					'slide-first': !this.previousPath,
					'slide-last': !this.nextPath
				}
			]
			if (this.activeSlide?.fragments.length > 0) {
				classes.push(`fragment-${this.activeSlide.shownFragments}`)
			}
			return classes
		},
		style () {
			return {
				'--slides-total': this.totalSlides,
				'--slides-active': this.progress,
				'--active-slide-x': this.activePath[0] || 0,
				'--active-slide-y': this.slides[this.activePath[0]]?.nestedSlides[this.activePath[1]] ? this.activePath[1] : 0
			}
		},
		elapsedTime () {
			const totalSeconds = Math.floor((this.currentTime - this.startTime) / 1000)
			const minutes = Math.floor(totalSeconds / 60)
			const seconds = totalSeconds % 60
			return `${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`
		}
	},
	watch: {
		activePath () {
			this.sendState()
			this.saveURL()
			this.$emit('pathChanged', this.activePath)
		}
	},
	mounted () {
		const findSlides = (parent) => {
			if (!parent) {
				console.error('COULD NOT FIND SLIDES')
				return
			}
			this.$nextTick(() => {
				if (parent.$children.some(c => c._isSlide)) {
					const children = Array.from(parent.$el.children)
					this.$store.commit('setSlides', parent.$children.filter(slide => slide._isSlide).sort((a, b) => children.indexOf(a.$el) - children.indexOf(b.$el)))
					this.$nextTick(() => {
						this.loadURL()
					})
				} else {
					this.$nextTick(() => {
						findSlides(parent.$children.find(child => child.$el.getRootNode() === document))
					})
				}
			})
		}
		findSlides(this)

		document.addEventListener('keydown', this.globalKeyHandler)
		window.addEventListener('message', this.globalMessageHandler)
		// report to parent
		window.opener?.postMessage(['loaded'])
		this._timerInterval = setInterval(() => this.currentTime = Date.now(), 1000)
	},
	beforeDestroy () {
		clearInterval(this._timerInterval)
		document.removeEventListener('keydown', this.globalKeyHandler)
	},
	methods: {
		next () {
			if (this.nextPath) {
				this.$store.commit('setActivePath', this.nextPath)
			}
		},
		previous () {
			if (this.previousPath) {
				this.$store.commit('setActivePath', this.previousPath)
			}
		},
		toggleSpeakerMode () {
			if (!this.speakerMode) {
				this._speakerWindow = window.open(window.location.pathname, 'Daumenkino Speaker Mode', 'height=900,width=1600')
			}
		},
		toggleOverview () {
			this.$store.commit('setOverview', !this.overview)
		},
		globalKeyHandler (event) {
			switch (event.key) {
				case ' ': {
					event.preventDefault()
					this.next()
					break
				}
				case 'Backspace': {
					event.preventDefault()
					this.previous()
					break
				}
				case 'PageDown':
				case 'ArrowRight': {
					event.preventDefault()
					if (this.overview && (this.activePath[0] || 0) < this.slides.length - 1) {
						this.$store.commit('setActivePath', [(this.activePath[0] || 0) + 1])
					} else {
						this.next()
					}
					break
				}
				case 'PageUp':
				case 'ArrowLeft': {
					event.preventDefault()
					if (this.overview && this.activePath[0] > 0) {
						this.$store.commit('setActivePath', [this.activePath[0] - 1])
					} else {
						this.previous()
					}
					break
				}
				case 'ArrowUp': {
					event.preventDefault()
					if (this.overview && this.slides[this.activePath[0]]?.nestedSlides.length > 0 && this.activePath[1] > 0) {
						this.previous()
					}
					break
				}
				case 'ArrowDown': {
					event.preventDefault()
					if (this.overview && this.slides[this.activePath[0]]?.nestedSlides.length > 0 && (this.activePath[1] || 0) < this.slides[this.activePath[0]]?.nestedSlides.length - 1) {
						this.next()
					}
					break
				}
				case 's': {
					event.preventDefault()
					this.toggleSpeakerMode()
					break
				}
				case 't': {
					event.preventDefault()
					this.startTime = Date.now()
					break
				}
				case 'Escape': {
					event.preventDefault()
					this.toggleOverview()
					break
				}
				case 'Enter': {
					event.preventDefault()
					this.$store.commit('setOverview', false)
					break
				}
				case 'Home': {
					event.preventDefault()
					this.$store.commit('setActivePath', [])
					break
				}
				case 'End': {
					event.preventDefault()
					this.$store.commit('setActivePath', [this.slides.length - 1])
					break
				}
			}
		},
		// Speaker Mode Methods
		globalMessageHandler (event) {
			switch (event.data[0]) {
				case 'loaded': {
					this._speakerWindow.postMessage(['initSpeakerMode'])
					this.sendState()
					break
				}
				case 'initSpeakerMode': {
					this.$store.commit('setSpeakerMode', true)
					break
				}
				case 'updateState': {
					if (isEqual(this.activePath, event.data[1].activePath)) return
					this._receivedChange = true
					this.$store.commit('setActivePath', event.data[1].activePath)
					break
				}
			}
		},
		sendState () {
			if (window === undefined) return
			const otherWindow = this._speakerWindow || (this.speakerMode && window.opener)
			if (!otherWindow) return
			// don't send change back
			if (this._receivedChange) {
				this._receivedChange = false
				return
			}
			otherWindow.postMessage(['updateState', {
				activePath: this.activePath
			}])
		},
		loadURL () {
			this.$store.commit('setActivePath', window.location.hash.substr(1).split('/').map(n => Number(n)))
		},
		saveURL () {
			window.location.hash = this.activePath.join('/')
		}
	}
}
</script>
<style src="prismjs/themes/prism-tomorrow.css"></style>
<style src="../styles/theme.styl" lang="stylus"></style>
