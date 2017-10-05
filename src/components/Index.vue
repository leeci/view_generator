<template>
  <div>
    <button type="button"
            @click="$refs.upelm.click($event)">匯入TrendTag_VC</button>
    <button type="button" @click="toCsvEvent">導出合併csv</button>
    <button type="button" @click="toViewEvent">導出資料庫View</button>
    <p>資料庫：
      <select v-model="selection.db">
        <option v-for="item in getDbList" :value="item.value">{{ item.label }}</option>
      </select>
    </p>
    <p>
      <button @click="removeEvent">&lt;</button>
      <button @click="addEvent">&gt;</button>
    </p>
    <template v-if="selection.db !== ''">
      <p>
        <select v-for="(item, index) in selData[selection.db].groups"
                @focus="focusEvent(index)"
                v-model="selection.groups[index].sels" multiple style="width: 100px;">
          <option v-for="item2 in groups[index].sels" :value="item2">{{ item2 }}</option>
        </select>
      </p>
      <p>
        <input type="text" v-for="(item, index) in selData[selection.db].groups"
               v-model="selection.groups[index].name"
               @keyup="item.name=selection.groups[index].name"
               style="width: 96px">
      </p>
    </template>
    <input type="file"
           style="display: none;"
           ref="upelm"
           accept=".csv"
           @change="selectFileEvent">
    <textarea v-model="output"
              style="width: 100%; height: 800px;"></textarea>
  </div>
</template>

<script>
export default {
  data () {
    return {
      rawData: '',
      output: '',
      struct: {},
      splitNum: 0,
      selection: {
        focus: 0,
        db: '',
        groups: []
      },
      selData: {}
    }
  },
  computed: {
    getDbList () {
      var dbList = Object.keys(this.struct)
      var list = []

      if (dbList.length === 0) {
        return [{
          label: '無資料',
          value: ''
        }]
      }

      for (var i = 0; i < dbList.length; i++) {
        list.push({
          label: dbList[i],
          value: dbList[i]
        })
      }

      return list
    },
    groups: {
      get () {
        if (this.selection.db === '') {
          return []
        }

        return this.selData[this.selection.db].groups
      },
      set (newVal) {
        if (this.selection.db !== '') {
          this.selData[this.selection.db].groups = newVal
        }
      }
    }
  },
  methods: {
    selectFileEvent (event) {
      var vm = this
      var input = event.srcElement

      if (input.files && input.files[0]) {
        var reader = new FileReader()

        reader.onload = function (e) {
          vm.rawData = e.target.result
        }

        reader.readAsText(input.files[0])
      }
    },
    addEvent () {
      var db = this.selData[this.selection.db]
      var focus = this.selection.focus
      var sel = this.selection.groups[focus].sels

      if (typeof db === 'undefined') {
        return
      }

      if (db.groups[focus].sels.length - sel.length <= 0) {
        if (typeof db.groups[focus + 1] !== 'undefined') {
          this.groups[focus].sels = this.groups[focus + 1].sels.concat(this.groups[focus].sels)
          this.groups.splice(focus + 1, 1)
          this.selection.groups.splice(focus + 1, 1)
        }

        return
      }

      if (typeof db.groups[focus + 1] === 'undefined') {
        db.groups.push({
          sels: [],
          name: ''
        })

        this.selection.groups.push({
          sels: [],
          name: ''
        })
      }

      for (var i = 0; i < sel.length; i++) {
        var index = this.groups[focus].sels.indexOf(sel[i])

        if (index >= 0) {
          var elm = this.groups[focus].sels.splice(index, 1)
          this.groups[focus + 1].sels.push(elm[0])
        }
      }

      this.selection.groups[focus + 1].sels = this.selection.groups[focus].sels
      this.selection.groups[focus].sels = []
    },
    removeEvent () {
      var db = this.selData[this.selection.db]
      var focus = this.selection.focus
      var sel = this.selection.groups[focus].sels

      if (typeof db === 'undefined') {
        return
      }

      if (db.groups[focus].length - sel.length <= 0) {
        if (typeof db.groups[focus - 1] !== 'undefined') {
          this.groups[focus - 1].sels = this.groups[focus - 1].sels.concat(this.groups[focus].sels)
          this.groups.splice(focus, 1)
          this.selection.groups.splice(focus - 1, 1)
        }

        return
      }

      for (var i = 0; i < sel.length; i++) {
        var index = this.groups[focus].sels.indexOf(sel[i])

        if (index >= 0) {
          var elm = this.groups[focus].sels.splice(index, 1)
          this.groups[focus - 1].sels.push(elm[0])
        }
      }

      this.selection.groups[focus - 1].sels = this.selection.groups[focus].sels
      this.selection.groups[focus].sels = []
    },
    toCsvEvent () {
      var str = ''

      for (var dbName in this.selData) {
        var groups = this.selData[dbName].groups

        for (var i = 0; i < groups.length; i++) {
          str += `Database,${dbName},Table,${groups[i].name},,,,,,,,,,,,\r\n`

          for (var j = 0; j < groups[i].sels.length; j++) {
            var detail = this.struct[dbName][groups[i].sels[j]]

            for (var k = 0; k < detail.length; k++) {
              var fName = detail[k].FieldName + '_' + groups[i].sels[j]
              str += `OPC_Server,${detail[k].OPC_Server},IPAddress,${detail[k].IPAddress},OPCTag,${detail[k].OPCTag},FieldName,${fName},FailValue,${detail[k].FailValue},Scale,${detail[k].Scale},Offset,${detail[k].Offset},String Len,${detail[k]['String Len']}\r\n`
            }
          }
        }
      }

      this.output = str
    },
    toViewEvent () {
      var str = ''

      for (var dbName in this.selData) {
        var groups = this.selData[dbName].groups

        for (var i = 0; i < groups.length; i++) {
          for (var j = 0; j < groups[i].sels.length; j++) {
            var detail = this.struct[dbName][groups[i].sels[j]]

            str += `CREATE VIEW ${groups[i].sels[j]} AS SELECT Time_Stamp,Time_Stamp_ms,Bias,Data_Hour,Data_Minute,Data_Second,\r\n`

            for (var k = 0; k < detail.length; k++) {
              var fName = detail[k].FieldName + '_' + groups[i].sels[j]

              str += `[${fName}] as ${groups[i].sels[j]}\r\n`
            }

            str += `FROM ${dbName}.dbo.${groups[i].name}\r\nGO\r\n`
          }
        }
      }

      this.output = str
    },
    focusEvent (index) {
      this.selection.focus = index
    }
  },
  watch: {
    rawData (val, oldVal) {
      var arr = val.split('\r\n')
      var struct = {}
      var db = null
      var table = null

      // 往下讀取每一排
      for (var i = 0; i < arr.length; i++) {
        var colum = arr[i].split(',')

        if (colum[1] === undefined) {
          continue
        }
        // 讀取機器名稱
        if (colum[0] === 'Database') {
          db = colum[1]
          table = colum[3]

          if (typeof struct[db] === 'undefined') {
            struct[db] = {}
          }

          if (typeof struct[db][table] === 'undefined') {
            struct[db][table] = []
          }
        }

        if (colum[0] === 'OPC_Server' && (db !== null || table !== null)) {
          struct[db][table].push({
            OPC_Server: colum[1],
            IPAddress: colum[3],
            OPCTag: colum[5],
            FieldName: colum[7],
            FailValue: colum[9],
            Scale: colum[11],
            Offset: colum[13],
            'String Len': colum[15]
          })
        }
      }

      this.struct = struct
    },
    getDbList (val, oldVal) {
      this.selData = {}

      if (val[0].value === '') {
        return
      }

      for (var i = 0; i < val.length; i++) {
        var all = []
        var dbIns = this.struct[val[i].value]

        for (var key in dbIns) {
          all.push(key)
        }

        this.$set(this.selData, val[i].value, {
          groups: [{
            sels: all,
            name: ''
          }]
        })
      }

      this.selection.db = val[0].value
      this.selection.groups = [[]]
    },
    'selection.db' (val, oldVal) {
      this.selection.groups = []

      if (val !== '') {
        for (var key in this.selData[val].groups) {
          this.selection.groups.push({
            sels: [],
            name: this.selData[val].groups[key].name
          })
        }
      }
    }
  }
}
</script>

<style>
</style>
