<template>
  <div class="container_edit banner">
    <div class="header">
      <p>新增/编辑轮播图</p>
      <el-button type="primary" @click="handleGoBack">返回</el-button>
    </div>
    <div class="main">
      <el-form :model="user" :rules="rules" ref="user" label-width="150px">
        <el-form-item label="轮播图片" prop="cover">
          <el-upload
            class="avatar-uploader"
            :action="upLoadUrl"
            :header="upLoadHeaders"
            :show-file-list="false"
            :before-upload="beforeAvatarUpload"
          >
            <img v-if="imageUrl" :src="imageUrl" class="avatar">
            <i v-else class="el-icon-plus avatar-uploader-icon"></i>
          </el-upload>
        </el-form-item>
        <el-form-item label="目标名称：" prop="name">
          <el-input v-model="user.name" @change='searchTargetName' debounce="500" clearable placeholder="请输入目标名称"></el-input>
        </el-form-item>
        <el-form-item>
          <el-button type="primary" @click="handleSave('user')">保存</el-button>
        </el-form-item>
      </el-form>
    </div>

    <div class="list table">
      <div class="list-item">
        <div class="detail-table">
          <el-table :data="tableData">
            <el-table-column :formatter="formatter" prop="name" label="关联目标" />
            <el-table-column label="操作">
              <template slot-scope="scope">
                <el-button :type="user.outId == scope.row.id ? 'primary' : null" size="mini" @click="selectBtn(scope.row)">
                  {{user.outId == scope.row.id ? '已选':'选择'}}
                </el-button>
              </template>
            </el-table-column>
          </el-table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { swiper, publicSysApi, meeting } from '@/agent'

export default {
  name: 'meeting-swiper-edit',

  components: {
    "bg-editor": () => import("../../components/bg-editor.vue")
  },

  data() {
    return {
      id: null,
      currentPage:'',
      upLoadHeaders: {},
      upLoadUrl: "",
      imageUrl: "",
      tableData: [],
      pageIndex:1,
      pageSize:10,
      total: 0,
      user: {
        cover: '',
        name: null,
        id: null,
        outId: null
      },
      rules: {
        cover: [{ required: true, message: "请上传图片", trigger: "blur" }],
        // name: [{ required: true, message: "请输入目标名称", trigger: "blur" }]
      }
    }
  },

  mounted() {
    this.id = this.$router.currentRoute.query.id
    this.currentPage = this.$router.currentRoute.query.pageIndex || 1

    if(this.id){
      this.user.id = this.$router.currentRoute.query.id
      this.getSwiperDetail();
    }

    const param = {
      pageSize: this.pageSize,
      pageIndex: this.pageIndex
    }
    this.getList(param)
  },
  
  methods: {
    // 没有数据格式化
    formatter(row, column, cellValue) {
      return cellValue ? cellValue : "- -";
    },
    // 返回
    handleGoBack() {
      this.$router.push({ path: "/meeting-swiper-list" });
    },
    // 选中关联
    selectBtn(obj){
      this.user.name = obj.name
      this.user.outId = obj.id
    },
     // 文件上传时的钩子
    async beforeAvatarUpload(file) {
      const param = {
        suffix: file.name.split(".")[1], // 文件后缀  png
        contentType: file.type // 文件类型  image/png
      }
      const { data } = await publicSysApi.getOSSUploadUrl(param)

      if(data.status == 200) {
        this.upLoadUrl = data.data.bussData.uploadUrl
        this.user.cover = data.data.bussData.fileKey
        const downloadUrl = data.data.bussData.downloadUrl

        publicSysApi.uploadFile(this.upLoadUrl, file, file.type, res => {
          this.imageUrl = downloadUrl
        });
      }
    },
    // 获取轮播详情
    async getSwiperDetail() {
      const { data = {} } = await swiper.getSwiperDetail(this.id)
      const { bussData } = data && data.data
      
      if(data.status == 200) {
        this.user = bussData
        this.imageUrl = bussData.coverUrl

        if(this.user.category) {
          const param = {
            type: this.user.category,
            pageIndex: this.pageIndex,
            pageSize: this.pageSize
          }
          this.getList(param)
        }
      }
    },
    // 标记、保存
    handleSave: function(formName) {
      this.$refs[formName].validate(async valid => {
        if (valid) {
          const { category, name, ...extUser } = this.user
          const param = {
            type: 'meeting',
            category: 'meeting',
            ...extUser
          }
          const { msg, data = {} } = await swiper.save(param)

          if (data.status == 200) {
            this.$message({ type: "success", message: msg })
            this.handleGoBack()
          } else {
            this.$message({ type: "warning", message: msg });
          }
        } else {
          return false;
        }
      })
    },
    // 获取关联列表
    async getList(param) {
      const { data = {} } = await meeting.getMeetList(param)
      const { bussData = [] } = data && data.data

      if(data.status == 200 ) {
        this.tableData = data.data.bussData
        this.total = data.data.bussData.length
      }
    },
    
    // 模糊防抖查询目标名称
    searchTargetName(val) {
      const param = {
        name: val,
        pageIndex: this.pageIndex,
        pageSize: this.pageSize
      }

      this.getList(param)
    }
  }
};
</script>
<style lang="scss">
  .banner{
    .list.table{
      padding-bottom: 50px;
      background-color: #fff;
      // width: 90%;
      padding-left: 5%;
      padding-right: 5%;
    }
  }
</style>
