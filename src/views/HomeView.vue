<template>
  <div style="margin: 30px; min-width: 700px">
    <h1 style="margin-bottom: 20px">
      이세돌 클립 백업
    </h1>
    <div>
      <!-- <div>클립매핑정보 (필요한사람은 쓰세요)</div> -->
      <div style="margin-top: 5px; max-width: 450px; display:flex;">
        <div>
          <v-select
            v-model="select"
            prepend-inner-icon="mdi-content-save"
            :items="items"
            label="클립정보"
            dense
            outlined
          ></v-select>
        </div>
        <div style="display: flex;">
          <v-btn style="margin-left: 10px" @click="downloadClipInfo" :disabled="isDisabled">다운로드</v-btn>
          필요한 사람은 쓰세요
        </div>
      </div>
    </div>
    <v-form ref="form" lazy-validation>
      <div style="display: flex; align-items: center;">
        <v-text-field style="max-width: 100px"
          id="textField"
          label="조회수"
          v-model="vc"
          placeholder="100"
          :rules="rules"
        ></v-text-field>
        이상의 클립을 검색합니다.
      </div>
      아래 버튼을 클릭하여 클립 검색
      <div style="margin: 10px 0px">
        <v-btn small id="viichan" name="비챤" @click="search" class="btn">비챤</v-btn>
        <v-btn small id="gosegu" name="고세구"  @click="search" class="btn">고세구</v-btn>
        <v-btn small id="jururu" name="주르르"  @click="search" class="btn">주르르</v-btn>
        <v-btn small id="lilpa" name="릴파"  @click="search" class="btn">릴파</v-btn>
        <v-btn small id="jingburger" name="징버거"  @click="search" class="btn">징버거</v-btn>

        <v-btn small id="ine" name="아이네"  @click="search" class="btn">아이네</v-btn>
        <v-btn small id="woowakgood" name="우왁굳"  @click="search" class="btn">우왁굳</v-btn>
      </div>
    </v-form>

    <div v-if="loading">
      로딩중
      <v-progress-circular
        indeterminate
        color="black"
      ></v-progress-circular>
    </div>
    <v-card>
      <v-tabs
        v-model="tab"
        background-color="indigo"
        centered
        dark
        icons-and-text
      >
        <v-tabs-slider></v-tabs-slider>
        <v-tab href="#1">
          대기
          <v-icon>mdi-checkbox-blank-outline</v-icon>
        </v-tab>

        <v-tab href="#2">
          성공
          <v-icon color="green">mdi-checkbox-marked</v-icon>
        </v-tab>

        <v-tab href="#3">
          실패
          <v-icon color="red">mdi-close-box-outline</v-icon>
        </v-tab>
      </v-tabs>

      <v-tabs-items v-model="tab">
        <v-tab-item :value="'1'">
          <div style="background-color: #3f51b5; color: white; text-align: center;">
            {{ download.standby.length }}개 클립 대기중
            <v-spacer></v-spacer>
            <v-btn small color="primary" :disabled="downloading" @click="startDownload">다운로드 시작</v-btn>
            <v-btn small color="red" :disabled="!downloading" @click="cancelDownload">다운로드 중지</v-btn>
          </div>
          <div v-for="link in download.standby" :key="link">
            <v-icon>mdi-checkbox-blank-outline</v-icon>
            {{ link }}
          </div>
        </v-tab-item>
        <v-tab-item :value="'2'">
          <div style="background-color: #3f51b5; color: white; text-align: center;">{{ download.complete.length }}개 클립 다운로드 완료</div>
          <div v-for="link in download.complete" :key="link">
            <v-icon color="green">mdi-checkbox-marked</v-icon>
            {{ link }}
          </div>
        </v-tab-item>
        <v-tab-item :value="'3'">
          <div style="background-color: #3f51b5; color: white; text-align: center;">
            {{ download.fail.length }}개 클립 다운로드 실패
            <v-btn small color="primary" :disabled="downloading" @click="redownloadFaliClip">다운로드 재시도</v-btn>
          </div>
          <div v-for="link in download.fail" :key="link">
            <v-icon color="red">mdi-close-box-outline</v-icon>
            {{ link }}
          </div>
        </v-tab-item>
      </v-tabs-items>
    </v-card>
  </div>
</template>
<script>
/* eslint-disable */
export default {
  components: {},
  data() {
    return {
      bg: '',
      select: '',
      bid: '',
      bName: '',
      vc: '10000',
      loading: false,
      downloading: false,
      download: {
        standby: [],
        inProgress: [],
        complete: [],
        fail: [],
      },
      downloadCnt: 0,
      rules: [
        v => !!v || '필수항목입니다.',
        v => Number.isInteger(Number.parseInt(v)) || '숫자만 입력가능합니다.',
        v => v >= 100 || '100 이상만 입력 가능합니다.'
      ],
      items: [
        'viichan_clipInfo.json', 'viichan_clipInfo_pretty.json',
        'gosegu_clipInfo.json', 'gosegu_clipInfo_pretty.json',
        'jururu_clipInfo.json', 'jururu_clipInfo_pretty.json',
        'lilpa_clipInfo.json', 'lilpa_clipInfo_pretty.json',
        'jingburger_clipInfo.json', 'jingburger_clipInfo_pretty.json',
        'ine_clipInfo.json', 'ine_clipInfo_pretty.json',
        'woowakgood_clipInfo.json', 'woowakgood_clipInfo_pretty.json'
      ],
      tab: '',
      text: 'testtext',
      timer: 0,
      interval: ''
    }
  },
  watch: {
    timer: function() {
      this.watchDownloading()
    }
  },
  created() {
    this.bg = chrome.extension.getBackgroundPage()
  },
  methods: {
    search(e) {
      if (!this.$refs.form.validate()) {
        return
      }
      const vue = this
      this.loading = true
      this.bName = e.currentTarget.id
      fetch('https://danylee.xyz/clips?bName=' + this.bName + '&viewCount=' + this.vc)
      // fetch('http://localhost/clips?bName=' + this.bName + '&viewCount=' + this.vc)
      .then(resp => {
        vue.bg.console.log('resp: ', resp)
        resp.json().then(data => {
          if (resp.status === 200) {
            const set = new Set(data.dto)
            vue.download.standby = [...set]
            vue.loading = false
          }
        })
      })
      .catch(err => {
        vue.bg.console.log(err)
        vue.loading = false
      })
      this.tab = '1'
    },
    startDownload() {
      this.downloading = true
      this.interval = setInterval(() => this.timer++, 1000);
      this.addDownload()
    },
    addDownload() {
      let ms = 0
      while (this.downloadCnt < 3) {
        this.downloadCnt++
        const link = this.download.standby[0]
        this.download.standby.splice(0, 1)
        this.callDownload(link)
      }
    },
    callDownload(link) {
      const vue = this
      chrome.downloads.download({ url: link }, function(id) {
        vue.download.inProgress.push({ 'id': id, 'link': link })
      })
    },
    watchDownloading() {
      const vue = this
      this.download.inProgress.forEach(p => {
        chrome.downloads.search({ 'id': p.id }, function(search) {
          if (search[0].state === 'interrupted') {
            vue.download.fail.push(p.link)
            vue.afterInProgress(p)
          } else if (search[0].state === 'complete') {
            vue.download.complete.push(p.link)
            vue.afterInProgress(p)
          }
        })
      })
    },
    afterInProgress(p) {
      const i = this.download.inProgress.indexOf(p)
      this.download.inProgress.splice(i, 1)
      this.downloadCnt--
      this.addDownload()
    },
    cancelDownload() {
      clearInterval(this.interval)
      this.downloading = false
    },
    redownloadFaliClip() {
      if (confirm('다운로드 대기중, 다운로드 진행중 상태가 초기화됩니다')) {
        this.bg.console.log('init')
        this.downloading = false
        this.download.standby = this.download.fail
        this.download.fail = []
        this.download.complete = []
        this.download.inProgress = []
        this.downloadCnt = 0
        this.tab = 0
      }
    },
    downloadClipInfo() {
      if (this.items.indexOf(this.select) !== -1) {
        window.open('https://danylee.xyz/file?fileName=' + this.select, '_self')
      }
      // window.open('http://localhost/file?fileName=' + this.select, '_self')
    }
  }
}
</script>
<style scoped>
.btn {
  margin-left: 10px
}
</style>
