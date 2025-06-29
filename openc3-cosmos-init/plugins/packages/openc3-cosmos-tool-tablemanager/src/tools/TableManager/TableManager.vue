<!--
# Copyright 2022 Ball Aerospace & Technologies Corp.
# All Rights Reserved.
#
# This program is free software; you can modify and/or redistribute it
# under the terms of the GNU Affero General Public License
# as published by the Free Software Foundation; version 3 with
# attribution addendums as found in the LICENSE.txt
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU Affero General Public License for more details.

# Modified by OpenC3, Inc.
# All changes Copyright 2023, OpenC3, Inc.
# All Rights Reserved
#
# This file may also be used under the terms of a commercial license
# if purchased from OpenC3, Inc.
-->

<template>
  <div>
    <top-bar :menus="menus" :title="title" />
    <v-snackbar v-model="displayReadOnlyMessage" top :timeout="-1" color="orange">
      <v-icon> mdi-pencil-off </v-icon>
      {{ lockedBy }} is editing this script. Editor is in read-only mode
      <template v-slot:action="{ attrs }">
        <v-btn
          text
          v-bind="attrs"
          color="danger"
          @click="confirmLocalUnlock"
          data-test="unlock-button"
        >
          Unlock
        </v-btn>
        <v-btn
          text
          v-bind="attrs"
          @click="
            () => {
              displayReadOnlyMessage = false
            }
          "
        >
          dismiss
        </v-btn>
      </template>
    </v-snackbar>
    <v-file-input
      show-size
      v-model="selectedFileForUpload"
      ref="fileInput"
      accept=".bin"
      data-test="file-input"
      style="position: fixed; top: -100%"
    />
    <v-card>
      <v-card-text>
        <v-row dense>
          <v-col cols="6">
            <v-text-field
              outlined
              dense
              readonly
              hide-details
              label="Filename"
              v-model="fullFilename"
              id="filename"
              class="filename"
              data-test="filename"
            />
          </v-col>
          <v-col cols="6">
            <v-text-field
              outlined
              dense
              readonly
              hide-details
              label="Definition"
              v-model="definitionFilename"
              id="definition-filename"
              class="filename"
              data-test="definition-filename"
            />
          </v-col>
        </v-row>
        <v-row dense>
          <v-col cols="auto" class="mr-auto">
            <span class="text-body-1 ma-2">File Download:</span>
            <v-btn
              dense
              color="primary"
              class="mr-3"
              @click="downloadBinary(null)"
              :disabled="filename == ''"
              data-test="download-file-binary"
            >
              Binary
              <v-icon right dark> mdi-file-code </v-icon>
            </v-btn>
            <v-btn
              dense
              color="primary"
              class="mr-3"
              @click="downloadDefinition(null)"
              :disabled="filename == ''"
              data-test="download-file-definition"
            >
              Definition
              <v-icon right dark> mdi-file-document-edit </v-icon>
            </v-btn>
            <v-btn
              dense
              color="primary"
              @click="downloadReport(null)"
              :disabled="filename == ''"
              data-test="download-file-report"
            >
              Report
              <v-icon right dark> mdi-file-document </v-icon>
            </v-btn>
          </v-col>
          <v-col cols="auto">
            <v-btn
              dense
              color="primary"
              @click="upload()"
              :disabled="!uploadScript"
              data-test="upload-file"
            >
              Upload
              <v-icon right dark> mdi-file-upload </v-icon>
            </v-btn>
          </v-col>
          <v-col cols="auto">
            <v-btn
              dense
              color="primary"
              @click="download()"
              :disabled="!downloadScript"
              data-test="download-file"
            >
              Download
              <v-icon right dark> mdi-file-download </v-icon>
            </v-btn>
          </v-col>
          <v-col cols="auto">
            <v-tooltip bottom>
              <template v-slot:activator="{ on, attrs }">
                <span v-bind="attrs" v-on="on">
                  <v-checkbox
                    v-model="scriptBackground"
                    label="B/G"
                    class="shrink mt-0"
                    data-test="upload-background"
                  />
                </span>
              </template>
              <span>Run upload download scripts in the background</span>
            </v-tooltip>
          </v-col>
        </v-row>
      </v-card-text>
      <v-card-title style="padding-top: 0px">
        Items
        <v-spacer />
        <v-text-field
          v-model="search"
          label="Search"
          prepend-inner-icon="mdi-magnify"
          clearable
          outlined
          dense
          single-line
          hide-details
          class="search"
        />
      </v-card-title>
      <v-tabs v-model="curTab">
        <v-tab v-for="(table, index) in tables" :key="index">
          {{ table.name }}
        </v-tab>
      </v-tabs>
      <v-tabs-items v-model="curTab">
        <v-tab-item
          v-for="(table, index) in tables"
          :key="`${filename}${index}`"
        >
          <v-data-table
            :headers="table.headers"
            :items="table.rows"
            :search="search"
            :items-per-page="20"
            :footer-props="{
              itemsPerPageOptions: [10, 20, 50, 100, -1],
            }"
            calculate-widths
            multi-sort
            dense
            :data-test="table.name"
          >
            <template v-slot:item="{ item }">
              <table-row
                :items="item"
                :key="item[0].name"
                @change="onChange(item, $event)"
              />
            </template>
            <template v-slot:footer v-if="tables.length > 1">
              <div style="position: absolute" class="ma-3">
                <span class="text-body-1 mr-3">Table Download:</span>
                <v-btn
                  dense
                  color="primary"
                  class="mr-3"
                  @click="downloadBinary(table.name)"
                  :disabled="filename == ''"
                  data-test="download-table-binary"
                >
                  Binary
                  <v-icon right dark> mdi-file-code </v-icon>
                </v-btn>
                <v-btn
                  dense
                  color="primary"
                  class="mr-3"
                  @click="downloadDefinition(table.name)"
                  :disabled="filename == ''"
                  data-test="download-table-definition"
                >
                  Definition
                  <v-icon right dark> mdi-file-document-edit </v-icon>
                </v-btn>
                <v-btn
                  dense
                  color="primary"
                  @click="downloadReport(table.name)"
                  :disabled="filename == ''"
                  data-test="download-table-report"
                >
                  Report
                  <v-icon right dark> mdi-file-document </v-icon>
                </v-btn>
              </div>
            </template>
          </v-data-table>
        </v-tab-item>
      </v-tabs-items>
    </v-card>
    <file-open-save-dialog
      v-if="fileOpen"
      v-model="fileOpen"
      type="open"
      api-url="/openc3-api/tables"
      @file="setFile($event)"
      @error="setError($event)"
    />
    <file-open-save-dialog
      v-if="showSaveAs"
      v-model="showSaveAs"
      type="save"
      require-target-parent-dir
      api-url="/openc3-api/tables"
      :input-filename="filename"
      @filename="saveAsFilename($event)"
      @error="setError($event)"
    />
    <simple-text-dialog
      v-model="showError"
      :title="errorTitle"
      :text="errorText"
    />
  </div>
</template>

<script>
import Api from '@openc3/tool-common/src/services/api'
import { OpenC3Api } from '@openc3/tool-common/src/services/openc3-api'
import TopBar from '@openc3/tool-common/src/components/TopBar'
import TableRow from '@/tools/TableManager/TableRow'
import FileOpenSaveDialog from '@openc3/tool-common/src/components/FileOpenSaveDialog'
import SimpleTextDialog from '@openc3/tool-common/src/components/SimpleTextDialog'

export default {
  components: {
    TopBar,
    TableRow,
    FileOpenSaveDialog,
    SimpleTextDialog,
  },
  data() {
    return {
      title: 'Table Manager',
      search: '',
      curTab: null,
      tables: [],
      api: null,
      definition: null,
      fileInput: '',
      definitionFilename: '',
      fileNew: false,
      filename: '',
      fileModified: '',
      lockedBy: null,
      fileOpen: false,
      showSave: false,
      showSaveAs: false,
      showError: false,
      errorTitle: '',
      errorText: '',
      uploadScript: null,
      downloadScript: null,
      scriptBackground: true,
    }
  },
  computed: {
    readOnly: function () {
      return !!this.lockedBy
    },
    fullFilename() {
      return `${this.filename} ${this.fileModified}`.trim()
    },
    menus: function () {
      return [
        {
          label: 'File',
          items: [
            {
              label: 'New File',
              icon: 'mdi-file-plus',
              command: () => {
                this.newFile()
              },
            },
            {
              label: 'Open File',
              icon: 'mdi-folder-open',
              command: () => {
                this.openFile()
              },
            },
            {
              divider: true,
            },
            {
              label: 'Save File',
              icon: 'mdi-content-save',
              command: () => {
                this.saveFile()
              },
            },
            {
              label: 'Save As...',
              icon: 'mdi-content-save',
              command: () => {
                this.saveAs()
              },
            },
            {
              divider: true,
            },
            // {
            //   label: 'Load Binary',
            //   icon: 'mdi-cloud-upload',
            //   command: () => {
            //     this.loadBinary()
            //   },
            // },
            // {
            //   divider: true,
            // },
            {
              label: 'Delete File',
              icon: 'mdi-delete',
              command: () => {
                this.delete()
              },
            },
          ],
        },
      ]
    },
  },
  watch: {
    // Every time the filename changes we figure out if there is an associated upload & download script
    filename: function (val) {
      let upload =
        this.filename.split('/').slice(0, 2).join('/') + '/procedures/upload'
      let download =
        this.filename.split('/').slice(0, 2).join('/') + '/procedures/download'
      // First try Ruby
      Api.get(`/openc3-api/tables/${upload}.rb`, {
        headers: {
          Accept: 'application/json',
          // Since we're just checking for existence, 404 is possible so ignore it
          'Ignore-Errors': '404',
        },
      })
        .then((response) => {
          this.uploadScript = `${upload}.rb`
        })
        .catch((error) => {
          // Now try python
          Api.get(`/openc3-api/tables/${upload}.py`, {
            headers: {
              Accept: 'application/json',
              // Since we're just checking for existence, 404 is possible so ignore it
              'Ignore-Errors': '404',
            },
          })
            .then((response) => {
              this.uploadScript = `${upload}.py`
            })
            .catch((error) => {
              this.uploadScript = null
            })
        })
      // First check Ruby
      Api.get(`/openc3-api/tables/${download}.rb`, {
        headers: {
          Accept: 'application/json',
          // Since we're just checking for existence, 404 is possible so ignore it
          'Ignore-Errors': '404',
        },
      })
        .then((response) => {
          this.downloadScript = `${download}.rb`
        })
        .catch((error) => {
          // Now try python
          Api.get(`/openc3-api/tables/${download}.py`, {
            headers: {
              Accept: 'application/json',
              // Since we're just checking for existence, 404 is possible so ignore it
              'Ignore-Errors': '404',
            },
          })
            .then((response) => {
              this.downloadScript = `${download}.py`
            })
            .catch((error) => {
              this.downloadScript = null
            })
        })
    },
  },
  created() {
    // Ensure Offline Access Is Setup For the Current User
    this.api = new OpenC3Api()
    this.api.ensure_offline_access()
  },
  methods: {
    // File menu actions
    newFile: function () {
      this.fileModified = ''
      this.fileNew = true
      this.fileOpen = true
    },
    openFile: function () {
      this.fileOpen = true
    },
    // Called by the FileOpenDialog to set the file contents
    setFile: function ({ file, locked }) {
      // They opened a definition file so create a new binary
      if (file.name.includes('.txt')) {
        if (this.fileNew) {
          this.buildNewBinary(file.name)
          this.fileNew = false
        } else {
          this.getDefinition(file.name)
        }
      } else {
        this.unlockFile() // first unlock what was just being edited
        // Split off the ' *' which indicates a file is modified on the server
        this.filename = file.name.split('*')[0]
        this.fileModified = ''
        this.lockedBy = locked
        this.getDefinition()
      }
    },
    // Called by the FileOpenSaveDialog on error
    setError(event) {
      this.errorTitle = 'Error'
      this.errorText = `Error: ${event}`
      this.errorText = response.data.message
      this.showError = true
    },
    saveFile: function () {
      // Save a file by posting the new contents
      this.showSave = true

      const formData = new FormData()
      formData.append('binary', this.filename)
      formData.append('definition', this.definitionFilename)
      formData.append('tables', JSON.stringify(this.tables))
      Api.post(`/openc3-api/tables/${this.filename}`, {
        data: formData,
        headers: { 'Content-Type': 'multipart/form-data' },
      })
        .then((response) => {
          this.fileModified = ''
          setTimeout(() => {
            this.showSave = false
          }, 2000)
        })
        .catch(({ response }) => {
          this.showSave = false
          this.errorTitle = 'Save Error'
          this.errorText = response.data.message
          this.showError = true
        })
      this.lockFile() // Ensure this file is locked for editing
    },
    saveAs: function () {
      this.showSaveAs = true
    },
    saveAsFilename: function (filename) {
      Api.put(`/openc3-api/tables/${this.filename}/save-as/${filename}`).then(
        (response) => {
          this.filename = filename
          this.getDefinition(this.definitionFilename)
        },
      )
    },
    delete: function () {
      if (this.filename !== '') {
        this.$dialog
          .confirm(`Permanently delete file: ${this.filename}`, {
            okText: 'Delete',
            cancelText: 'Cancel',
          })
          .then((dialog) => {
            return Api.delete(`/openc3-api/tables/${this.filename}`, {
              data: {},
            })
          })
          .then((response) => {
            this.tables = []
            this.filename = ''
            this.definitionFilename = ''
          })
          .catch((error) => {
            // TODO: It returns true on cancel?
          })
      }
    },
    downloadBinary: function (tableName = null) {
      const formData = new FormData()
      formData.append('binary', this.filename)
      formData.append('definition', this.definitionFilename)
      if (tableName !== null) {
        formData.append('table', tableName)
      }
      Api.post(`/openc3-api/tables/binary`, {
        data: formData,
        headers: { 'Content-Type': 'multipart/form-data' },
      }).then((response) => {
        // Decode Base64 string
        const decodedData = window.atob(response.data.contents)
        // Create UNIT8ARRAY of size same as row data length
        const uInt8Array = new Uint8Array(decodedData.length)
        // Insert all character code into uInt8Array
        for (let i = 0; i < decodedData.length; ++i) {
          uInt8Array[i] = decodedData.charCodeAt(i)
        }
        // Return BLOB image after conversion
        const blob = new Blob([uInt8Array], {
          type: 'application/octet-stream',
        })
        // Make a link and then 'click' on it to start the download
        const link = document.createElement('a')
        link.href = URL.createObjectURL(blob)
        link.setAttribute('download', response.data.filename)
        link.click()
      })
    },
    downloadDefinition: function (tableName = null) {
      const formData = new FormData()
      formData.append('definition', this.definitionFilename)
      if (tableName !== null) {
        formData.append('table', tableName)
      }
      Api.post(`/openc3-api/tables/definition`, {
        data: formData,
        headers: { 'Content-Type': 'multipart/form-data' },
      }).then((response) => {
        const blob = new Blob([response.data.contents], {
          type: 'text/plain',
        })
        // Make a link and then 'click' on it to start the download
        const link = document.createElement('a')
        link.href = URL.createObjectURL(blob)
        link.setAttribute('download', response.data.filename)
        link.click()
      })
    },
    downloadReport: function (tableName = null) {
      const formData = new FormData()
      formData.append('binary', this.filename)
      formData.append('definition', this.definitionFilename)
      if (tableName !== null) {
        formData.append('table', tableName)
      }
      Api.post(`/openc3-api/tables/report`, {
        data: formData,
        headers: { 'Content-Type': 'multipart/form-data' },
      }).then((response) => {
        const header =
          `Binary: ${this.filename}\n` +
          `Definition: ${this.definitionFilename}\n\n`
        const blob = new Blob([header, response.data.contents], {
          type: 'text/plain',
        })
        // Make a link and then 'click' on it to start the download
        const link = document.createElement('a')
        link.href = URL.createObjectURL(blob)
        link.setAttribute('download', response.data.filename)
        link.click()
      })
    },
    upload() {
      Api.post(`/script-api/scripts/${this.uploadScript}/run`, {
        data: {
          environment: [{ key: 'TBL_FILENAME', value: this.filename }],
        },
      }).then((response) => {
        if (this.scriptBackground !== true) {
          window.open(`/tools/scriptrunner/${response.data}`, '_blank')
        }
      })
    },
    download() {
      this.$dialog
        .confirm(
          `Are you sure you want to overwrite ${this.filename}? ` +
            'You can Save As to create a new file and then Download to preserve the existing file. ' +
            'Note: Once the download completes you will need to re-open the file to see changes.',
          {
            okText: 'Download (Overwrite!)',
            cancelText: 'Cancel',
          },
        )
        .then(() => {
          Api.post(`/script-api/scripts/${this.downloadScript}/run`, {
            data: {
              environment: [{ key: 'TBL_FILENAME', value: this.filename }],
            },
          }).then((response) => {
            if (this.scriptBackground !== true) {
              window.open(`/tools/scriptrunner/${response.data}`, '_blank')
            }
          })
        })
        .catch((error) => {}) // Cancel, do nothing
    },
    // TODO: Need to load to tmp dir or something before we can saveAs to rename
    // async loadBinary() {
    //   this.fileInput = ''
    //   this.$refs.fileInput.$refs.input.click()
    //   // Wait for the file to be set by the dialog so upload works
    //   while (this.fileInput === '') {
    //     await new Promise((resolve) => setTimeout(resolve, 500))
    //   }
    //   this.filename = this.fileInput.name
    //   this.saveAs()
    // },
    confirmLocalUnlock: function () {
      this.$dialog
        .confirm(
          'Are you sure you want to unlock this file for editing? ' +
            'If another user is editing this file, your changes might conflict with each other.',
          {
            okText: 'Force Unlock',
            cancelText: 'Cancel',
          },
        )
        .then(() => {
          this.lockedBy = null
          return this.lockFile() // Re-lock it as this user so it's locked for anyone else who opens it
        })
    },
    lockFile: function () {
      return Api.post(`/openc3-api/tables/${this.filename}/lock`)
    },
    unlockFile: function () {
      if (this.filename !== '' && !this.readOnly) {
        Api.post(`/openc3-api/tables/${this.filename}/unlock`)
      }
    },
    getDefinition: function (definitionFilename = null) {
      if (!definitionFilename) {
        // Create a notional definition filename based on the binary
        definitionFilename = this.filename
          .replace('/bin/', '/config/')
          .replace('.bin', '_def.txt')
      }
      this.tables = [] // Clear so table is re-rendered
      const formData = new FormData()
      formData.append('binary', this.filename)
      formData.append('definition', definitionFilename)
      Api.post(`/openc3-api/tables/load`, {
        data: formData,
        headers: { 'Content-Type': 'multipart/form-data' },
      })
        .then((response) => {
          // Set the definition as actually loaded. The backend
          // checks what we sent and does a lookup to return
          // something close if it can't find an exact match.
          this.definitionFilename = response.data.definition
          this.tables = response.data.tables.map((table) => {
            return {
              ...table,
              // Build up the headers for proper searching
              headers: table.headers.map((text, i) => {
                const header = {
                  text,
                  filterable: text !== 'INDEX',
                }
                if (table.numColumns === 1) {
                  // In the 1D table the searchable value is the first value in the row
                  // Note the names in 1D are INDEX, NAME, VALUE
                  return {
                    ...header,
                    value: `[0].${text.toLowerCase()}`,
                  }
                } else {
                  // In the 2D table the searchable value is always in the value attribute
                  // of the current column item
                  return {
                    ...header,
                    value: `[${i}].value`,
                  }
                }
              }),
            }
          })

          if (response.data['errors']) {
            this.$notify.caution({
              title: 'Warning',
              body: response.data['errors'],
            })
          }
        })
        .catch((error) => {
          if (error.response.status == 404) {
            this.$notify.normal({
              title: 'Definition File Not Found',
              body: `Definition file ${definitionFilename} not found. Please select definition file.`,
            })
            this.fileOpen = true
          } else {
            this.$notify.serious({
              title: 'Error',
              body: `Error loading due to ${error.response.statusText}. Status: ${error.response.status}`,
            })
          }
        })
    },
    buildNewBinary: function (filename) {
      const formData = new FormData()
      formData.append('definition', filename)
      Api.post(`/openc3-api/tables/generate`, {
        data: formData,
        headers: { 'Content-Type': 'multipart/form-data' },
      }).then((response) => {
        this.filename = response.data.filename
        this.getDefinition(filename)
      })
    },
    onChange: function (item, { index, event }) {
      this.fileModified = '*'
      item[index].value = event
    },
  },
}
</script>
<style scoped>
.filename {
  background-color: var(--color-background-base-default);
}
</style>
