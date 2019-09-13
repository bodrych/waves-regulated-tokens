<template>
  <el-container v-cloak>
    <el-header>
      <span class="title">Regulated tokens</span>
    </el-header>
    <el-main>
      <el-row type="flex" style="flex-basis: 100%; margin-left: 0" :gutter="50">
        <el-col type="flex" :span="6" style="border-right: 1px solid #EBEEF5">
          <span class="subtitle">Issue token</span>
          <el-form ref="issueForm" :model="asset" :rules="issueFormRules" label-position="top">
            <el-form-item label="Name" prop="name">
              <el-input v-model="asset.name" clearable></el-input>
            </el-form-item>
            <el-form-item label="Description" prop="description">
              <el-input v-model="asset.description" clearable></el-input>
            </el-form-item>
            <el-form-item label="Quantity" prop="quantity">
              <el-input-number controls-position="right" :min="1" :step="1000" v-model="asset.quantity"></el-input-number>
            </el-form-item>
            <el-form-item label="Precision">
              <el-input-number controls-position="right" :min="0" :max="8" v-model="asset.precision"></el-input-number>
            </el-form-item>
            <el-form-item>
              <el-checkbox v-model="asset.reissuable" name="type">Reissuable</el-checkbox>
            </el-form-item>
            <el-form-item>
              <el-tooltip content="Issue new regulated smart asset" placement="top" effect="light">
                <el-button
                @click="submitForm('issueForm', issue)"
                :disabled="disabled"
                :loading="asset.loading"
                icon="el-icon-finished"
                >Issue regulated token</el-button>
              </el-tooltip>
              <el-button
              @click="resetForm('issueForm')"
              icon="el-icon-close">Reset form</el-button>
            </el-form-item>
          </el-form>
        </el-col>

        <el-col type="flex" :span="18">
          <span class="subtitle">Manage whitelist</span>
          <el-row type="flex" :gutter="20">
            <el-col type="flex">
              <el-row type="flex">
                <el-col type="flex">
                  <el-form ref="loadForm" :model="loadForm" label-position="top" :rules="loadFormRules">
                    <el-form-item label="Asset ID" prop="asset">
                      <el-input v-model="loadForm.asset" clearable></el-input>
                    </el-form-item>
                    <el-form-item>
                      <el-input v-model="currentAsset" readonly>
                        <template slot="prepend">Current asset</template>
                      </el-input>
                    </el-form-item>
                    <el-form-item>
                      <el-tooltip content="Load new asset whitelist" placement="top" effect="light">
                        <el-button
                        @click="submitForm('loadForm', load)"
                        :disabled="disabled || updateDisabled"
                        :loading="loadForm.loading"
                         icon="el-icon-download"
                        >Load whitelist</el-button>
                      </el-tooltip>
                      <el-tooltip content="Load current asset whitelist" placement="top" effect="light">
                        <el-button
                        @click="update"
                        :disabled="disabled || updateDisabled"
                        :loading="loadForm.loading1"
                         icon="el-icon-refresh"
                        >Update</el-button>
                      </el-tooltip>
                      <el-button
                      @click="resetForm('loadForm')"
                      icon="el-icon-close">Reset form</el-button>
                    </el-form-item>
                  </el-form>
                </el-col>
              </el-row>
            </el-col>

            <el-col type="flex">
              <el-form ref="newAddressForm" :model="newAddress" :rules="newAddressRules" label-position="top">
                <el-form-item label="Address" prop="address">
                  <el-input v-model="newAddress.address" clearable></el-input>
                </el-form-item>
                <el-form-item>
                  <el-checkbox v-model="newAddress.status" name="type">Whitelisted</el-checkbox>
                </el-form-item>
                <el-form-item>
                  <el-tooltip content="Add address to current asset whitelist" placement="top" effect="light">
                    <el-button
                    @click="submitForm('newAddressForm', addNewAddress)"
                    :disabled="disabled || newAddress === ''"
                    :loading="newAddress.loading"
                    icon="el-icon-plus"
                    >Add to whitelist</el-button>
                  </el-tooltip>
                  <el-button
                  @click="resetForm('newAddressForm')"
                  icon="el-icon-close">Reset form</el-button>
                </el-form-item>
              </el-form>
            </el-col>
          </el-row>
          <el-row type="flex" justify="center">
            <el-col type="flex">
              <el-table
                :data="addresses"
                height="50vh"
                v-loading="loadForm.loading || loadForm.loading1">
                <el-table-column
                  label="Address"
                  width="500"
                  align="left">
                  <template slot-scope="scope">
                    <span style="margin-left: 10px">{{ scope.row.address }}</span>
                  </template>
                </el-table-column>
                <el-table-column
                  label="Status"
                  align="center">
                  <template slot-scope="scope">
                    <el-checkbox v-model="scope.row.newStatus"></el-checkbox>
                  </template>
                </el-table-column>
                <el-table-column
                  label="Operations"
                  align="right">
                  <template slot-scope="scope">
                    <el-tooltip content="Save address status" placement="top" effect="light">
                      <el-button
                        size="mini"
                        @click="editStatus(scope.$index, scope.row.address, scope.row.newStatus)"
                        :disabled="scope.row.disabled || disabled || scope.row.status === scope.row.newStatus"
                        :loading="scope.row.loading"
                        >Save</el-button>
                      </el-tooltip>
                  </template>
                </el-table-column>
              </el-table>
            </el-col>
          </el-row>
        </el-col>
      </el-row>
    </el-main>
    <el-footer>
      <el-link href="https://github.com/bodrych/waves-regulated-tokens" target="_blank">Github</el-link>
    </el-footer>
  </el-container>
</template>

<script>
  import axios from 'axios'
  import { waitForTx } from '@waves/waves-transactions'

  export default {
    name: 'App',
    data: () => {
      return {
        dApp: '3PQEoRrRrrrrN5DVCerPBM4Tzz5qnEWzbo7',
        networkCode: 'W',
        assetId: '',
        publicState: {
          network: {
            code: 'W',
            server: 'https://nodes.wavesnodes.com/'
          }
        },
        disabled: true,
        updateDisabled: false,
        addresses: [],
        newAddress: {
          address: '',
          status: false,
          loading: false
        },
        asset: {
          id: '',
          name: '',
          description: '',
          quantity: 100000000,
          precision: 8,
          reissuable: false,
          script: 'base64:AwQAAAAEZEFwcAkBAAAABXZhbHVlAAAAAQkBAAAAEWFkZHJlc3NGcm9tU3RyaW5nAAAAAQIAAAAjM1BRRW9SclJycnJyTjVEVkNlclBCTTRUeno1cW5FV3pibzcEAAAAByRtYXRjaDAFAAAAAnR4AwkAAAEAAAACBQAAAAckbWF0Y2gwAgAAABNUcmFuc2ZlclRyYW5zYWN0aW9uBAAAAAF0BQAAAAckbWF0Y2gwBAAAAA5zZW5kZXJBY2NlcHRlZAQAAAAHJG1hdGNoMQkABBsAAAACBQAAAARkQXBwCQABLAAAAAIJAAEsAAAAAgkAAlgAAAABCAUAAAAEdGhpcwAAAAJpZAIAAAABXwkAAlgAAAABCAgFAAAAAXQAAAAGc2VuZGVyAAAABWJ5dGVzAwkAAAEAAAACBQAAAAckbWF0Y2gxAgAAAAdCb29sZWFuBAAAAAFzBQAAAAckbWF0Y2gxBQAAAAFzBwQAAAARcmVjaXBpZW50QWNjZXB0ZWQEAAAAByRtYXRjaDEJAAQbAAAAAgUAAAAEZEFwcAkAASwAAAACCQABLAAAAAIJAAJYAAAAAQgFAAAABHRoaXMAAAACaWQCAAAAAV8JAAJYAAAAAQgJAAQkAAAAAQgFAAAAAXQAAAAJcmVjaXBpZW50AAAABWJ5dGVzAwkAAAEAAAACBQAAAAckbWF0Y2gxAgAAAAdCb29sZWFuBAAAAAFzBQAAAAckbWF0Y2gxBQAAAAFzBwMDAwUAAAAOc2VuZGVyQWNjZXB0ZWQFAAAAEXJlY2lwaWVudEFjY2VwdGVkBwYDCQAAAAAAAAIIBQAAAAF0AAAABnNlbmRlcggFAAAABHRoaXMAAAAGaXNzdWVyBQAAABFyZWNpcGllbnRBY2NlcHRlZAcGAwUAAAAOc2VuZGVyQWNjZXB0ZWQJAAAAAAAAAggFAAAAAXQAAAAJcmVjaXBpZW50CAUAAAAEdGhpcwAAAAZpc3N1ZXIHAwkAAAEAAAACBQAAAAckbWF0Y2gwAgAAABNFeGNoYW5nZVRyYW5zYWN0aW9uBAAAAAFlBQAAAAckbWF0Y2gwBAAAAA5zZW5kZXJBY2NlcHRlZAQAAAAHJG1hdGNoMQkABBsAAAACBQAAAARkQXBwCQABLAAAAAIJAAEsAAAAAgkAAlgAAAABCAUAAAAEdGhpcwAAAAJpZAIAAAABXwkAAlgAAAABCAgIBQAAAAFlAAAACXNlbGxPcmRlcgAAAAZzZW5kZXIAAAAFYnl0ZXMDCQAAAQAAAAIFAAAAByRtYXRjaDECAAAAB0Jvb2xlYW4EAAAAAXMFAAAAByRtYXRjaDEFAAAAAXMHAwUAAAAOc2VuZGVyQWNjZXB0ZWQGCQAAAAAAAAIICAUAAAABZQAAAAlzZWxsT3JkZXIAAAAGc2VuZGVyCAUAAAAEdGhpcwAAAAZpc3N1ZXIHmYjjRg==',
          loading: false
        },
        loadForm : {
          asset: '',
          loading: false,
          loading1: false
        },
        page: 'Manage',
        pages: [
          'Manage',
          'Issue'
        ],
        issueFormRules: {
          name: [
            { required: true, message: 'Please input asset name', trigger: 'blur' }
          ]
        },
        loadFormRules: {
          asset: [
            { required: true, message: 'Please input asset ID', trigger: 'blur' }
          ]
        },
        newAddressRules: {
          address: [
            { required: true, message: 'Please input address', trigger: 'blur' }
          ]
        },
        currentAsset: ''
      }
    },
    created: function () {
      window.addEventListener('load', this.checkKeeper)
    },
    mounted: function () {
      if (localStorage.currentAsset) {
        this.currentAsset = localStorage.currentAsset
      }
    },
    watch: {
      currentAsset (value) {
        localStorage.currentAsset = value
      }
    },
    methods: {
      submitForm(formName, method, params) {
        this.$refs[formName].validate((valid) => {
          if (valid) {
            method(params)
          } else {
            // this.setStatus('error', 'Invalid data')
            return false
          }
        });
      },
      resetForm: function (formName) {
        this.$refs[formName].resetFields()
      },
      setStatus: function (type, message, duration) {
        // this.status = text
        this.$message({
          showClose: true,
          dangerouslyUseHTMLString: true,
          customClass: 'message',
          type,
          message,
          duration: duration || 3000
        })
      },
      listenKeeper: function (data) {
        this.disabled = false
        if (!data.initialized) {
          this.setStatus('warning', 'Create waves account')
          return null
        }
        if (data.locked) {
          this.setStatus('warning', 'Unlock keeper')
          return null
        }
        this.checkNetwork(data)
        this.publicState = data
      },
      parseKeeperErrors: function (error) {
        let errorText = error.message || 'Error';
        this.setStatus('error', errorText);
      },
      checkKeeper: function () {
        if (!window.WavesKeeper) {
          this.setStatus('warning', 'Install <a href="https://wavesplatform.com/products-keeper">Waves Keeper</a> and restart page')
          return null
        }

        window.WavesKeeper.initialPromise.then(() => {
          window.WavesKeeper.on('update', this.listenKeeper)
          return window.WavesKeeper.publicState()
        }).then(
        this.listenKeeper,
        this.parseKeeperErrors
        )
      },
      checkNetwork: function (data) {
        if (data.network.code !== this.networkCode) {
          this.setStatus('warning', 'Change network')
          this.disabled = true
          return null
        }
        this.disabled = false
      },
      load: async function () {
        this.loadWhitelist(this.loadForm.asset)
        this.loadForm.loading = true
      },
      update: async function () {
        if (this.currentAsset !== '') {
          this.loadWhitelist(this.currentAsset)
          this.loadForm.loading1 = true
        } else {
          this.setStatus('error', 'Invalid current asset')
        }
      },
      loadWhitelist: async function (asset) {
        this.currentAsset = asset
        let regex = asset + '.*'
        let data = await this.getData(regex)
        let addresses = data.map(item => {
          return {
            address: item.key.split('_')[1],
            status: item.value,
            newStatus: item.value,
            disabled: false,
            loading: false
          }
        })
        this.addresses = addresses.splice(0)
        this.loadForm.loading = false
        this.loadForm.loading1 = false
      },
      findByKey: function (array, addr) {
        let index = false
        for (let i in array) {
          if (array[i].address == addr) {
            index = i
          }
        }
        return index
      },
      getData: async function (regex) {
        try {
          const response = await axios.get(`${this.publicState.network.server}addresses/data/${this.dApp}?matches=${regex}`)
          return response.data
        } catch (e) {
          throw e
        }
      },
      addNewAddress: async function () {
        if (this.currentAsset !== '') {
          if (!this.findByKey(this.addresses, this.newAddress.address)) {
            this.newAddress.loading = true
            this.updateDisabled = true
            let tx = await this.add(this.newAddress.address, this.newAddress.status)
            try {
              await waitForTx(tx.id, { apiBase: this.publicState.network.server, timeout: 200000 })
              this.addresses.push({
                address: this.newAddress.address,
                status: this.newAddress.status,
                newStatus: this.newAddress.status,
                disabled: false
              })
              this.newAddress.address = ''
              this.updateDisabled = false
              this.newAddress.loading = false
            } catch (e) {
              this.updateDisabled = false
              this.newAddress.loading = false
              this.setStatus('error', e.message)
            }
          } else {
            this.setStatus('message', 'Address already added')
          }
        } else {
          this.setStatus('error', 'Invalid current asset')
        }
      },
      editStatus: async function (index, address, status) {
        if (this.currentAsset !== '') {
          this.updateDisabled = true
          this.addresses[index].disabled = true
          this.addresses[index].loading = true
          let tx = await this.add(address, status)
          try {

            await waitForTx(tx.id, { apiBase: this.publicState.network.server, timeout: 200000 })
            this.addresses[index].status = this.addresses[index].newStatus
            this.updateDisabled = false
            this.addresses[index].disabled = false
          this.addresses[index].loading = false
          } catch (e) {
            this.updateDisabled = false
            this.addresses[index].disabled = false
          this.addresses[index].loading = false
            this.setStatus('error', e.message)
          }
        } else {
          this.setStatus('error', 'Invalid current asset')
        }
      },
      add: async function (address, status) {
        const params = {
          type: 16,
          data: {
            fee: {
              assetId: 'WAVES',
              tokens: '0.005'
            },
            dApp: this.dApp,
            call: {
              args: [
              { type: 'string', value: this.currentAsset },
              { type: 'string', value: address },
              { type: 'boolean', value: status }
              ],
              function: 'add'
            },
            payment: []
          }
        }
        try {
          const result = await window.WavesKeeper.signAndPublishTransaction(params)
          const data = JSON.parse(result)
          return data
        } catch (e) {
          this.setStatus('error', e.message)
        }
      },
      issue: async function () {
        const params = {
          type: 3,
          data: {
            fee: {
              assetId: 'WAVES',
              tokens: '1.0'
            },
            name: this.asset.name,
            description: this.asset.description,
            quantity: this.asset.quantity,
            precision: this.asset.precision,
            reissuable: this.asset.reissuable,
            script: this.asset.script,
          }
        }
        try {
          this.asset.loading = true
          const result = await window.WavesKeeper.signAndPublishTransaction(params)
          const data = JSON.parse(result)
          await waitForTx(data.id, { apiBase: this.publicState.network.server, timeout: 200000 })
          this.setStatus('success', 'Asset ID: ' + data.id, 10000)
          this.asset.id = data.id
          this.asset.loading = false
          return data
        } catch (e) {
          this.asset.loading = false
          this.setStatus('error', e.message)
        }
      }
    }
  }
</script>

<style>
  .el-header {
    color: #409EFF;
    display: flex;
    align-items: center;
    /*justify-content: center;*/
    border-bottom: 1px solid #EBEEF5;
  }
  .el-footer {
    display: flex;
    align-items: center;
    justify-content: center;
    border-top: 1px solid #EBEEF5;
  }
  .title {
    font-size: 1.25rem;
    color: #409EFF;
  }
  .subtitle {
    font-size: 1rem;
    color: #409EFF;
  }
  .el-main {
    display: flex !important;
  }
  .el-container {
    height: 100vh;
  }
  a {
    color: inherit;
  }
  [v-cloak] {
    display: none;
  }
</style>