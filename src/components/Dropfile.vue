<template>
  <div
    class="drop_area"
    @dragenter="dragEnter"
    @dragleave="dragLeave"
    @dragover.prevent
    @drop.prevent="dropFile"
    :class="{enter: isEnter}"
  >
    ファイルアップロード
  </div>
</template>

<script lang="ts">
import Vue from 'vue';
// @ts-ignore
import Papa from 'papaparse';

// @ts-ignore
const fileRead = (file) => {
  return new Promise((res, rej) => {
    try {
      const reader = new FileReader()
      reader.onload = res
      reader.readAsText(file)
    } catch (err) {
      rej(err)
    }
  })

}

export default Vue.extend({
  name: 'Dropfile',
  data() {
    return {
      isEnter: false
    }
  },
  methods: {
    dragEnter() {
      this.isEnter = true;
    },
    dragLeave() {
      this.isEnter = false;
    },
    // @ts-ignore
    async dropFile(e) {
      const files = e.dataTransfer.files
      const params = {}
      for (const f of files) {
        const res = await fileRead(f)
        // @ts-ignore
        params[f.name.replace(/\.csv$/, '')] = this.parse(Papa.parse(res.target.result, {header: true}).data);
        console.log(params[f.name.replace(/\.csv$/, '')])
      }
      this.$emit('success', params)
      this.isEnter = false
    },
    parse(ary) {
      const results = [];
      for (const params of ary) {
        const values = {};
        for (const column in params) {
          const config = column.split(':');
          const value = params[column];
          if (config.length === 1) {
            values[column] = value;
          }
          switch (config[1]) {
            case 'String':
              values[config[0]] = value;
              break;
            case 'Boolean':
              if (value.toUpperCase() === 'TRUE') {
                values[config[0]] = true;
              }
              if (value.toUpperCase() === 'FALSE') {
                values[config[0]] = false;
              }
              break;
            case 'Date':
              if (value && value !== '') {
                const d = new Date(value)
                if (d.toString() !== 'Invalid Date') {
                  values[config[0]] = {
                    __type: 'Date',
                    iso: d.toISOString()
                  }
                }
              }
              break;
            case 'Number':
              if (value && value !== '' && !Number.isNaN(parseFloat(value))) {
                values[config[0]] = parseFloat(value);
              }
              break;
            case 'Array':
            case 'Object':
              try {
                values[config[0]] = JSON.parse(value);
              } catch (e) {

              }
              break;
          }
        }
        results.push(values);
      }
      return results;
    }
  }
});
</script>

<style scoped>
.drop_area {
  color: gray;
  font-weight: bold;
  font-size: 1.2em;
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 200px;
  border: 5px solid gray;
  border-radius: 15px;
}

.enter {
  border: 10px dotted powderblue;
}
</style>