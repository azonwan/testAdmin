<template>
  <div class="font16 hgt_full" v-cloak>
    <div class="flex_column hgt_full">
      <!-- 查询表单 -->
      <div class="p-t-20">
        <el-form :inline="true">
          <el-form-item>
            <el-input
              placeholder="请输入搜索内容"
              v-model="searchVal"
              @keyup.enter.native="getAllManagerOfPlatform"
              class="input-with-select"
            >
              <el-select
                v-model="searchConditionVal"
                slot="prepend"
                placeholder="请选择查询条件"
                class="wid90"
              >
                <el-option
                  :label="item.Label"
                  :key="index"
                  :value="item.value"
                  v-for="(item,index) in searchConditionOptions"
                ></el-option>
              </el-select>
            </el-input>
          </el-form-item>
          <el-form-item>
            <el-button type="primary" @click="getAllManagerOfPlatform">查询</el-button>
          </el-form-item>
        </el-form>
      </div>
      <!-- 用户列表 -->

      <el-table
        height="100%"
        :data="teacherList"
        tooltip-effect="light"
        border
        style="width: 100%"
        ref="refElTabel"
      >
        <el-table-column width="80" label="头像">
          <template slot-scope="scope">
            <img :src="scope.row.Face" class="wid28" />
          </template>
        </el-table-column>
        <el-table-column prop="Id" label="ID" width="50"></el-table-column>
        <el-table-column label="姓名" width="120">
          <template slot-scope="scope">
            <span
              class="color-1f85aa font-w6 cursor"
              @click="openMoreOperationDialog(scope.$index, scope.row)"
            >{{scope.row.Realname}}</span>
          </template>
        </el-table-column>
        <el-table-column prop="Role" label="身份" width="80">
          <template slot-scope="scope">
            <span>{{common.FormatSelect(common.managerRoleList,scope.row.Role)}}</span>
          </template>
        </el-table-column>
        <el-table-column prop="Sex" width="50" label="性别"></el-table-column>
        <el-table-column prop="Tel" label="电话号码" width="100"></el-table-column>
        <el-table-column prop="Info" label="个人描述" :show-overflow-tooltip="true"></el-table-column>
        <el-table-column label="操作" width="280">
          <template slot-scope="scope">
            <el-button type="danger" @click="resetPassword(scope.$index, scope.row)">重置密码</el-button>
            <el-button
              v-show="scope.row.Status==1"
              type="info"
              @click="setTeacherStatus(scope.$index, scope.row,0)"
            >禁用</el-button>
            <el-button
              v-show="scope.row.Status==0"
              type="success"
              @click="setTeacherStatus(scope.$index, scope.row,1)"
            >启用</el-button>
            <el-button type="warning" @click="sendMessage(scope.$index, scope.row)">给他留言</el-button>
          </template>
        </el-table-column>
      </el-table>
      <!-- 用户操作 -->
      <div class="between-center m-v-15">
        <el-button type="primary" @click="openNewItem()">新增用户</el-button>
        <div> 
          
          <el-pagination
            background
            @current-change=" getDataChangePage"
            :current-page.sync="nowPage"
            :page-size="rows"
            layout="total,prev, pager, next, jumper"
            :total="allRows"
          ></el-pagination>
        </div>
      </div>
    </div>

    <!-- 老师信息的弹出框 -->

    <!-- 更多操作弹窗 -->
    <my-dialog
      :visible.sync="moreOperationDialog"
      :close-show="true"
      :title="currentRowData.Realname"
    >
      <!-- 展示校区的基本信息 -->
      <div slot="left_content">
        <teacher-row-detail :formItemData="currentRowData" />
      </div>
      <div slot="right_content" class="p_both20 p-b-20">
        <el-tabs   @tab-click="onChangeTabs">
          <el-tab-pane label="管辖校区" name="gxxq" id="gxxq">
            <myPlatform :formItemData="currentRowData" :currentPlatform="currentPlatform"></myPlatform>
          </el-tab-pane>
          <el-tab-pane label="权限设置" name="qxsz" id="qxsz">
            <set-right :formItemData="currentRowData" :currentPlatform="currentPlatform"></set-right>
          </el-tab-pane>
        </el-tabs>
      </div>
    </my-dialog>
    <addAlarmDialog ref="refAlarmForm" />
    <!-- 新增校区信息弹出框 -->
    <el-dialog
      :visible.sync="editDialog"
      width="500px"
      :title="currentRowData.Id>0?'编辑'+currentRowData.Label:'新增'"
    >
      <teacher-row-detail
        :editEnable="true" :platform="currentPlatform"
        :formItemData="currentRowData"
        @subClickEvent="updateTeacherList"
      />
    </el-dialog>
  </div>
</template>

<script>
import common from "@/utils/common";
import {
  getManagerList,
  setStateManager,
  resetPasswordManager,
  getManagerPower
} from "@/api/manager";
import { getAllManagerOfPlatform } from "@/api/platform";
import myDialog from "@/components/myDialog/myDialog";
import teacherRowDetail from "@/views/manager/components/teacherRowDetail";
import setRight from "@/views/manager/components/setRight";
import myPlatform from "@/views/manager/components/myPlatform";
import addAlarmDialog from "@/views/manager/components/addAlarmDialog";
export default {
  name: "managerList",
  components: {
    myDialog,
    teacherRowDetail,
    setRight,
    addAlarmDialog,
    myPlatform
  },
  data() { 
    return {
      common,
      // 搜索用户条件选择的选项
      searchConditionOptions: [
        {
          value: "realname",
          Label: "姓名"
        },
        {
          value: "username",
          Label: "昵称"
        },

        {
          value: "tel",
          Label: "电话"
        }
      ],
      // 搜索的输入值，内容，类型，角色
      searchVal: "",
      searchConditionVal: "",
      searchRoleVal: "",
      // 存放用户列表数据
      teacherList: [],
      // 数据总条数
      allRows: 0,
      // 当前页数
      nowPage: 1,
      // 每页获取数据的总条数
      rows: 30,
      // 模态框获得的单条数据
      currentRowData: {},
      // 当前选中行的数据索引
      currentTeacherIndex: null,
      // 点开弹出默认显示老师信息
      activElTab: "qxsz",
      editDialog: false,
      // 更多操作弹框展示
      moreOperationDialog: false,
      // 当前用户所有的权限数据
      managerRightsMap: {},
      //当前所在校区
      currentPlatform: 0
    };
  },
  methods: {
     onChangeTabs(item) { 
        item.$children[0].fire(); 
    },
    // 获取用户信息的列表
    async getAllManagerOfPlatform() {
      let offsetRow = (this.nowPage - 1) * this.rows;
      let searchCondition = this.searchConditionVal;
      let searchVal = this.searchVal;
      let res;
      if (this.currentPlatform > 0) {
        res = await getAllManagerOfPlatform(this.currentPlatform, {
          simple: false
        });
      } else {
        res = await getManagerList("", {
          limit: this.rows,
          offset: offsetRow,
          role: this.searchRoleVal,
          [searchCondition]: searchVal
        });
      }

      // 获取数据的总条数
      this.allRows = 0;
      this.teacherList = [];
      if (res.data) {
        this.allRows = res.attach;
        this.teacherList = res.data;
      }
    },

    // 分页获取数据
    getDataChangePage(val) {
      this.nowPage = val;
      this.getAllManagerOfPlatform();
    },
    // 禁用或启用账户
    setTeacherStatus(index, row, status) {
      let msg;
      if (status == 0) {
        msg = "确认禁用该账户?";
      } else if (status == 1) {
        msg = "确认启用该账户?";
      }
      this.$confirm(msg, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning"
      })
        .then(async () => {
          let res = await setStateManager(
            row.Id,
            { status: status },
            false,
            true
          );
          if (res.code == 200) {
            this.$set(this.teacherList, index, res.data);
          }
        })
        .catch(() => {});
    },
    // 打开校区的弹出框
    openNewItem() {
      this.editDialog = true;
      this.currentRowData = {};
    },

    // 发送留言消息
    sendMessage(index, row) {
       let from = this.$store.getters.manager;
      this.$refs.refAlarmForm.setTarget(
        row.Id,
        "给["+row.Realname+"]发送提醒",
        from.Realname + "(" + from.Sex + ")" + from.Tel + " 对你说:"
      );
    },
    // 重置密码
    resetPassword(index, row) {
      this.$confirm("确认重置该账户密码?", "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning"
      })
        .then(async () => {
          let res = await resetPasswordManager(row.Id);
          if (res.code == 200) {
            this.$alert("当前密码是:" + res.title, "密码", {
              confirmButtonText: "确定",
              callback: action => {
                this.$set(this.teacherList, index, res.data);
              }
            });
          }
        })
        .catch(() => {});
    },
    // 打开更多操作的弹出框
    openMoreOperationDialog(index, row) {
      this.currentTeacherIndex = index;
      this.currentRowData = row;

      if (this.currentRowData.Platform) {
        this.currentRowData.platformSelect = [];
        this.currentRowData.platformSelect = this.currentRowData.Platform.split(
          ","
        ).map(Number);
      }

      this.moreOperationDialog = true;
    },

    // 修改或编辑老师个人信息后更新老师数据列表
    updateTeacherList(type, rowData) {
      // type=0添加，type=1修改，
      if (type == 0) {
        this.teacherList.unshift(rowData);
      } else if (type == 1) {
        this.currentRowData = { ...rowData };
        this.$set(this.teacherList, this.currentTeacherIndex, rowData);
      }
      this.editDialog = false;
    }
  },
  mounted() {
    let paths = this.$router.currentRoute.path.split("/");
    this.currentPlatform = parseInt(paths[paths.length - 1]);
    if (isNaN(this.currentPlatform)) {
      this.currentPlatform = 0;
    }
    this.getAllManagerOfPlatform();
  },
  created() {
    this.searchConditionVal = this.searchConditionOptions[0].value;
    // this.searchRoleVal = this.common.managerRoleList[0].value;
  }
};
</script>
<style  scope>
</style>