<template>
  <div class="modal-card box box-modal">
    <header class="box-modal-header is-spaced">
      <div class="box-modal-title">{{ currentStepTitle }}</div>
      <button type="button" class="delete" @click="$emit('close')" />
    </header>

    <!-- Step 1: Acceptance -->
    <div v-if="currentStep === 1">
      <div class="content">
        <p></p>
        <p class="has-text-weight-bold">Please draw a funny mustache on Kim Jong Un</p>
        <div class="drawing-container" ref="drawingContainer">
          <img ref="baseImage" src="@/assets/img/kimmy.jpg" alt="deposit-modal-1" @load="initializeCanvas" />
          <canvas
            ref="drawingCanvas"
            class="drawing-canvas"
            @mousedown="startDrawing"
            @mousemove="draw"
            @mouseup="stopDrawing"
            @mouseleave="stopDrawing"
            @touchstart="handleTouch"
            @touchmove="handleTouch"
            @touchend="stopDrawing"
          ></canvas>
        </div>
        <div class="field mt-4">
          <b-checkbox v-model="hasDrawnMustache">
            I solemnly swear that I have drawn a funny mustache and that I believe Kim Jong Un is retarded
          </b-checkbox>
        </div>
        <div class="field">
          <b-label>Say something mean about Kim Jong Un</b-label>
          <b-input v-model="mustacheDescription" placeholder="Kim is a very fat man" maxlength="100" />
        </div>
      </div>
      <b-button type="is-primary is-fullwidth" :disabled="!canProceed" @click="proceedToNextStep">
        Proceed
      </b-button>
    </div>

    <!-- Step 2: Main Deposit Content -->
    <div v-else>
      <div class="note">
        <div>{{ $t('pleaseBackupYourNote') }}</div>
        <div>{{ $t('treatYourNote') }}</div>
      </div>
      <div class="znote">
        {{ prefix }}-{{ note }}
        <b-tooltip :label="tooltipCopy" position="is-top">
          <button
            v-clipboard:copy="`${prefix}-${note}`"
            v-clipboard:success="onCopy"
            class="button is-primary has-icon"
          >
            <span class="icon icon-copy"></span>
          </button>
        </b-tooltip>
        <b-tooltip :label="$t('saveNote')" position="is-top">
          <button class="button is-primary has-icon" @click="onSave">
            <span class="icon icon-save"></span>
          </button>
        </b-tooltip>
      </div>
      <div v-show="isEnabledSaveFile" class="note">
        {{ $t('saveAsFile') }} <span class="has-text-primary">{{ filename }}</span>
      </div>
      <template v-if="!isSetupAccount">
        <i18n tag="div" path="yourDontHaveAccount" class="notice">
          <template v-slot:account>
            <a @click="_redirectToAccount">{{ $t('account.button') }}</a>
          </template>
        </i18n>
      </template>
      <b-checkbox v-if="isSetupAccount" v-model="isEncrypted">
        <i18n v-show="isSetupAccount" tag="div" path="iEncryptedTheNote">
          <template v-slot:address>
            <b-tooltip :label="tooltipCopy" position="is-top">
              <a class="has-text-primary" @click.prevent.stop="copyNoteAccount">{{ getEncryptAccount }}</a>
            </b-tooltip>
          </template>
        </i18n>
      </b-checkbox>
      <template v-if="!isSetupAccount || !isEncrypted">
        <b-checkbox v-model="isBackuped" data-test="backup_note_checkbox">{{
          $t('iBackedUpTheNote')
        }}</b-checkbox>
        <div v-if="isBackuped && isIPFS" class="notice is-warning">
          <div class="notice__p">{{ $t('yourNoteWontBeSaved') }}</div>
        </div>
      </template>
      <connect-button v-if="!isLoggedIn" type="is-primary is-fullwidth" />
      <b-button
        v-else
        type="is-primary is-fullwidth"
        :disabled="disableButton"
        data-test="send_deposit_button"
        @click="_sendDeposit"
      >
        {{ $t('sendDeposit') }}
      </b-button>
    </div>
  </div>
</template>
<script>
/* eslint-disable no-console */
import { mapActions, mapState, mapGetters } from 'vuex'

import { sliceAddress } from '@/utils'
import { ConnectButton } from '@/components/web3Connect'

export default {
  components: {
    ConnectButton
  },
  data() {
    return {
      currentStep: 1,
      hasDrawnMustache: false,
      mustacheDescription: '',
      isBackuped: false,
      tooltipCopy: this.$t('clickToCopy'),
      isEncrypted: false,
      copyTimer: null,
      // Drawing related data
      isDrawing: false,
      brushSize: 3,
      brushColor: '#000000',
      hasDrawn: false,
      ctx: null,
      lastX: 0,
      lastY: 0
    }
  },
  computed: {
    ...mapGetters('metamask', ['isLoggedIn']),
    ...mapGetters('txHashKeeper', ['addressExplorerUrl']),
    ...mapGetters('encryptedNote', ['isSetupAccount', 'accounts', 'isEnabledSaveFile']),
    ...mapState('application', ['note', 'prefix']),
    isValidMustacheDescription() {
      const text = this.mustacheDescription.toLowerCase()
      return (
        text.includes('retarded') ||
        text.includes('fat') ||
        text.includes('stupid') ||
        text.includes('dumb') ||
        text.includes('idiot') ||
        text.includes('dumbass') ||
        text.includes('retard') ||
        text.includes('fatass') ||
        text.includes('fatty')
      )
    },
    canProceed() {
      return (
        this.hasDrawnMustache &&
        this.mustacheDescription.trim().length > 0 &&
        this.hasDrawn &&
        this.isValidMustacheDescription
      )
    },
    currentStepTitle() {
      return this.currentStep === 1 ? 'Anti North Korean Countermeasures Activated!' : this.$t('yourNote')
    },
    isIPFS() {
      return this.$isLoadedFromIPFS()
    },
    filename() {
      return `backup-${this.prefix}-${this.note.slice(0, 10)}.txt`
    },
    getEncryptAccount() {
      return sliceAddress(this.accounts.encrypt)
    },
    disableButton() {
      if (this.isBackuped) {
        return !this.isBackuped
      } else {
        return !this.isEncrypted || !this.isSetupAccount
      }
    }
  },
  beforeMount() {
    this.isEncrypted = this.isSetupAccount
  },
  beforeDestroy() {
    clearTimeout(this.copyTimer)
  },
  methods: {
    ...mapActions('application', ['sendDeposit', 'saveFile']),
    ...mapActions('encryptedNote', ['redirectToAccount']),
    proceedToNextStep() {
      this.currentStep = 2
    },
    onCopy() {
      this.tooltipCopy = this.$t('copied')
      this.copyTimer = setTimeout(() => {
        this.tooltipCopy = this.$t('clickToCopy')
      }, 1500)
    },
    onSave() {
      this.saveFile({ note: this.note, prefix: this.prefix })
    },
    _redirectToAccount() {
      this.redirectToAccount()
      this.$emit('close')
    },
    async _sendDeposit() {
      this.$store.dispatch('loading/enable', { message: this.$t('preparingTransactionData') })
      await this.sendDeposit({ isEncrypted: this.isEncrypted })
      this.$store.dispatch('loading/disable')
      this.$parent.close()
    },
    async copyNoteAccount() {
      await this.$copyText(this.accounts.encrypt)
      this.onCopy()
    },
    initializeCanvas() {
      const canvas = this.$refs.drawingCanvas
      const img = this.$refs.baseImage
      const container = this.$refs.drawingContainer

      // Set canvas size to match image
      canvas.width = img.width
      canvas.height = img.height
      container.style.width = `${img.width}px`
      container.style.height = `${img.height}px`

      // Get context and set initial properties
      this.ctx = canvas.getContext('2d')
      this.ctx.lineCap = 'round'
      this.ctx.lineJoin = 'round'
    },
    startDrawing(e) {
      this.isDrawing = true
      const { offsetX, offsetY } = this.getCoordinates(e)
      this.lastX = offsetX
      this.lastY = offsetY
    },
    draw(e) {
      if (!this.isDrawing) return

      const { offsetX, offsetY } = this.getCoordinates(e)
      this.ctx.beginPath()
      this.ctx.strokeStyle = this.brushColor
      this.ctx.lineWidth = this.brushSize
      this.ctx.moveTo(this.lastX, this.lastY)
      this.ctx.lineTo(offsetX, offsetY)
      this.ctx.stroke()

      this.lastX = offsetX
      this.lastY = offsetY
      this.hasDrawn = true
    },
    stopDrawing() {
      this.isDrawing = false
    },
    handleTouch(e) {
      e.preventDefault()
      const touch = e.touches[0]
      const rect = this.$refs.drawingCanvas.getBoundingClientRect()
      const offsetX = touch.clientX - rect.left
      const offsetY = touch.clientY - rect.top

      if (e.type === 'touchstart') {
        this.startDrawing({ offsetX, offsetY })
      } else if (e.type === 'touchmove') {
        this.draw({ offsetX, offsetY })
      }
    },
    getCoordinates(e) {
      if (e.offsetX) return e
      const rect = this.$refs.drawingCanvas.getBoundingClientRect()
      return {
        offsetX: e.clientX - rect.left,
        offsetY: e.clientY - rect.top
      }
    },
    clearCanvas() {
      if (this.ctx) {
        this.ctx.clearRect(0, 0, this.$refs.drawingCanvas.width, this.$refs.drawingCanvas.height)
        this.hasDrawn = false
      }
    }
  }
}
</script>

<style scoped>
.drawing-container {
  position: relative;
  margin: 0 auto;
}

.drawing-container img {
  display: block;
  max-width: 100%;
  height: auto;
}

.drawing-canvas {
  position: absolute;
  top: 0;
  left: 0;
  cursor: crosshair;
}

.drawing-controls {
  display: flex;
  justify-content: center;
  gap: 1rem;
}
</style>
