<template>
  <video
    ref="streamingElement"
    autoplay
    playsinline
    muted
    :style="cameraStyle"
    :crossorigin="crossorigin"
    @play="updateStatus('connected')"
    @error="updateStatus('error')"
  />
</template>

<script lang="ts">
import { Component, Ref, Mixins } from 'vue-property-decorator'
import { consola } from 'consola'
import CameraMixin from '@/mixins/camera'
import sleep from '@/util/sleep'

type RTCConfigurationWithSdpSemantics = RTCConfiguration & {
  sdpSemantics: 'unified-plan'
}

@Component({})
export default class WebrtcCrealityk2Camera extends Mixins(CameraMixin) {
  @Ref('streamingElement')
  readonly cameraVideo!: HTMLVideoElement

  pc: RTCPeerConnection | null = null
  remoteId: string | null = null
  playbackAbortController: AbortController | null = null
  sleepAbortController: AbortController | null = null

  // adapted from https://github.com/ayufan/camera-streamer/blob/4203f89df1596cc349b0260f26bf24a3c446a56b/html/webrtc.html

  async loadStream () {
    this.pc?.close()

    const abortControllerSignal = this.playbackAbortController?.signal

    if (!abortControllerSignal || abortControllerSignal.aborted) {
      return
    }

    this.updateStatus('connecting')

    const url = this.buildAbsoluteUrl(this.camera.stream_url || '')

    this.updateRawCameraUrl(url.toString())

    try {
      const config: RTCConfigurationWithSdpSemantics = {
        sdpSemantics: 'unified-plan',
        iceServers: [
          {
            urls: [
              'stun:stun.l.google.com:19302'
            ]
          }
        ]
      }

      const pc = this.pc = new RTCPeerConnection(config)

      pc.addTransceiver('video', {
        direction: 'recvonly'
      })

      pc.ontrack = (event: RTCTrackEvent) => {
        if (event.track.kind === 'video') {
          this.cameraVideo.srcObject = event.streams[0]
        }
      }

      pc.onicecandidate = async (event: RTCPeerConnectionIceEvent) => {
        if (event.candidate === null) {
          try {
            const offer = pc.localDescription
            const response = await fetch(`${url}call/webrtc_local`, {
              body: btoa(JSON.stringify({
                type: offer?.type,
                sdp: offer?.sdp
              })),
              headers: {
                'Content-Type': 'plain/text'
              },
              method: 'POST',
              signal: abortControllerSignal
            })
            const res = await response.text()
            const jsonRes = JSON.parse(atob(res)) as any
            if (jsonRes.type === 'answer') {
              await pc.setRemoteDescription(new RTCSessionDescription(jsonRes as RTCSessionDescriptionInit))
            }
          } catch (e) {
            consola.error('[WebrtcCrealityk2Camera] onicecandidate', e)
          }
        }
      }

      const rtcLocalSessionDescriptionInit = await pc.createOffer()

      await pc.setLocalDescription(rtcLocalSessionDescriptionInit)
    } catch (e) {
      consola.error(`[WebrtcCrealityk2Camera] failed to start playback "${this.camera.name}"`, e)

      this.onError()
    }
  }

  async onError () {
    this.updateStatus('error')
    this.pc?.close()
    this.pc = null

    const playbackAbortSignal = this.playbackAbortController?.signal

    if (!playbackAbortSignal || playbackAbortSignal.aborted) {
      return
    }

    this.sleepAbortController?.abort()

    const sleepAbortController = this.sleepAbortController = new AbortController()

    try {
      const signals = [
        playbackAbortSignal,
        sleepAbortController.signal,
      ]

      await sleep(2000, AbortSignal.any(signals))

      this.loadStream()
    } catch {}
  }

  async startPlayback () {
    this.playbackAbortController = new AbortController()

    await this.loadStream()
  }

  stopPlayback () {
    this.updateStatus('disconnected')
    this.playbackAbortController?.abort()
    this.playbackAbortController = null

    this.pc?.close()
    this.pc = null
    this.cameraVideo.src = ''
    this.cameraVideo.srcObject = null
  }
}
</script>
