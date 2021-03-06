<template>
  <div>
    <el-menu ref="connectionMenu" @open="openConnection" class="connection-menu" active-text-color="#ffd04b">
      <el-submenu v-for="(item, index) of connections" :key="item.showName" :index="getConnectionPoolKey(item) + '_' + index">

        <!-- connection item -->
        <template slot="title">
          <span slot="title" :title="item.showName" class="connection-name">{{item.showName}}</span>
          <i :title="$t('message.edit_connection')" class="el-icon-edit-outline" @click.stop.prevent="showEditConnection(item, getConnectionPoolKey(item))"></i>
          <i :title="$t('message.del_connection')" class="el-icon-delete" @click.stop.prevent="deleteConnection(item)"></i>
        </template>

        <el-form :inline="true" class="connection-form" size="mini">
          <el-form-item>
            <el-row :gutter="6">
              <el-col :span="12">
                <!-- db index select -->
                <el-select class="db-select" v-model="selectedDbIndex[getConnectionPoolKey(item)]" placeholder="DB" size="mini" @change="changeDb(getConnectionPoolKey(item))">
                  <el-option
                    v-for="index in dbs[getConnectionPoolKey(item)]"
                    :key="index"
                    :label="'DB' + index"
                    :value="index">
                  </el-option>
                  <!-- <span slot="prefix" class="fa fa-sitemap" style="font-size: 80%"></span> -->
                </el-select>
              </el-col>

              <el-col :span="12">
                <!-- new key btn -->
                <el-button class="new-key-btn el-icon-plus" @click="showNewKeyDialog(getConnectionPoolKey(item))">{{ $t('message.add_new_key')}}</el-button>
              </el-col>
            </el-row>
          </el-form-item>

          <!-- search match -->
          <el-form-item class="search-item">
            <el-row>
              <el-col :span="24">
                <el-input class="search-input" v-model="searchMatch[getConnectionPoolKey(item)]" @keyup.enter.native="changeMatchMode(getConnectionPoolKey(item))" :placeholder="$t('message.enter_to_search')" size="mini">
                  <span slot="suffix" >
                    <i class="el-input__icon search-icon" :class="searchIcon"  @click="changeMatchMode(getConnectionPoolKey(item))"></i>

                    <el-tooltip effect="dark" :content="$t('message.exact_search')" placement="bottom">
                      <el-checkbox v-model="searchExact[getConnectionPoolKey(item)]"></el-checkbox>
                    </el-tooltip>
                  </span>
                </el-input>
              </el-col>
            </el-row>

          </el-form-item>
        </el-form>

        <!-- key list -->
        <ul class="key-list">
          <RightClickMenu :items="rightMenus" :clickValue="{key: key, menuIndex: getConnectionPoolKey(item)}" :key="key" v-for="key of keyList[getConnectionPoolKey(item)]">
            <li class="key-item" :title="key"  @click="clickKey(key, getConnectionPoolKey(item))">{{key}}</li>
          </RightClickMenu>
        </ul>

        <!-- page -->
        <div class="pagenation">
          <el-button ref="pagePreButton" type="text" @click="pagePre(getConnectionPoolKey(item))" :disabled="preButtonDisable(getConnectionPoolKey(item))" size="mini" icon="el-icon-arrow-left"></el-button>
          <input
            :value="getPageIndex(getConnectionPoolKey(item))"
            :ref="'pageIndexInput' + getConnectionPoolKey(item)"
            @click="$event.target.select()"
            @keyup.enter="jumpToPage(getConnectionPoolKey(item), $event.target.value)"
            class="page-jumper el-input__inner"
            type="number"
            min="1"
            step="1"
          >
          </input>
          <el-button ref="pageNextButton" type="text" @click="pageNext(getConnectionPoolKey(item))" :disabled="nextPageDisabled[getConnectionPoolKey(item)]" size="mini" icon="el-icon-arrow-right"></el-button>
        </div>

      </el-submenu>
    </el-menu>


    <!-- edit connection dialog -->
    <el-dialog :title="$t('message.edit_connection')" :visible.sync="editConnectionDialog">
      <el-form v-model="afterEditData" label-width="80px">
        <el-form-item label="Host">
          <el-input v-model="afterEditData.host" autocomplete="off" placeholder="127.0.0.1"></el-input>
        </el-form-item>

        <el-form-item label="Port">
          <el-input v-model="afterEditData.port" autocomplete="off" placeholder="6379"></el-input>
        </el-form-item>

        <el-form-item label="Auth">
          <el-input v-model="afterEditData.auth" autocomplete="off"></el-input>
        </el-form-item>

        <el-form-item label="Name">
          <el-input v-model="afterEditData.name" autocomplete="off"></el-input>
        </el-form-item>

        <el-form-item label="">
          <el-checkbox v-model="sshOptionsShow">SSH Tunnel</el-checkbox>
        </el-form-item>

        <el-form v-if="sshOptionsShow" v-show="sshOptionsShow" label-width="80px">
          <el-form-item label="Host">
            <el-input v-model="afterEditData.sshOptions.host" autocomplete="off" placeholder=""></el-input>
          </el-form-item>

          <el-form-item label="Port">
            <el-input v-model="afterEditData.sshOptions.port" autocomplete="off"></el-input>
          </el-form-item>

          <el-form-item label="Username">
            <el-input v-model="afterEditData.sshOptions.username" autocomplete="off" placeholder=""></el-input>
          </el-form-item>

          <el-form-item label="Password">
            <el-input v-model="afterEditData.sshOptions.password" autocomplete="off"></el-input>
          </el-form-item>
        </el-form>

      </el-form>

      <div slot="footer" class="dialog-footer">
        <el-button @click="editConnectionDialog = false">{{ $t('el.messagebox.cancel') }}</el-button>
        <el-button type="primary" @click="editConnection">{{ $t('el.messagebox.confirm') }}</el-button>
      </div>
    </el-dialog>


    <!-- new key dialog -->
    <el-dialog :title="$t('message.add_new_key')" :visible.sync="newKeyDialog" width="320px">
      <el-form>
        <el-form-item :label="$t('message.key_type')">
          <el-select size="mini" v-model="selectedNewKeyType">
              <el-option
                v-for="(type, showType) in newKeyTypes"
                :key="type"
                :label="showType"
                :value="type">
              </el-option>
            </el-select>
        </el-form-item>
      </el-form>

      <div slot="footer" class="dialog-footer">
        <el-button @click="newKeyDialog = false">{{ $t('el.messagebox.cancel') }}</el-button>
        <el-button type="primary" @click="addNewKey">{{ $t('el.messagebox.confirm') }}</el-button>
      </div>
    </el-dialog>

  </div>

</template>

<script>
import Vue from 'vue';
import storage from '../storage.js';
import redisClient from '../redisClient.js';
import RightClickMenu from '@/components/RightClickMenu';

export default {
  data() {
    return {
      rightMenus: [
        {
          name: this.$t('message.open'),
          click: (clickValue) => {
            console.log('from callback....', clickValue);
            this.clickKey(clickValue.key, clickValue.menuIndex);
          },
        },
        {
          name: this.$t('message.open_new_tab'),
          click: (clickValue) => {
            console.log('from callback....', clickValue);
            this.clickKey(clickValue.key, clickValue.menuIndex, true);
          },
        },
      ],
      // dbs: [...Array(16).keys()],
      dbs: {},
      connections: [],
      keysPageSize: 50,
      connectionPool: {},
      openedStatus: {},
      selectedDbIndex: {},
      searchMatch: {},
      searchIcon: 'el-icon-search',
      searchPageSize: 10000,
      searchExact: {},
      keyList: [],
      scanCursorList: {},
      pageIndex: {},
      nextPageDisabled: {},
      beforeEditData: {},
      afterEditData: {},
      editConnectionDialog: false,
      newKeyDialog: false,
      selectedNewKeyType: 'string',
      newKeyTypes: {
        String: 'string', Hash: 'hash', List: 'list', Set: 'set', Zset: 'zset',
      },
      sshOptionsShow: false,
    };
  },
  components: {RightClickMenu},
  created() {
    this.$bus.$on('refreshKeyList', () => {
      this.refreshKeyList();
    });
    this.$bus.$on('refreshConnections', () => {
      this.initConnections();
    });
  },
  methods: {
    openConnection(openedMenuKey) {
      // get connection config
      // openedMenuKey like 10.20.1.1336379_0
      const menuIndex = openedMenuKey.split('_')[0];
      const connectionIndex = openedMenuKey.split('_')[1];
      const connection = this.connections[connectionIndex];

      if (!connection) {
        alert('open error'); return;
      }

      this.getDbIndex(menuIndex);
      const client = this.setGlobalConnection(menuIndex, connection);

      // ssh tunnel promise client
      if (typeof client.then === 'function') {
        client.then((realClient) => {
            this.afterOpenConnection(realClient, menuIndex, connection);
        });
      }

      // normal client
      else {
        this.afterOpenConnection(client, menuIndex, connection);
      }
    },
    afterOpenConnection(client, menuIndex, connection) {
      // new connection, not ready
      if (!client.ready) {
        client.on('ready', () => {
          // open status tab
          if (!this.openedStatus[menuIndex]) {
            this.$bus.$emit('openStatus', connection.showName);
            this.openedStatus[menuIndex] = true;
          }

          this.initDatabaseSelect(menuIndex);
          this.refreshKeyList();
        });
      }

      // connection is ready
      else {
        this.refreshKeyList();
      }
    },
    initDatabaseSelect(menuIndex) {
      this.$util.get('client').configAsync('get', 'databases').then((reply) => {
        console.log(reply);

        if (reply[1]) {
          this.$set(this.dbs, menuIndex, [...Array(parseInt(reply[1])).keys()]);
        }
        else {
          this.$set(this.dbs, menuIndex, [...Array(16).keys()]);
        }
      });
    },
    setGlobalConnection(menuIndex, connection) {
      let client = this.connectionPool[menuIndex];

      if (!client || !client.ready) {
        // ssh tunnel
        if (connection.sshOptions) {
          let sshPromise = redisClient.createSSHConnection(connection.sshOptions, connection.host, connection.port, connection.auth, menuIndex);

          sshPromise.then((client) => {
            client.on('error', (err) => {
              alert(err);
            });

            this.$util.set('client', client);
            this.connectionPool[menuIndex] = client;
          });

          return sshPromise;
        }

        else {
          client = redisClient.createConnection(connection.host, connection.port, connection.auth, menuIndex);

          client.on('error', (err) => {
            alert(err);
          });

          this.connectionPool[menuIndex] = client;
        }
      }

      // set global client
      this.$util.set('client', client);

      return client;
    },
    deleteConnection(connection) {
      console.log(connection);

      this.$confirm(
        this.$t('message.confirm_to_delete_connection'),
        { type: 'warning' },
      ).then(() => {
        storage.deleteConnection(connection);
        this.initConnections();

        this.$message.success({
          message: this.$t('message.delete_success'),
          duration: 1000,
        });
      }).catch(() => {
        //
      });
    },
    showEditConnection(oldConnection, menuIndex) {
      this.$confirm(
        this.$t('message.close_to_edit_connection'),
        { type: 'warning' },
      ).then(() => {
        this.closeAllConnection(oldConnection, menuIndex);

        this.editConnectionDialog = true;
        this.beforeEditData = oldConnection;
        this.sshOptionsShow = oldConnection.sshOptions ? true : false;

        this.afterEditData = JSON.parse(JSON.stringify(oldConnection));
        !this.afterEditData.sshOptions && (this.afterEditData.sshOptions = {port: 22});
        delete this.afterEditData.showName;
      }).catch((_) => {});
    },
    editConnection() {
      console.log('edit connection', this.beforeEditData, this.afterEditData);

      let afterEditData = JSON.parse(JSON.stringify(this.afterEditData));

      if (!this.sshOptionsShow || !afterEditData.sshOptions.host) {
        delete afterEditData.sshOptions;
      }

      if (storage.editConnection(this.beforeEditData, afterEditData) !== false) {
        this.editConnectionDialog = false;
        this.initConnections();
        return;
      }

      this.$message.error({
        message: this.$t('message.modify_failed'),
        duration: 1000,
      });
    },
    initConnections() {
      const connections = storage.getConnections(true);

      for (const item of connections) {
        item.showName = item.name || `${item.host}@${item.port}`;
      }

      this.connections = connections;
    },
    closeAllConnection() {
      console.log('closing all...');

      const connections = storage.getConnections(true);

      for (const i in connections) {
        // get menu index
        const index = `${this.getConnectionPoolKey(connections[i])}_${i}`;
        this.$refs.connectionMenu.close(index);
      }

      this.connectionPool = {};
      this.openedStatus = {};
      this.$bus.$emit('removeAllTab');
    },
    getConnectionPoolKey(connection) {
      return connection.host + connection.port + connection.name;
    },
    changeDb(menuIndex) {
      this.resetDb(menuIndex);

      const dbIndex = this.getDbIndex(menuIndex);

      this.setGlobalConnection(menuIndex);

      this.$util.get('client').selectAsync(dbIndex).then(() => {
        this.refreshKeyList();
      });
    },
    getDbIndex(menuIndex) {
      if (this.selectedDbIndex[menuIndex] === undefined) {
        this.selectedDbIndex[menuIndex] = 0;
      }

      return this.selectedDbIndex[menuIndex];
    },
    resetDb(menuIndex) {
      this.$set(this.pageIndex, menuIndex, 1);
      this.scanCursorList[menuIndex] = [0];
    },
    clickKey(key, menuIndex, newTab = false) {
      console.log(`clicked key ${key}`, `dbIndex ${this.selectedDbIndex}`);

      this.setGlobalConnection(menuIndex);
      this.$bus.$emit('clickedKey', key, newTab);
    },
    pagePre(menuIndex) {
      let pageIndex = this.getPageIndex(menuIndex);

      if (--pageIndex < 1) {
        pageIndex = 1;
      }

      this.$set(this.pageIndex, menuIndex, pageIndex);
      this.setGlobalConnection(menuIndex);

      this.refreshKeyList(false);
    },
    pageNext(menuIndex) {
      let pageIndex = this.getPageIndex(menuIndex);
      const cursorListLength = this.scanCursorList[menuIndex].length;

      this.$set(this.pageIndex, menuIndex, ++pageIndex);
      this.setGlobalConnection(menuIndex);

      this.refreshKeyList(pageIndex >= cursorListLength);
    },
    pageNextRecursive(menuIndex, targetPage) {
      let pageIndex = this.getPageIndex(menuIndex);
      const cursorList = this.scanCursorList[menuIndex];
      const cursorListLength = cursorList.length;

      console.log('begin jump recursive...', pageIndex, targetPage);

      // last page
      if (cursorList[cursorListLength - 1] == '0') {
        return false;
      }


      if (pageIndex < targetPage) {
        this.$set(this.pageIndex, menuIndex, ++pageIndex);

        this.refreshKeyList(pageIndex >= cursorListLength).then(() => {
          this.pageNextRecursive(menuIndex, targetPage);
        });
      }
    },
    jumpToPage(menuIndex, targetPage) {
      targetPage = parseInt(targetPage);

      if (isNaN(targetPage) || targetPage <= 0) {
        this.$refs[`pageIndexInput${menuIndex}`][0].value = 1;
        targetPage = 1;
      }

      const nowPage = this.getPageIndex(menuIndex);
      const cursorListLength = this.scanCursorList[menuIndex].length;

      this.setGlobalConnection(menuIndex);

      console.log('prepare to jump to', nowPage, targetPage);

      if (targetPage >= cursorListLength) {
        // to biggest page index
        this.$set(this.pageIndex, menuIndex, cursorListLength - 1);
        const recursiveResult = this.pageNextRecursive(menuIndex, targetPage);

        // last page
        if (recursiveResult === false) {
          this.$refs[`pageIndexInput${menuIndex}`][0].value = this.pageIndex[menuIndex];

          this.$message.error({
            message: this.$t('message.max_page_reached'),
            duration: 2000,
          });
        }
      } else {
        this.$set(this.pageIndex, menuIndex, targetPage);
        this.refreshKeyList(false);
      }
    },
    getPageIndex(menuIndex) {
      if (this.pageIndex[menuIndex] === undefined) {
        this.pageIndex[menuIndex] = 1;
      }

      return this.pageIndex[menuIndex];
    },
    preButtonDisable(menuIndex) {
      const pageIndex = this.getPageIndex(menuIndex);
      return pageIndex <= 1;
    },
    exactMatch() {
      const client = this.$util.get('client');
      const menuIndex = client.options.menu_index;
      const match = this.getMatchMode(menuIndex, false);

      client.existsAsync(match).then((reply) => {
        console.log(match, reply);
        this.$set(this.keyList, menuIndex, (reply === 1) ? [match] : []);
      });

      this.$set(this.nextPageDisabled, menuIndex, true);
      this.$set(this.preButtonDisable, menuIndex, true);
    },
    changeMatchMode(menuIndex) {
      this.resetDb(menuIndex);
      this.setGlobalConnection(menuIndex);

      // extract search
      if (this.searchExact[menuIndex] === true) {
        this.exactMatch();
      }

      else {
        this.refreshKeyList();
      }
    },
    getScanCursor(menuIndex) {
      if (this.scanCursorList[menuIndex] === undefined) {
        this.scanCursorList[menuIndex] = [0];
      }

      const pageIndex = this.getPageIndex(menuIndex);
      const cursorList = this.scanCursorList[menuIndex];

      return cursorList[pageIndex - 1];
    },
    getMatchMode(menuIndex, fillAsterisk = true) {
      if (this.searchMatch[menuIndex] === undefined) {
        this.searchMatch[menuIndex] = '';
      }

      let match = this.searchMatch[menuIndex];
      match = match || '*';

      if (fillAsterisk && !match.match(/\*/)) {
        match = (`*${match}*`);
      }

      return match;
    },
    refreshKeyList(pushToCursorList = true) {
      const client = this.$util.get('client');
      const menuIndex = client.options.menu_index;

      const cursor = this.getScanCursor(menuIndex);
      const match = this.getMatchMode(menuIndex);
      // let promise = client.scanAsync(cursor, 'MATCH', match, 'COUNT', this.keysPageSize).then();
      const pageSize = (match === '*') ? this.keysPageSize : this.searchPageSize;

      // search loading
      this.searchIcon = 'el-icon-loading';

      const promise = this.beginScanning(cursor, match, pageSize, (reply, tmpShow = false) => {
        // refresh key list
        this.$set(this.keyList, menuIndex, reply[1]);

        if (tmpShow) {
          return true;
        }

        pushToCursorList && this.scanCursorList[menuIndex].push(reply[0]);

        if (reply[0] === '0') {
          this.$set(this.nextPageDisabled, menuIndex, true);
        } else {
          this.$set(this.nextPageDisabled, menuIndex, false);
        }


        // search input recover
        this.searchIcon = 'el-icon-search';

        console.log('new cursor list', this.scanCursorList);
      });

      return promise;
    },
    beginScanning(cursor, match, count, callback, recursive = true, minLength = null, lastList = []) {
      const client = this.$util.get('client');
      !minLength && (minLength = this.keysPageSize);

      const promise = client.scanAsync(cursor, 'MATCH', match, 'COUNT', count).then((reply) => {
        console.log(`cursor:${cursor} count: ${count} match:${match}`, 'scan result', reply);

        reply[1] = reply[1].concat(lastList);

        // key list length smaller than minLength
        if (recursive && (reply[0] !== '0') && (reply[1].length < minLength)) {
          callback && callback(reply, true);
          return this.beginScanning(reply[0], match, count, callback, recursive, minLength, reply[1]);
        }

        callback && callback(reply);
      });

      return promise;
    },
    showNewKeyDialog(menuIndex) {
      this.newKeyDialog = true;
      this.setGlobalConnection(menuIndex);
    },
    addNewKey() {
      this.$bus.$emit('addNewKey', this.selectedNewKeyType);
      this.newKeyDialog = false;
    },
  },
  mounted() {
    this.initConnections();
  },
};
</script>

<style type="text/css">
  body {
    /*margin-left: 0;*/
  }
  .connection-menu {
    margin-top: 10px;
  }
  .connection-menu .connection-name {
    display: inline-block;
    width: 115px;
    word-break:keep-all;
    white-space:nowrap;
    overflow:hidden;
    text-overflow:ellipsis;
  }
  .connection-menu .db-select {
    width: 100%;
  }
  .connection-menu .new-key-btn {
    width: 100%;
  }
  .connection-menu .search-item {
    margin-top: -10px;
    margin-bottom: 15px;
  }
  .connection-menu .search-input .el-input__inner {
    padding-right: 43px;
    /*margin-top: -10px;;
    margin-bottom: 15px;*/
  }
  .connection-menu .el-submenu__title {
    padding-left: 0px !important;
    padding-right: 0px;
    font-size: 13px !important;
  }
  .connection-menu .el-submenu__title .el-submenu__icon-arrow {
    right: 7px;
    top: 54%;
  }
  .connection-menu .el-submenu [class^=el-icon-] {
    font-size: 13px;
    margin: 0px;
    width: auto;
    color: grey;
  }

  .connection-menu .el-submenu.is-opened {
    /*background: #ECF5FF;*/
  }

  .connection-menu .search-item .search-icon {
    font-size: 128%;
    color: #a5a8ad;
    cursor: pointer;
  }
  .connection-menu .search-item .el-checkbox__input {
    /*line-height: 28px;*/
    display: inline;
  }

  .connection-menu .key-list {
    list-style-type: none;
    padding-left: 10px;
  }
  .connection-menu .key-list .key-item {
    white-space:nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    cursor: pointer;
    color: #3c5765;
    font-size: 82%;
    line-height: 1.6;
    margin-right: 3px;
  }
  .connection-menu .key-list .key-item:hover {
    color: #409EFF;
  }

  .pagenation {
    margin-top: 10px;
  }
  .pagenation .page-jumper {
    width: 40px;
    height: 24px;
    padding: 5px;
    font-size: 12px;
    margin: 0 10px;
    text-align: center;
  }

  .pagenation input::-webkit-outer-spin-button,
  .pagenation input::-webkit-inner-spin-button {
    -webkit-appearance: none !important;
    margin: 0;
  }
</style>
