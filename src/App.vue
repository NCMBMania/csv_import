<template>
  <v-app>
    <v-app-bar
      app
      color="primary"
      dark
    >
    </v-app-bar>

    <v-main>
      <v-container class="grey lighten-5">
        <v-row>
          <v-col
            md="8"
            offset-md="2"
          >
            <h2>Monacaバックエンドからニフクラ mobile backendへの移行ツール</h2>
            <p>このツールではアプリケーションキーとクライアントキーを利用しますが、キーは保存していません。安心してご利用ください。</p>
            <v-row>
              <v-col>
                <form>
                  <v-text-field
                    v-model="applicationKey"
                    :error-messages="applicationKeyErrors"
                    label="アプリケーションキー"
                    required
                  ></v-text-field>
                  <v-text-field
                    v-model="clientKey"
                    :error-messages="clientKeyErrors"
                    label="クライアントキー"
                    required
                  ></v-text-field>
                  <h3>データストアへのアップロードファイル(CSVファイルをドロップしてください）
                  </h3>
                  <Dropfile @success="success">
                    ファイルアップロード
                  </Dropfile>
                  <div v-if="Object.keys(files).length > 0">
                    <h4>取り込み予定のファイル</h4>
                    <v-alert
                      outlined
                      color="red"
                    >
                      <div>
                        フィールド名が次のように変わるので注意してください
                        <ul>
                          <li>createDate → oldCreateDate</li>
                          <li>updateDate → oldUpdateDate</li>
                          <li>objectId → oldObjectId</li>
                        </ul>
                      </div>
                    </v-alert>
                    <v-list>
                      <v-list-item two-line v-for="(contents, className) in files" :key="className">
                        <v-list-item-content>
                          <v-list-item-title>{{ className }} <v-icon @click="deleteFile(className)">mdi-delete</v-icon></v-list-item-title>
                          <v-list-item-subtitle>取り込み予定行数 {{ contents.length }}</v-list-item-subtitle>
                        </v-list-item-content>
                      </v-list-item>
                    </v-list>
                  </div>
                  <h3>デバイストークンのアップロードファイル（プッシュ通知のエクスポートしたファイルをドラッグ&amp;ドロップしてください。圧縮されている場合は伸張してください）</h3>
                  <Dropfile @success="successInstallation">
                    ファイルアップロード
                  </Dropfile>
                  <div v-if="installations.length > 0">
                    <h4>取り込み予定のファイル</h4>
                    <v-list>
                      <v-list-item tree-line>
                        <v-list-item-content>
                          <v-list-item-title>プッシュ通知 <v-icon @click="deleteInstallation">mdi-delete</v-icon></v-list-item-title>
                          <v-list-item-subtitle>取り込み予定行数 {{ installations.length }}</v-list-item-subtitle>
                        </v-list-item-content>
                      </v-list-item>
                    </v-list>
                  </div>
                  <v-row v-if="message">
                    <v-col>
                      <v-alert
                        outlined
                        color="blue"
                      >
                        <div v-html="message">
                        </div>
                      </v-alert>
                    </v-col>
                  </v-row>
                  <v-row>
                    <v-col>
                      <v-btn
                        color="info"
                        @click="upload"
                      >
                       アップロードを開始する
                      </v-btn>
                    </v-col>
                  </v-row>
                </form>
              </v-col>
            </v-row>
          </v-col>
        </v-row>
      </v-container>
    </v-main>
  </v-app>
</template>

<script lang="ts">
import Vue from 'vue';
// @ts-ignore
import Dropfile from './components/Dropfile'
// @ts-ignore
import { NCMB, NCMBObject, NCMBInstallation, NCMBQuery } from 'ncmb_ts'

export default Vue.extend({
  name: 'App',
  components: {
    Dropfile
  },
  data: () => ({
    applicationKey: '',
    clientKey: '',
    applicationKeyErrors: '',
    clientKeyErrors: '',
    files: {},
    installations: [],
    message: {type: String},
    ncmb: {type: NCMB}
  }),
  methods: {
    success(contents: { [s: string]: any }[]) {
      for (const key in contents) {
        this.$set(this.files, key, contents[key])
      }
    },
    deleteFile(fileName: string) {
      this.$delete(this.files, fileName);
    },
    successInstallation(contents: { [s: string]: any }) {
      const first = Object.keys(contents)[0];
      this.$set(this, 'installations', contents[first])
    },
    deleteInstallation() {
      this.$set(this, 'installations', [])
    },
    async upload() {
      // @ts-ignore
      this.log('アップロード処理を開始します')
      try {
        // @ts-ignore
        this.ncmb = new NCMB(this.applicationKey, this.clientKey);
        // コネクションチェック
        const query = new NCMBQuery('Hello');
        await query.limit(1).fetch();
        // データストアのアップロード
        for (const className in this.files) {
          // @ts-ignore
          this.log(`${className}にデータを追加します`)
          // @ts-ignore
          await this.import(className, this.files[className])
          // @ts-ignore
          this.log(`${className}へのデータ追加完了しました`)
        }
        // デバイストークンのアップロード
        if (this.installations.length > 0) {
          // @ts-ignore
          this.log(`デバイストークンをInstallationsに登録します`)
          // @ts-ignore
          await this.importInstallation()
        }
      } catch (err) {
        // @ts-ignore
        this.log(JSON.stringify(err.message))
        // @ts-ignore
        this.log('エラーが発生しました')
      }
    },
    async importInstallation() {
      // @ts-ignore
      for (const row of this.installations) {
        // @ts-ignore
        for (const field of ['objectId', 'createDate', 'updateDate']) {
          row[`old${field[0].toUpperCase()}${field.slice(1)}`] = row[field]
          delete row[field]
        }
        const install = new NCMBInstallation;
        await install.sets(row).save();
        // @ts-ignore
        this.log(`デバイストークン追加完了。Installations - objectId ${install.get('objectId')}`)
      }
    },

    async import(className: string, rows: { [s: string]: any }) {
      // @ts-ignore
      for (const row of rows) {
        const obj = new NCMBObject(className)
        for (const field of ['objectId', 'createDate', 'updateDate']) {
          row[`old${field[0].toUpperCase()}${field.slice(1)}`] = row[field]
          delete row[field]
        }
        await obj.sets(row).save()
        // @ts-ignore
        this.log(`データ追加完了。${className} - objectId ${obj.get('objectId')}`)
      }
    },
    log(message: string) {
      // @ts-ignore
      this.message = `${message}<br />${this.message === null ? '': this.message }`;
    }
  }
});
</script>
