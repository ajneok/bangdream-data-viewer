<template>
  <div>
    <p v-html="$t('declaration')"></p>
    <div v-if="isReady">
      <div class="row">
        <div class="col-lg-6 col-xs-12">
          <q-progress indeterminate v-if="audioDisable" />
          <div class="row gutter justify-center items-center">
            <q-btn class="col-3" @click="startRhythm(), $ga.event('music-score', 'start', 'view', data.musicId)" :disable="(audioDisable) || (audioStarted && !audioPaused)">Start</q-btn>
            <q-btn class="col-3" @click="pauseRhythm(), $ga.event('music-score', 'pause', 'view', data.musicId)" :disable="!audioStarted">Pause</q-btn>
            <q-btn class="col-3" @click="stopRhythm(), $ga.event('music-score', 'stop', 'view', data.musicId)" :disable="!audioStarted">Stop</q-btn>
            <q-select class="col-3" float-label="Difficulty" v-model="difficulty" :options="difficultyOptions"></q-select>
          </div>
          <canvas ref="game"></canvas>
        </div>
        <div class="col-lg-6 col-xs-12">
          <h3>{{data.title}}</h3>
          <p>{{$t('composer')}}: {{data.composer}}</p>
          <p>{{$t('lyricist')}}: {{data.lyricist}}</p>
          <p>{{$t('arranger')}}: {{data.arranger}}</p>
          <p>{{$t('band')}}:
            <span v-if="Number(data.bandId) > 5">{{data.bandName}}</span>
            <img height="60px" width="100px" v-if="Number(data.bandId) <= 5" v-lazy="`/assets/band/logo/00${data.bandId}_logoL.png`">
          </p>
          <p>{{$t('combo')}}: {{data.combo}}</p>
        </div>
      </div>
    </div>
    <q-spinner v-else color="pink" size="48px"></q-spinner>
  </div>
</template>

<i18n>
{
  "en": {
    "composer": "Composer",
    "lyricist": "Lyricist",
    "arranger": "Arranger",
    "band": "Band",
    "combo": "Total notes",
    "declaration": "This is still a test version, and not fit for mobile view. If you find any problem, please contact me through <a href='mailto:dev@bangdream.ga'>email</a> or on <a href='https://discord.gg/vGb3eHH'>Discord</a>."
  },
  "zh-cn": {
    "composer": "作曲",
    "lyricist": "作词",
    "arranger": "编曲",
    "band": "演奏者",
    "combo": "音符数",
    "declaration": "仍在测试中，未适配移动设备，欢迎通过<a href='mailto:dev@bangdream.ga'>email</a>或者<a href='https://discord.gg/vGb3eHH'>Discord</a>向我反映问题。"
  },
  "zh-tw": {
    "composer": "作曲",
    "lyricist": "作詞",
    "arranger": "編曲",
    "band": "演奏者",
    "combo": "音符數",
    "declaration": "仍在測試中，未適配行動設備，歡迎通過<a href='mailto:dev@bangdream.ga'>email</a>或者<a href='https://discord.gg/vGb3eHH'>Discord</a>向我回報問題。"
  },
  "ja": {
    "composer": "作曲",
    "lyricist": "作詞",
    "arranger": "編曲",
    "band": "出演者",
    "combo": "トータルノート",
    "declaration": "これはまだテスト版です、モバイルビューに適合しない。問題が見つかった場合は、<a href='mailto:dev@bangdream.ga'>メール</a>または<a href='https://discord.gg/vGb3eHH'>Discord</a>で私に連絡してください。"
  }
}
</i18n>

<script>
/* global PIXI */
import {
  QBtn,
  QIcon,
  QSpinner,
  QProgress,
  QSelect
} from 'quasar'
import { mapState, mapActions } from 'vuex'
import BufferLoader from 'assets/bufferLoader'

export default {
  name: 'music-score-modal',
  components: {
    QBtn,
    QIcon,
    QSpinner,
    QProgress,
    QSelect
  },
  data () {
    return {
      difficulty: 'expert',
      audioDisable: true,
      audioStarted: false,
      audioPaused: false,
      audioContext: new AudioContext(),
      noteStartPoint: [
        {
          x: 50,
          y: 50
        }, {
          x: 120,
          y: 50
        }, {
          x: 190,
          y: 50
        }, {
          x: 260,
          y: 50
        }, {
          x: 330,
          y: 50
        }, {
          x: 400,
          y: 50
        }, {
          x: 470,
          y: 50
        }
      ],
      noteEndPoint: [
        {
          x: 50,
          y: 750
        }, {
          x: 120,
          y: 750
        }, {
          x: 190,
          y: 750
        }, {
          x: 260,
          y: 750
        }, {
          x: 330,
          y: 750
        }, {
          x: 400,
          y: 750
        }, {
          x: 470,
          y: 750
        }
      ],
      currentNotes: [[], [], [], [], [], [], []],
      typeNoteMap: {
        Single: 'statics/gameres/note_normal_3.png',
        Flick: 'statics/gameres/note_flick_3.png',
        FeverFlick: 'statics/gameres/note_flick_3.png',
        Long: 'statics/gameres/note_long_3.png',
        Slide_A: 'statics/gameres/note_slide_among.png',
        Slide_B: 'statics/gameres/note_slide_among.png',
        Slide_End_A: 'statics/gameres/note_slide_among.png',
        Slide_End_B: 'statics/gameres/note_slide_among.png',
        Slide_End_Flick_A: 'statics/gameres/note_flick_3.png',
        Slide_End_Flick_B: 'statics/gameres/note_flick_3.png',
        Skill: 'statics/gameres/note_skill_3.png',
        LongLine: 'statics/gameres/longNoteLine.png'
      },
      containerNotes: [[], [], [], [], [], [], []],
      isReady: false,
      data: {},
      leftLaneBase: 10,
      currBaseTime: 0,
      difficultyOptions: [
        {
          label: 'Expert',
          value: 'expert'
        },
        {
          label: 'Hard',
          value: 'hard'
        },
        {
          label: 'Normal',
          value: 'normal'
        },
        {
          label: 'Easy',
          value: 'easy'
        }
      ]
    }
  },
  mounted () {
    this.$nextTick(async () => {
      await this.getMusicById({musicId: this.$route.params.musicId, server: 'jp'})
      this.data = this.musicMap.jp[this.$route.params.musicId]
      this.isReady = true

      this.loadRes()
    })
  },
  computed: {
    ...mapState('music', [
      'musicMap'
    ])
  },
  destroyed () {
    this.audioContext.close()
  },
  methods: {
    ...mapActions('music', [
      'getMusicById'
    ]),
    renderStage () {
      const bg = new PIXI.Graphics()
      bg.lineStyle(2, 0xd4d4d4)
        .moveTo(this.leftLaneBase, 0).lineTo(this.leftLaneBase, 750)
        .moveTo(this.leftLaneBase + 70, 0).lineTo(this.leftLaneBase + 70, 750)
        .moveTo(this.leftLaneBase + 70 * 2, 0).lineTo(this.leftLaneBase + 70 * 2, 750)
        .moveTo(this.leftLaneBase + 70 * 3, 0).lineTo(this.leftLaneBase + 70 * 3, 750)
        .moveTo(this.leftLaneBase + 70 * 4, 0).lineTo(this.leftLaneBase + 70 * 4, 750)
        .moveTo(this.leftLaneBase + 70 * 5, 0).lineTo(this.leftLaneBase + 70 * 5, 750)
        .moveTo(this.leftLaneBase + 70 * 6, 0).lineTo(this.leftLaneBase + 70 * 6, 750)
        .moveTo(this.leftLaneBase + 70 * 7, 0).lineTo(this.leftLaneBase + 70 * 7, 750)
        .moveTo(0, 750).lineTo(600, 750)
      this.app.stage.addChild(bg)

      this.lane1Container = new PIXI.Container()
      this.lane1Container.x = this.leftLaneBase
      this.lane1Container.width = 70
      this.lane1Container.height = 750
      this.app.stage.addChild(this.lane1Container)

      this.lane2Container = new PIXI.Container()
      this.lane2Container.x = this.leftLaneBase + 70
      this.lane2Container.width = 70
      this.lane2Container.height = 750
      this.app.stage.addChild(this.lane2Container)

      this.lane3Container = new PIXI.Container()
      this.lane3Container.x = this.leftLaneBase + 70 * 2
      this.lane3Container.width = 70
      this.lane3Container.height = 750
      this.app.stage.addChild(this.lane3Container)

      this.lane4Container = new PIXI.Container()
      this.lane4Container.x = this.leftLaneBase + 70 * 3
      this.lane4Container.width = 70
      this.lane4Container.height = 750
      this.app.stage.addChild(this.lane4Container)

      this.lane5Container = new PIXI.Container()
      this.lane5Container.x = this.leftLaneBase + 70 * 4
      this.lane5Container.width = 70
      this.lane5Container.height = 750
      this.app.stage.addChild(this.lane5Container)

      this.lane6Container = new PIXI.Container()
      this.lane6Container.x = this.leftLaneBase + 70 * 5
      this.lane6Container.width = 70
      this.lane6Container.height = 750
      this.app.stage.addChild(this.lane6Container)

      this.lane7Container = new PIXI.Container()
      this.lane7Container.x = this.leftLaneBase + 70 * 6
      this.lane7Container.width = 70
      this.lane7Container.height = 750
      this.app.stage.addChild(this.lane7Container)

      this.app.renderer.render(this.app.stage)
    },
    playSound (buffer, time) {
      // console.log(time, this.audioContext.currentTime)
      const source = this.audioContext.createBufferSource()
      source.buffer = buffer
      source.connect(this.audioContext.destination)
      source.start(time)
    },
    loadRes () {
      this.bufferLoader = new BufferLoader(
        this.audioContext,
        [
          `/assets/sound/${this.data.bgmId}.mp3`,
          '/statics/hihat.wav',
          '/statics/flick.mp3'
        ],
        () => { this.audioDisable = false }
      )

      this.bufferLoader.load()
    },
    startRhythm () {
      if (!this.app) {
        this.app = new PIXI.Application(510, 800, {
          view: this.$refs.game
        })

        if (Object.keys(PIXI.loader.resources).length) {
          this.renderStage()
        }
        else {
          PIXI.loader.add([
            'statics/gameres/note_normal_3.png',
            'statics/gameres/note_flick_3.png',
            'statics/gameres/note_long_3.png',
            'statics/gameres/note_skill_3.png',
            'statics/gameres/note_slide_among.png',
            'statics/gameres/longNoteLine.png'
          ]).load(this.renderStage)
        }
      }
      if (this.audioPaused) {
        this.audioContext.resume()
        this.audioPaused = false
        return
      }
      this.$http.get(`/api/v1/jp/music/chart/${this.$route.params.musicId}/${this.difficulty}`)
        .then(res => res.json())
        .then(res => {
          const baseTime = this.audioContext.currentTime + 0.1
          this.data.combo = res.length - 1
          res.forEach(note => {
            switch (note.type) {
              case 'Music':
                this.playSound(this.bufferLoader.bufferList[0], baseTime + note.timing)
                break
              default:
                if (note.type.indexOf('Flick') !== -1) this.playSound(this.bufferLoader.bufferList[2], baseTime + note.timing)
                else this.playSound(this.bufferLoader.bufferList[1], baseTime + note.timing)
                if (note.endTiming) this.playSound(this.bufferLoader.bufferList[1], baseTime + note.endTiming)
                break
            }
          })

          // prepare note lane data
          const noteLaneArr = [
            res.filter(note => note.column === 'SC'),
            res.filter(note => note.column === '1'),
            res.filter(note => note.column === '2'),
            res.filter(note => note.column === '3'),
            res.filter(note => note.column === '4'),
            res.filter(note => note.column === '5'),
            res.filter(note => note.column === '6')
          ]
          this.currBaseTime = baseTime
          this.tickerFunc = this.drawNotes.bind(this, noteLaneArr, this.bufferLoader.bufferList[0].duration)
          this.app.ticker.add(this.tickerFunc)

          this.audioStarted = true
        })
    },
    drawNotes (noteLaneArr, maxLength) {
      if (this.audioPaused) return
      this.drawLaneNotes(noteLaneArr[0], this.lane1Container, this.currBaseTime, 0)
      this.drawLaneNotes(noteLaneArr[1], this.lane2Container, this.currBaseTime, 1)
      this.drawLaneNotes(noteLaneArr[2], this.lane3Container, this.currBaseTime, 2)
      this.drawLaneNotes(noteLaneArr[3], this.lane4Container, this.currBaseTime, 3)
      this.drawLaneNotes(noteLaneArr[4], this.lane5Container, this.currBaseTime, 4)
      this.drawLaneNotes(noteLaneArr[5], this.lane6Container, this.currBaseTime, 5)
      this.drawLaneNotes(noteLaneArr[6], this.lane7Container, this.currBaseTime, 6)

      if (this.audioContext.currentTime - this.currBaseTime > maxLength) this.app.ticker.remove(this.tickerFunc)
    },
    drawLaneNotes (noteArr, container, baseTime, laneIndex) {
      const comingNotes = noteArr.filter(note => {
        const noteTime = note.timing - (this.audioContext.currentTime - baseTime)
        const noteEndTime = note.endTiming - (this.audioContext.currentTime - baseTime)
        return (noteTime > 0 && noteTime < 1) || (noteEndTime > 0 && noteTime < 1)
      })

      // check new notes
      if (!this.currentNotes[laneIndex].length && comingNotes.length) {
        // all new, put them
        comingNotes.forEach(note => {
          container.addChild(this.drawNewNote(note))

          this.containerNotes[laneIndex].push(note)
        })

        this.currentNotes[laneIndex] = comingNotes
        // console.log(comingNotes, this.currentNotes, this.containerNotes, laneIndex)
      }
      else if (!comingNotes.length) {
        // all old, remove them
        this.currentNotes[laneIndex].forEach(() => {
          container.removeChildAt(0)
          this.containerNotes[laneIndex].splice(0, 1)
        })
        this.currentNotes[laneIndex] = []
      }
      else {
        // check if next notes and current notes are matched
        const nextFirstNote = comingNotes[0]
        const nextFirstNoteOldIndex = this.currentNotes[laneIndex].findIndex(note => note.index === nextFirstNote.index)
        if (!nextFirstNoteOldIndex) {
          // fist note matches, check if more notes appeared
          if (comingNotes.length !== this.currentNotes[laneIndex].length) {
            // not matched, then add it
            const newNotes = comingNotes.slice(this.currentNotes[laneIndex].length)
            // console.log('new', newNotes, comingNotes, this.currentNotes[laneIndex])
            newNotes.forEach(note => {
              container.addChild(this.drawNewNote(note))

              this.containerNotes[laneIndex].push(note)
            })
          }
        }
        else if (nextFirstNoteOldIndex === -1) {
          // a note appears when another disappears
          this.currentNotes[laneIndex].forEach(() => {
            container.removeChildAt(0)
            this.containerNotes[laneIndex].splice(0, 1)
          })
          comingNotes.forEach(note => {
            container.addChild(this.drawNewNote(note))

            this.containerNotes[laneIndex].push(note)
          })
        }
        else {
          // first note not matches, then remove old notes
          const oldNotes = this.currentNotes[laneIndex].slice(0, nextFirstNoteOldIndex)
          // console.log('old', oldNotes)
          oldNotes.forEach(() => {
            container.removeChildAt(0)
            this.containerNotes[laneIndex].splice(0, 1)
          })

          // check if new notes are there
          if (this.currentNotes[laneIndex].length - nextFirstNoteOldIndex !== comingNotes.length) {
            // got new notes
            const newNotes = comingNotes.slice(-(comingNotes.length - (this.currentNotes[laneIndex].length - nextFirstNoteOldIndex)))
            newNotes.forEach(note => {
              container.addChild(this.drawNewNote(note))

              this.containerNotes[laneIndex].push(note)
            })
          }
        }
        // console.log('tick')

        this.currentNotes[laneIndex] = comingNotes
      }

      // for debug: validity data
      if (JSON.stringify(this.currentNotes[laneIndex]) !== JSON.stringify(this.containerNotes[laneIndex])) {
        console.log(this.currentNotes[laneIndex], this.containerNotes[laneIndex], laneIndex)
        this.audioContext.close()
        this.app.ticker.remove(this.tickerFunc)
      }

      // update notes position
      this.currentNotes[laneIndex].forEach((note, idx) => {
        const drawNote = container.getChildAt(idx)
        const noteTime = note.timing - (this.audioContext.currentTime - baseTime)
        // const noteEndTime = note.endTiming - (this.audioContext.currentTime - baseTime)

        // let the note drop down for 1s
        drawNote.x = 0
        drawNote.y = 736 * (1 - noteTime)
      })
    },
    drawNewNote (note) {
      const noteAsset = this.typeNoteMap[note.type] || this.typeNoteMap.Single
      let ret
      if (note.endTiming) {
        ret = new PIXI.Container()
        const startNote = new PIXI.Sprite(PIXI.loader.resources[noteAsset].texture)
        startNote.x = 0
        startNote.y = 0
        startNote.width = 70
        startNote.height = 27.3
        ret.addChild(startNote)

        const endNote = new PIXI.Sprite(PIXI.loader.resources[this.typeNoteMap.Long].texture)
        endNote.x = 0
        endNote.y = 750 * (note.timing - note.endTiming)
        endNote.width = 70
        endNote.height = 27.3
        ret.addChild(endNote)

        const longLine = new PIXI.Sprite(PIXI.loader.resources[this.typeNoteMap.LongLine].texture)
        longLine.x = 0
        longLine.y = 750 * (note.timing - note.endTiming) + 27.3 / 2
        longLine.width = 70
        longLine.height = startNote.y - endNote.y
        ret.addChild(longLine)
      }
      else {
        ret = new PIXI.Sprite(PIXI.loader.resources[noteAsset].texture)
        ret.x = 0
        ret.y = 0
        ret.width = 70
        ret.height = 27.3
      }

      return ret
    },
    stopRhythm () {
      this.tmpDeltaTime = 0
      this.currBaseTime = 0
      this.audioContext.close()
      this.audioContext = new AudioContext()
      this.app.ticker.remove(this.tickerFunc)
      this.audioStarted = false
      this.audioPaused = false
    },
    pauseRhythm () {
      this.audioContext.suspend()
      this.audioStarted = true
      this.audioPaused = true
    }
  }
}
</script>

<style lang="stylus" scoped>

</style>
